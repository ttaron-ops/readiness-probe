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
sources: 14
---

# 01b.8 · eBPF — the observability substrate

> **Concept.** eBPF is a sandboxed virtual machine inside the kernel: you attach small verified programs to kernel and userspace hooks, they run at those events, and they ship data out through maps — turning the kernel into a programmable, production-safe observability datapath.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 07 ended by naming a mechanism without explaining it: Cilium's socket-LB rewrites a connection's destination "at connect time" using "a BPF program attached to a socket hook," and keeps state in "its own eBPF conntrack map." You were asked to take that on faith. This lesson removes the faith requirement. It builds eBPF from the instruction set upward — the VM, the load path, the verifier's actual proof obligations, the map types and their cost model, the attach-point taxonomy and where each one physically intercepts the kernel — so that "Cilium is eBPF-based" and "Hubble sees Kubernetes identity that iptables logging can't" become mechanisms you can verify with `bpftool` on a node in front of you.

This is deliberately the deepest lesson in the module. Everything you have learned so far — the run queue (01), namespaces (02), cgroup files (03), PSI (04), reclaim and the OOM killer (05), NUMA (06), the network datapath (07) — has been observed through `/proc` snapshots and after-the-fact counters. eBPF is the instrument that lets you watch those subsystems *while they happen*, on a production node, without a redeploy and without a reboot. Next: **09 — perf / ftrace / USE method**, which adds statistical CPU profiling and the kernel's built-in static tracer, and assembles all of it into a systematic diagnostic flow.

## Why this matters

"eBPF" is a line item in the job descriptions you are targeting. CoreWeave wants kernel-level observability on GPU fleets; Datadog's Linux-kernel/GPU-monitoring role names "the intersection of eBPF, the Linux kernel, and GPU infrastructure"; NVIDIA's Base OS team lists eBPF tracing alongside kdump and GDB. What is being filtered for is not `bpftrace` syntax — it is whether you can explain the *machine*: what the verifier proves, why a per-CPU map is cheaper than a ring buffer, what breaks when a kprobe target is renamed in the next kernel.

The day-job payoff is bigger than the interview one. On a GPU node you routinely face questions that no metric endpoint can answer: *which container is issuing the 4 KiB random reads that are pinning this NVMe while the trainer stalls?* *Which process is calling `execve` on something that isn't in the image?* *Is the data loader blocked in the kernel's page-cache read path or in the NFS client?* Each of those is a five-minute answer with eBPF and an open-ended afternoon without it. And when the answer costs $30/hour of idle H100 to find, five minutes versus an afternoon is a real number on a real invoice.

It is also not one vendor's bet. The **eBPF Foundation** sits under the Linux Foundation with Meta, Google, Isovalent/Cisco, Microsoft, Netflix and others as members — direct competitors who independently converged on the same in-kernel substrate. That convergence is the strongest available signal that this is durable infrastructure rather than a cycle to wait out.

## What's new here (calibration)

Per the [module README](../README.md)'s calibration, you already know the *surface*: you have used `cilium hubble observe`, a Datadog network map, maybe `execsnoop` or `opensnoop`, and you know `strace` shows syscalls and is dangerous on a hot process because ptrace stops the target twice per call. None of that surface is re-taught. What is genuinely new:

- **The machine underneath.** The 11-register VM, the ELF-to-`bpf()`-syscall load path, the verifier's register-state lattice and pruning, the JIT — as a mechanism you can debug, not a black box that emits events.
- **The real constraints**, with their constant names and numbers: 512-byte stack, 4096-instruction ceiling for unprivileged programs, one million instructions of verifier budget, bounded loops since 5.3, 33 nested tail calls.
- **The attach-point taxonomy placed on the kernel's own code path** — why a tracepoint costs ~15 ns, an fentry probe ~24 ns, a kprobe ~137 ns, and a uprobe ~1.7 µs, and what that means for what you can leave running.
- **Map choice as the entire performance story**: in-kernel aggregation versus per-event streaming, with the arithmetic that decides between them.
- **BTF and CO-RE** as a concrete relocation mechanism — what the compiler emits, what the loader patches, and what still breaks.
- **The GPU-specific wrinkle**, corrected: uprobes on `libcuda.so`/NCCL resolve through the ELF symbol tables, not debuginfo — so exported symbols *are* attachable in a stripped image, while typed argument access is what you lose.

## Core concepts

### 1. The problem: extending the kernel without becoming the kernel

You want to answer a question the kernel already knows the answer to. Every block I/O completion passes through `blk_mq_end_request` with a `struct request` carrying the device, sector, size, issuing task and queue timestamps — and the kernel throws almost all of it away, keeping a handful of aggregated counters in `/proc/diskstats`. Before eBPF you had four bad options:

| Option | How it works | Why it fails in production |
|---|---|---|
| Patch the kernel | Add your counters, rebuild, reboot | Reboot per question; you now maintain a kernel fork |
| Kernel module | `insmod` code with full kernel privileges | A NULL dereference panics the node; no safety boundary at all |
| `strace` / ptrace | Stop the target on every syscall, tracer reads state | Two context switches *per event*; 10–100× slowdown on syscall-heavy processes |
| `/proc` polling | Sample counters from userspace | You see aggregates after the fact, never the individual event or its stack |

eBPF's answer: let userspace hand the kernel a program, but **prove it safe before running it**, restrict it to a fixed helper API, and compile it to native code so the proof does not cost speed. The lineage is the 1992 BSD Packet Filter (McCanne & Jacobson) — a tiny in-kernel VM that let `tcpdump` push its filter into the kernel instead of copying every packet up. That "classic BPF" had two 32-bit registers and did one job; the extended version merged into Linux 3.15 (2014) widened it to 64-bit registers, added maps and a syscall (`bpf(2)`, Linux 3.18), and hooks far beyond packets.

The payoff of "push the filter down" is unchanged and still the whole point: **filter and aggregate where the data already is, so the expensive boundary crossing happens once per summary instead of once per event.**

### 2. The virtual machine: registers, stack, instructions

An eBPF program is bytecode for a deliberately small RISC-like machine, sized so that the verifier's job stays tractable and so that registers map one-to-one onto x86-64/arm64 registers when JITed.

| Property | Value | Consequence for you |
|---|---|---|
| Registers | 11 (`r0`–`r10`), 64-bit | `r10` is a read-only frame pointer; you cannot manufacture pointers |
| Return value | `r0` | Helper return and program return both land here |
| Helper arguments | `r1`–`r5` (`MAX_BPF_FUNC_REG_ARGS` = 5) | No helper takes more than five arguments |
| Callee-saved | `r6`–`r9` | Preserved across helper calls; scratch lives in `r1`–`r5` |
| Stack | 512 bytes (`MAX_BPF_STACK`) | A single `char buf[512]` exhausts it — this is the #1 beginner wall |
| Instruction width | 8 bytes (16 for 64-bit immediate loads) | Program size is counted in instructions, not bytes |
| Max instructions, unprivileged | 4096 (`BPF_MAXINSNS`) | Historical limit, still applies without `CAP_BPF` |
| Max instructions, privileged | 1,000,000 (since Linux 5.2) | Cilium-scale programs live here |
| Nested tail calls | 33 (`MAX_TAIL_CALL_CNT`) | Program chaining has a hard depth |

The 512-byte stack shapes how real programs are written: a 4 KiB scratch buffer (a path string, say) cannot live on the stack, so you allocate a **per-CPU array map with one element** and use that element as scratch. That idiom appears in nearly every non-trivial libbpf program, purely because of this table's fifth row.

Everything else comes from **helpers** — `bpf_map_lookup_elem()`, `bpf_probe_read_kernel()`, `bpf_ktime_get_ns()`, `bpf_get_current_pid_tgid()`, and roughly two hundred more — plus a curated set of **kfuncs** (kernel functions exported to BPF which, unlike helpers, carry no ABI-stability promise). Arbitrary kernel calls are not available.

### 3. The load path, end to end

Here is the whole pipeline, from a `.c` file on your laptop to native instructions executing inside a kernel function, with the map plane drawn beside it. Study this diagram; the rest of the section walks each stage.

```
  USERSPACE                                  |   KERNEL
                                             |
  prog.bpf.c                                 |
     │  clang -O2 -g -target bpf -c          |
     ▼                                       |
  prog.bpf.o   (ELF)                         |
   ├── .text / SEC("fentry/vfs_read")  ──┐   |
   ├── .maps  (BTF-defined map descrs)  │   |
   ├── .BTF   (types + CO-RE relocs)    │   |
   └── .rodata (const config)           │   |
     │                                   │   |
     │ libbpf: parse ELF, create maps    │   |
     │         apply CO-RE relocations ◀─┼───┼── /sys/kernel/btf/vmlinux
     ▼                                   │   |      (target kernel's real
  bpf(BPF_MAP_CREATE, ...)  ─────────────┼───┼──▶ [ map objects ]  struct layouts)
  bpf(BPF_PROG_LOAD, insns, ...) ────────┼───┼──▶ ┌───────────────┐
                                         │   |    │   VERIFIER    │  reject → EACCES/EINVAL
                                         │   |    │  DAG check    │  + verifier log
                                         │   |    │  path walk    │
                                         │   |    │  bounds/types │
                                         │   |    └──────┬────────┘
                                         │   |           │ accept
                                         │   |           ▼
                                         │   |    ┌───────────────┐
                                         │   |    │      JIT      │ bytecode → x86-64/arm64
                                         │   |    └──────┬────────┘
                                         │   |           │
  bpf(BPF_LINK_CREATE / PERF_EVENT_IOC)  |   |           ▼
        ──────────────────────────────────────▶  attach to hook
                                             |     ┌──────────────────────────┐
                                             |     │ vfs_read() { ... }       │
                                             |     │   ▲ program runs here    │
                                             |     └───┬──────────────────────┘
                                             |         │ bpf_map_update_elem()
                                             |         ▼
                                             |     [ PERCPU_HASH ] ◀── aggregate in kernel
                                             |     [ RINGBUF     ] ──┐ per-event stream
   read()/mmap ◀──────────────────────────────────────────────────────┘
   ring_buffer__poll()  ── epoll wakeup when consumer has caught up
```

**Stage 1 — compile.** `clang -target bpf` emits an ELF object, not an executable. Section names carry semantics: `SEC("fentry/vfs_read")` gives libbpf both the program type and the attach target. Maps are structs in a `.maps` section, encoded in BTF. `-g` is not optional — it is what emits the `.BTF` section CO-RE relocations live in.

**Stage 2 — load.** libbpf (or bpftrace, or the Go/Rust equivalents) parses that ELF and drives the `bpf(2)` syscall:

| `bpf()` command | What it does |
|---|---|
| `BPF_MAP_CREATE` | Allocate a map; returns an fd |
| `BPF_PROG_LOAD` | Submit instructions + license + type; runs the verifier and the JIT; returns an fd |
| `BPF_BTF_LOAD` | Register the program's BTF blob with the kernel |
| `BPF_LINK_CREATE` | Attach a loaded program to a hook, returning a link fd |
| `BPF_MAP_LOOKUP_ELEM` / `_UPDATE_ELEM` / `_GET_NEXT_KEY` | Userspace read/write/iterate of a map |
| `BPF_OBJ_PIN` / `BPF_OBJ_GET` | Pin an fd into bpffs / re-open a pinned object |
| `BPF_ENABLE_STATS` | Turn on per-program run-count and run-time accounting |

Everything is an **fd**. That is the lifetime model: when the last fd to a program or map closes, the object is freed — which is why a bpftrace script's maps vanish when you press Ctrl-C, and why long-lived agents pin their objects (§8).

**Stage 3 — verify.** Covered in detail next. Note where it sits: *before* the program can run, at load time, once. There is no runtime sandbox check per instruction.

**Stage 4 — JIT.** The verified bytecode becomes native machine code; eBPF registers map onto real registers and a JITed program is ordinary kernel text with no interpreter dispatch. `sysctl net.core.bpf_jit_enable` controls it (1 = on, the default; 2 = on plus disassembly to the kernel log). Interpreted execution is roughly an order of magnitude slower — enough to change whether a hot-path probe is affordable.

**Stage 5 — attach.** Loading and attaching are separate operations (§8). A loaded-but-unattached program consumes memory and does nothing.

### 4. The verifier: what it actually proves

"The verifier checks that your program is safe" is the sentence that teaches nothing. Here is what it does.

The verifier runs in two passes over your instruction array.

**Pass 1 — control-flow graph check.** A depth-first search over the instruction graph. It rejects: back-edges that are not provably bounded loops, unreachable instructions, jumps outside the program, and programs longer than the instruction limit. The output of this pass is the guarantee that the program is a DAG (or a DAG plus provably-terminating loops), which is what makes pass 2 finite.

**Pass 2 — abstract interpretation of every reachable path.** The verifier simulates execution, carrying a *state* for each of the 11 registers and each 8-byte slot of the 512-byte stack. Each register holds a type from a lattice:

| Register state | Meaning | What you may do with it |
|---|---|---|
| `NOT_INIT` | Never written | Nothing. Reading it → `R2 !read_ok` |
| `SCALAR_VALUE` | A number, with tracked bounds | Arithmetic; never dereference |
| `PTR_TO_CTX` | Pointer to the program's context struct | Load fields the program type permits |
| `PTR_TO_STACK` | Offset from `r10` | Read only slots you have written |
| `PTR_TO_MAP_VALUE` | Verified-non-NULL map value | Dereference within `value_size` |
| `PTR_TO_MAP_VALUE_OR_NULL` | Fresh from `bpf_map_lookup_elem()` | **Nothing** until you branch on NULL |
| `PTR_TO_PACKET` / `PTR_TO_PACKET_END` | Network data cursor and its end | Dereference only after comparing against `_END` |
| `PTR_TO_BTF_ID` | A typed kernel pointer (fentry args, `tp_btf`) | Field access checked against BTF; faults handled |

Four proof obligations fall out of that lattice, and they are the four things that will actually reject your program:

1. **No uninitialized reads.** Every register and stack slot must be written before it is read. This is why `struct event e;` followed by filling only some fields fails when the whole struct is submitted: the unwritten padding is `NOT_INIT`. The fix — `struct event e = {};` or `__builtin_memset()` — is not superstition, it is discharging a proof obligation.

2. **Every pointer dereference is provably in bounds.** `bpf_map_lookup_elem()` returns `PTR_TO_MAP_VALUE_OR_NULL`. Until you write `if (!p) return 0;`, the verifier's state for that register is "might be NULL," and any load through it is rejected with `R0 invalid mem access 'map_value_or_null'`. After the branch, the verifier *splits the state*: on the taken path the register is NULL and unusable, on the fall-through path it is `PTR_TO_MAP_VALUE` with a known size. Packet access works the same way: `if (data + sizeof(struct ethhdr) > data_end) return XDP_PASS;` is what converts an unusable pointer into a usable one. You are not appeasing a linter; you are supplying the premise the proof needs.

3. **Arithmetic stays in known ranges.** The verifier tracks each scalar as a *tnum* — a (value, mask) pair recording which bits are known and which are unknown — plus signed and unsigned min/max bounds. It propagates those through every ALU op. `arr[idx]` where `idx` came from a packet is rejected until you write `idx &= (ARRAY_SIZE - 1);` or `if (idx >= ARRAY_SIZE) return 0;`, because only then can the verifier bound the resulting offset. This is also where the notorious "unbounded register" messages come from: after a `bpf_probe_read` into a variable-length loop counter, the bounds widen to "anything," and every subsequent use fails.

4. **Termination.** Originally: no loops at all, at all, ever. Since **Linux 5.3** the verifier accepts *bounded* loops — it walks the loop body repeatedly and, if the induction variable's tracked bounds make the exit condition provably reachable, accepts it. Since **5.17** `bpf_loop(nr_loops, callback, ctx, 0)` gives you a helper-driven loop whose iteration count is checked at runtime instead of unrolled at verification time, which is dramatically cheaper on the verifier's budget (the helper caps at `BPF_MAX_LOOPS`, 8,388,608).

**The budget, and why big programs get rejected for being big.** Because pass 2 explores paths, its cost is combinatorial. The kernel caps it:

| Limit | Constant | Value |
|---|---|---|
| Instructions the verifier will process in total | `BPF_COMPLEXITY_LIMIT_INSNS` | 1,000,000 |
| Jump-sequence depth before giving up | `BPF_COMPLEXITY_LIMIT_JMP_SEQ` | 8,192 |
| Stored states per instruction before pruning gets aggressive | `BPF_COMPLEXITY_LIMIT_STATES` | 64 |

Hit the first one and you get `BPF program is too large. Processed 1000001 insn`. Note the wording: *processed*, not *contains*. A 300-instruction program with a nest of branches can exhaust the budget while a 5,000-instruction straight-line program sails through.

**State pruning is what makes it tractable at all.** Every time verification reaches an instruction, the verifier compares the current register/stack state against states it has already verified at that instruction. If the current state is a *subset* of a previously-verified one — everything known now was known then, and everything safe then is safe now — the branch is pruned and not re-explored. Liveness tracking sharpens this: registers whose values never affect a later memory access are ignored in the comparison. Without pruning, any program with a dozen independent branches would blow the budget.

```
  Verifier path walk, with pruning
  ────────────────────────────────
                    insn 0   {r1=PTR_TO_CTX, r0..r9=NOT_INIT}
                      │
                 insn 4  if r2 > 0 ────────────┐
                      │ (false)                │ (true)
                      ▼                        ▼
     insn 5 {r2=SCALAR umax=0}      insn 9 {r2=SCALAR umin=1}
                      │                        │
                      └──────────┬─────────────┘
                                 ▼
                    insn 14  {r6=PTR_TO_MAP_VALUE_OR_NULL}
                      ┌──────────┴──────────┐
              (r6==0) │                     │ (r6!=0)
                      ▼                     ▼
             insn 15 return 0     insn 16 {r6=PTR_TO_MAP_VALUE, sz=8}
                                            │
                                            ▼
                                   insn 20 *(u64*)r6 = r7      ← LEGAL:
                                                                 non-NULL proven,
                                                                 offset 0 < size 8

  Second arrival at insn 14 with state ⊆ a state already verified there
      → PRUNED, not re-explored.  This is the difference between
        "verifies in 12 ms" and "1,000,001 insns processed".
```

**Reading a rejection.** The verifier prints a log (libbpf shows it on failure; `bpftool prog load ... --log_level 2` gives more). The messages are terse but mechanical:

| Message | What the verifier means | Fix |
|---|---|---|
| `R2 !read_ok` | You read a register never written on some path | Initialize on every path |
| `invalid mem access 'map_value_or_null'` | Dereferenced a lookup result without a NULL check | `if (!v) return 0;` |
| `invalid stack off=-520 size=8` | Stack access outside the 512-byte frame | Move the buffer into a per-CPU array map |
| `math between pkt pointer and register with unbounded ...` | Pointer arithmetic with an untracked scalar | Mask or range-check the offset first |
| `BPF program is too large. Processed 1000001 insn` | Path explosion, not source size | Reduce branching; use `bpf_loop()`; split with tail calls |
| `back-edge from insn 42 to 12` | An unbounded loop | Give the loop a constant bound, or use `bpf_loop()` |
| `Unreleased reference id=2 alloc_insn=17` | Acquired a refcounted object and did not release it | Call the matching release kfunc on every path |

**The consequence to internalize:** safety here is *by construction at load time*, not by trapping faults at runtime. There is no in-kernel exception handler rescuing a bad BPF program — except for one deliberate case, `bpf_probe_read_kernel()` and friends, which perform a fault-tolerant copy and **return `-EFAULT` instead of oopsing** when the address is not mapped. That single exception exists precisely because tracing programs must be able to follow pointers the verifier cannot vouch for. It also means every `bpf_probe_read*` return value is a value you are supposed to check.

### 5. Attach points: where your program physically intercepts the kernel

A hook is *where* your program runs, and the choice determines three things at once: what data you can see, how stable the probe is across kernel upgrades, and what each event costs. Here is the taxonomy laid on the actual code path a read from a training job's dataset takes.

```
        TRAINING PROCESS (python / trainer binary)
        ─────────────────────────────────────────
        libcuda.so: cuMemcpyHtoD()        ◀── uprobe / uretprobe        ~1,670 ns
        libc:       read()                ◀── uprobe, USDT              ~1,670 ns
              │
        ══════╪═══════════ syscall boundary ═════════════════════════════════
              ▼
        sys_enter_read                    ◀── tracepoint (raw_syscalls,     ~15 ns
              │                                syscalls:sys_enter_read)
              ▼
        ksys_read → vfs_read              ◀── kprobe / kretprobe          ~137 ns
              │                           ◀── fentry / fexit               ~24 ns
              ▼
        filemap_read  ──── page cache hit ──▶ copy_to_user → return
              │ (miss)
              ▼
        readahead → submit_bio            ◀── tracepoint block:block_rq_issue
              │
              ▼
        blk_mq_start_request              ◀── fentry (struct request * typed)
              │
              ▼
        [ NVMe device ]
              │  irq
              ▼
        blk_mq_end_request                ◀── tracepoint block:block_rq_complete
              │                                (this is where biolatency stops its clock)
              ▼
        wake_up_process → sched_switch    ◀── tracepoint sched:sched_switch
                                               (this is where offcputime works)

        NETWORK SIDE (same node, NCCL / gRPC traffic)
        ─────────────────────────────────────────────
        NIC ─▶ driver rx ─▶ XDP            ◀── XDP        (before skb allocation)
                     ─▶ skb ─▶ tc/clsact   ◀── tc ingress/egress
                          ─▶ netfilter (conntrack — lesson 07)
                               ─▶ socket   ◀── cgroup/connect4, sock_ops   (Cilium socket-LB)
                                    ─▶ tcp_sendmsg  ◀── kprobe/fentry  (Datadog USM)

        SECURITY / POLICY
        ─────────────────
        security_file_open, bprm_check_security ◀── BPF LSM (sleepable)
```

The families, with the properties that decide between them:

| Family | Attaches to | ABI stability | Typed args | Cost/event | Use when |
|---|---|---|---|---|---|
| **tracepoint** | Static kernel instrumentation (`sched:`, `block:`, `syscalls:`) | Stable — kernel-maintained | Yes, via the tracepoint's format | ~15 ns | Always, if one exists for your question |
| **raw tracepoint / `tp_btf`** | Same sites, raw args, no format-string marshalling | Stable site, raw arg layout | `tp_btf` gives BTF-typed args | Below tracepoint | High-frequency sites where marshalling matters |
| **fentry / fexit** | Any traceable kernel function, via a BPF trampoline (Linux 5.5+) | Unstable — function names/signatures can change | **Yes** — `PTR_TO_BTF_ID`, direct field access, no `bpf_probe_read` | ~24 ns | The modern default for kernel-function tracing on 5.5+ |
| **kprobe / kretprobe** | Any kernel instruction address | Unstable | No — args by register convention, reads via `bpf_probe_read_kernel` | ~137 ns | Older kernels, or mid-function addresses fentry can't reach |
| **uprobe / uretprobe** | Userspace function in a binary/library | Depends on the binary | Only with DWARF | ~1,670 ns | Application internals: TLS reads, CUDA/NCCL entry points |
| **USDT** | Static probes compiled into apps (JVM, PostgreSQL, Node) | Stable per-app contract | Yes | uprobe-class | The app already ships probes |
| **perf_event / PMU** | Timer or hardware counter overflow | Stable | Sample context | Sampling-rate driven | Profiling: `profile:hz:99`, cache-miss sampling |
| **XDP** | Driver receive path, before `skb` allocation | Stable program contract | `xdp_md` | Lowest in the network stack | Line-rate drop/redirect (DDoS, load balancing) |
| **tc / clsact** | `sk_buff` ingress/egress in the traffic-control layer | Stable | `__sk_buff` | Higher than XDP, full skb metadata | Policy and observability with full packet context |
| **cgroup hooks** | `connect(2)`, `bind`, `sendmsg`, socket ops, per-cgroup | Stable | Socket context | Once per connection, not per packet | Cilium socket-LB (lesson 07) — rewrite the destination once |
| **LSM** | Kernel security hooks, sleepable | Stable hook set | BTF-typed | Per security decision | Enforcement, not just observation |
| **iter** | Kernel object iteration (task, socket, map) | Stable | BTF-typed | Per read of the iterator file | Dumping kernel state as a file |

*(Per-event overheads: cloudflare/ebpf_exporter benchmark, `getpid()` on Linux 6.5-rc1, Apple M1 under QEMU. Empty probe: tracepoint 15 ns, fentry 24 ns, kprobe 137 ns. With a simple map increment: 35 / 42 / 160 ns. With a more complex map key: 96 / 103 / 229 ns. Uprobe measured separately at ~1,670 ns/call on 6.7-rc3. Absolute values are hardware-specific; the **ordering and the order-of-magnitude gaps are the durable facts** — a uprobe costs roughly 10× a kprobe and 100× a tracepoint.)*

Three mechanisms behind those numbers, because the "why" is the part that transfers:

- **A tracepoint is a patched-in static branch.** The kernel source contains a `trace_sched_switch(...)` call site compiled as a no-op jump ("jump label"). Enabling it rewrites that jump to call the registered probes. There is no trap, no breakpoint, no exception — that is why it is 15 ns.
- **A kprobe is a breakpoint.** Classically an `int3` overwrites the target instruction; the trap handler runs your probe, emulates or single-steps the displaced instruction, and returns. That trap-and-return is the ~137 ns. The kernel optimizes where it can (a jump to a trampoline, or routing entry kprobes through ftrace's call site), but it is still doing more than a static branch. A **kretprobe** also hijacks the return address using a pre-allocated instance pool (`maxactive`); exhaust the pool under concurrency and returns are silently missed, undercounting your latency histogram.
- **An fentry probe uses a BPF trampoline.** Every kernel function compiled with `-mfentry` starts with a call into ftrace's 5-byte patch site; Linux 5.5 taught BPF to install a generated trampoline there that marshals arguments into BPF's calling convention and jumps straight to your JITed program. No trap — and because the trampoline knows the function's BTF signature, arguments arrive **typed**: `req->__data_len` directly, instead of `bpf_probe_read_kernel(&len, sizeof(len), &req->__data_len)`. `fexit` sees arguments *and* return value, with no instance pool to exhaust.

**The stability rule that matters operationally:** prefer a tracepoint; if none exists, use fentry on 5.5+; treat kprobes as version-pinned instrumentation. `bpftrace -l 'tracepoint:*'` lists every tracepoint on your kernel (`-l 'tracepoint:block:*'` for a subsystem); `/sys/kernel/tracing/available_filter_functions` lists every function a kprobe or fentry probe can attach to. Check the list *on the target kernel* before shipping anything.

### 6. Maps: the data plane, and the choice that decides your overhead

A map is a kernel-resident data structure addressed by an fd, readable and writable from both the BPF program and userspace. It is the only channel out. Choosing the right one is not a detail — it *is* the performance design.

| Map type | Structure | Concurrency | Typical use |
|---|---|---|---|
| `BPF_MAP_TYPE_HASH` | Hash table, key→value | Shared, internally locked | Per-PID/per-cgroup state where per-CPU would fragment |
| `BPF_MAP_TYPE_PERCPU_HASH` | One hash table per CPU | **Lock-free on the hot path** | Counting at millions of events/sec; userspace sums the copies |
| `BPF_MAP_TYPE_ARRAY` | Fixed-size, index-keyed | Shared | Config/flags read by the program |
| `BPF_MAP_TYPE_PERCPU_ARRAY` | Per-CPU fixed array | Lock-free | Histogram buckets; **scratch memory bigger than the 512-byte stack** |
| `BPF_MAP_TYPE_LRU_HASH` | Hash with LRU eviction | Shared or per-CPU variant | Unbounded key spaces (per-connection state) with a hard memory cap |
| `BPF_MAP_TYPE_RINGBUF` | MPSC ring, shared across CPUs (5.8+) | Reservation under spinlock, commit lock-free | Per-event streaming with correct ordering |
| `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | Per-CPU perf ring buffers | Per-CPU | Legacy per-event channel; pre-5.8 |
| `BPF_MAP_TYPE_STACK_TRACE` | Stack id → captured stack | Shared | Profilers: fold thousands of stacks in-kernel |
| `BPF_MAP_TYPE_PROG_ARRAY` | Index → program fd | — | Tail calls (33 deep) for program chaining |
| `BPF_MAP_TYPE_LPM_TRIE` | Longest-prefix-match trie | Shared | CIDR lookups in network policy |
| `BPF_MAP_TYPE_TASK_STORAGE` / `CGROUP_STORAGE` | Value attached to a task / cgroup | Per-object | Per-task or per-cgroup state without a global hash |

**The ring buffer, mechanically.** Before 5.8, per-event output meant `BPF_MAP_TYPE_PERF_EVENT_ARRAY`: one ring per CPU, so memory was `N_CPUs × ring_size` (on a 128-core node at 64 pages per CPU, 32 MiB of pinned memory for one tool) and cross-CPU events had no global ordering. `BPF_MAP_TYPE_RINGBUF` is one multi-producer/single-consumer ring shared by all CPUs, power-of-two sized. Producers call `bpf_ringbuf_reserve()`, write straight into the reserved space, then `bpf_ringbuf_submit()`: reservation increments the producer counter under a spinlock (strict ordering) while commits are lock-free, so a slow producer delays visibility of later records but can never corrupt the ring, and the verifier enforces submit-or-discard on every path. The data area is mapped twice contiguously, so wrapping needs no special case. Notification is **self-pacing** — the kernel wakes the epoll consumer only when it has caught up, so a busy producer pays no wakeup per event.

**The arithmetic that decides between aggregation and streaming.** Take a node issuing 1,000,000 block I/Os per second — a realistic figure for a 128-core box with several NVMe drives under a data-loading workload.

- *Streaming design*: emit a 32-byte event per I/O. `1e6 events/s × 32 B = 32 MB/s` copied through the ring and consumed by userspace, plus a `bpf_ringbuf_reserve`/`submit` pair per event (tens of ns), plus userspace parsing 1e6 records/s. That is a real CPU core's worth of work, permanently.
- *Aggregation design*: a per-CPU array of 27 histogram buckets, one atomic-free increment per event. Per event you pay one bucket-index computation plus one increment — a handful of nanoseconds. Userspace reads `27 buckets × N_CPUs` values once per second. At 128 CPUs that is 3,456 values/second, roughly **five orders of magnitude less data movement** than the streaming design, for the same latency-distribution answer.

**The rule: aggregate in a per-CPU map and read summaries; stream through a ring buffer only when the individual event is the answer.** `execve` tracing is a legitimate ring-buffer case — there are perhaps tens of execs per second and you need the filename. Block-I/O latency is not; you want the histogram, not four million records.

Why *per-CPU* specifically: a shared `HASH` updated from 128 CPUs puts every CPU on the same cache lines, and the bouncing costs more than the probe. A per-CPU map gives each CPU its own copy — zero hot-path contention — and pushes the cheap summation onto userspace. The price is memory, `value_size × max_entries × N_CPUs` (a 1 MiB per-CPU value on 128 cores is 128 MiB), and since 5.11 that memory is charged to the loader's memory cgroup — a surprising OOM source for agent pods.

### 7. BTF and CO-RE: one binary, every kernel

**The problem.** Your program wants `task->pid`. The compiler turns that into "load 8 bytes at offset 2464 from this pointer." That offset is a property of the exact kernel that struct was compiled for — it moves between versions, and between two builds of the *same* version with different `CONFIG_` options. So a program compiled against your laptop's headers reads garbage on a node running a different kernel. The old answers were both bad: BCC shipped LLVM and kernel headers to every host and compiled at runtime (slow, fragile, hundreds of MB per node), or you built one binary per kernel version and maintained a matrix.

**BTF (BPF Type Format)** is a compact type-description format: struct and union layouts, member names and offsets, enums, typedefs, function prototypes. The kernel is built with `CONFIG_DEBUG_INFO_BTF=y` (default on every mainstream distro kernel since roughly 2020) and exposes its own BTF at a fixed path:

```
$ ls -l /sys/kernel/btf/vmlinux
-r--r--r-- 1 root root 5495747 Aug 17 09:14 /sys/kernel/btf/vmlinux

$ bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
$ wc -l vmlinux.h
185432 vmlinux.h
```

That second command is how every libbpf project gets its headers: it reconstructs C declarations for *the running kernel's* types, so `#include "vmlinux.h"` replaces the kernel headers package entirely. Per-module BTF lives under `/sys/kernel/btf/<module>`.

**CO-RE (Compile Once – Run Everywhere)** is the relocation machinery on top. When you write `BPF_CORE_READ(task, mm, rss_stat.count[0])` or use `__builtin_preserve_access_index`, clang does *not* bake in the offset. It emits the access plus a **CO-RE relocation record** in the `.BTF.ext` section saying "this instruction's immediate is the offset of member `pid` of `struct task_struct`, as known at compile time." At load time libbpf reads the target kernel's BTF, finds `struct task_struct`, looks up `pid`, and **patches the instruction's immediate with the real offset for this kernel** before calling `BPF_PROG_LOAD`.

The relocation kinds cover more than offsets:

| Relocation | Question it answers at load time | Typical use |
|---|---|---|
| field offset | Where does this member live now? | The common case |
| field exists | Does this member exist on this kernel? | `bpf_core_field_exists()` — feature-gate a code path |
| field size | How wide is it now? | Fields that changed from `u32` to `u64` |
| type exists / type size | Does this struct exist here? | Handle structs that were split or renamed |
| enum value exists / value | What is this enumerator's numeric value here? | Enums whose values were reordered |

```
   COMPILE ONCE                          RUN EVERYWHERE
   ────────────                          ──────────────
   prog.bpf.c                            node A: kernel 5.15   node B: kernel 6.8
   task->pid                             task_struct.pid @2464  task_struct.pid @2512
        │                                        ▲                      ▲
        ▼ clang -g -target bpf                   │                      │
   ldxdw r1, [r2 + 0]  ◀── placeholder           │  libbpf reads        │
   + reloc: {struct task_struct, "pid"}          │  /sys/kernel/btf/vmlinux
        │                                        │  and patches the immediate
        └────────────── one .o file ─────────────┴──────────────────────┘
```

**What CO-RE does not fix.** If a field is *renamed* (not moved), the relocation fails and the program refuses to load — correctly, because silently reading the wrong bytes would be worse. That is what `bpf_core_field_exists()` is for: check, and branch to a fallback. If the target kernel has no BTF (`CONFIG_DEBUG_INFO_BTF=n`, some minimal or embedded kernels), CO-RE has nothing to relocate against; the community answer is **BTFHub**, a repository of pre-generated BTF blobs for older distro kernels that libbpf can be pointed at.

Verify BTF exists before you plan around it (`ls /sys/kernel/btf/vmlinux`) — on a vendor-built GPU-node kernel this decides whether your agent ships as one binary or as a build farm.

### 8. Load vs attach, lifetime, and pinning

Three things get conflated constantly:

1. **Load** — `bpf(BPF_PROG_LOAD)`: verify + JIT. Returns an fd. Nothing runs.
2. **Attach** — `bpf(BPF_LINK_CREATE)` or an older per-type ioctl: bind the loaded program to a hook. Returns a **link** fd. Now it runs.
3. **Pin** — `bpf(BPF_OBJ_PIN)`: expose an fd as a path under **bpffs**, conventionally mounted at `/sys/fs/bpf`.

Lifetime is refcounted on fds: close the last fd to a program, map or link and the kernel frees it — which is why a bpftrace script's maps vanish at Ctrl-C and why "it was working until I closed my SSH session" is almost always this. Pinning adds a reference held by the filesystem, so the object survives its loader exiting; that is how a CNI keeps its datapath alive across agent restarts. The first move on an unfamiliar node is to ask what is already loaded:

```
$ sudo bpftool prog show
23: cgroup_skb  name egress  tag 6deef7357e7b4530  gpl
        loaded_at 2026-08-11T04:12:17+0000  uid 0
        xlated 64B  jited 54B  memlock 4096B  map_ids 12
147: sched_cls  name cil_from_container  tag 9b0f2b1cf1b28d0e  gpl
        loaded_at 2026-08-11T04:12:31+0000  uid 0
        xlated 4192B  jited 2673B  memlock 8192B  map_ids 41,42,58
        btf_id 89
312: tracing  name blk_account  tag 3c31e0a2c5b8d901  gpl
        loaded_at 2026-08-17T06:02:44+0000  uid 0
        xlated 512B  jited 331B  memlock 4096B  map_ids 77

$ sudo bpftool map show id 77
77: percpu_hash  name lat_hist  flags 0x0
        key 8B  value 8B  max_entries 4096  memlock 1150976B

$ sudo bpftool net show                 # what is attached to which interface
tc:
eth0(2) clsact/ingress cil_from_netdev id 148
eth0(2) clsact/egress cil_to_netdev id 151

$ ls /sys/fs/bpf/tc/globals/            # Cilium's pinned maps
cilium_call_policy  cilium_ct4_global  cilium_lb4_services_v2  cilium_metrics
```

*(Representative transcript from a Cilium-managed node.)* Program 147 is Cilium's per-container tc classifier on `eth0`'s clsact qdisc; its state lives in pinned maps under `/sys/fs/bpf/tc/globals/`, which is why `cilium-agent` can restart without dropping traffic. Program 312 is a tracing program somebody loaded this morning — worth asking about. Follow-ups: `bpftool prog dump xlated id 312` (post-verifier bytecode), `... dump jited` (native code), `bpftool map dump id 77` (live contents), `bpftool prog tracelog` (whatever `bpf_printk()` writes — really `/sys/kernel/tracing/trace_pipe`, lesson 09).

### 9. Cost, privilege, and the safety envelope

**Measure, do not assume.** The kernel can account per-program execution time, and it is off by default because the accounting itself costs a little:

```
$ sudo sysctl -w kernel.bpf_stats_enabled=1
$ sleep 10
$ sudo bpftool prog show id 312
312: tracing  name blk_account  tag 3c31e0a2c5b8d901  gpl
        loaded_at 2026-08-17T06:02:44+0000  uid 0
        xlated 512B  jited 331B  memlock 4096B  map_ids 77
        run_time_ns 41822190 run_cnt 612044
$ sudo sysctl -w kernel.bpf_stats_enabled=0
```

`41822190 / 612044 ≈ 68 ns` average per invocation, at 61,204 events/second → `61204 × 68 ns ≈ 4.2 ms` of CPU per second, i.e. **0.42% of one core**, or 0.003% of a 128-core node. That is the calculation to run before leaving anything on permanently, and it is exactly what Netflix's `bpftop` automates: it toggles `BPF_ENABLE_STATS` while it runs and renders per-program average runtime, events/sec, and estimated CPU%. Netflix reported their scheduler-hook instrumentation adding **under 600 ns per `sched_*` hook** — measured, before fleet rollout, not assumed.

**Privilege.** Before Linux 5.8 everything needed `CAP_SYS_ADMIN` (effectively root). 5.8 split it: `CAP_BPF` for loading programs and creating maps, `CAP_PERFMON` for the tracing/profiling types, `CAP_NET_ADMIN` for network attach points. Unprivileged BPF has been progressively locked down after a decade of speculative-execution issues — `kernel.unprivileged_bpf_disabled` is 1 or 2 by default on most distros, and setting it to 1 is a one-way door until reboot. **Design for privileged loaders:** an eBPF agent needs `CAP_BPF`+`CAP_PERFMON` (in Kubernetes, `securityContext.capabilities.add`, plus `hostPID`/host mounts for symbol resolution) — plan for it rather than discovering it at deploy time.

**Non-obvious costs and hazards, in the order they bite:**

- **Hot-path multiplication.** A 68 ns probe on a function called 50,000 times/second is free; the same probe on `__kmalloc` — millions of calls/second — is not. Sample the rate first (`bpftrace -e 'kprobe:__kmalloc { @ = count(); }'` for five seconds) before attaching anything expensive.
- **Uprobes are ~1.7 µs each** and trap from userspace into the kernel — the one category where a "read-only" tool can measurably slow a production workload.
- **Silent failures.** `bpf_probe_read_kernel()` returns `-EFAULT` on a non-present page; `bpf_ringbuf_reserve()` returns NULL when the ring is full; kretprobes drop returns when `maxactive` is exhausted. All three leave you aggregating a plausible-looking wrong answer.
- **Map memory is charged to a cgroup** (5.11+). Large per-CPU maps in an agent pod count against its `memory.max`.
- **Sleepable programs are the deliberate exception to "cannot block."** Most program types run in an atomic, interrupt-like context: no mutexes, no page faults you can wait on, no sleeping helpers. Program types marked `BPF_F_SLEEPABLE` (`SEC("fentry.s/...")`, `SEC("lsm.s/...")`, and sleepable uprobes; introduced in 5.7 and extended since) run where sleeping is safe and may call helpers such as `bpf_copy_from_user()` that can fault in a userspace page. They pay extra RCU/trampoline bookkeeping, so you opt in only when the hook genuinely needs to touch pageable memory — a security decision reading a userspace path, for example.

### 10. bpftrace: the whole language, on one page

bpftrace compiles a small awk-like language to BPF at runtime. Every program is a list of blocks:

```
probe /filter/ { action }
```

**Probes** — `type:target`, comma-separated for multiple, wildcards allowed:

| Form | Fires on |
|---|---|
| `kprobe:vfs_read` / `kretprobe:vfs_read` | Kernel function entry / return |
| `fentry:vfs_read` / `fexit:vfs_read` | Same, via BPF trampoline; `fexit` sees args *and* `retval` |
| `tracepoint:syscalls:sys_enter_openat` | A static tracepoint; args via `args.<field>` |
| `rawtracepoint:sched_switch` | Raw tracepoint args, lower overhead |
| `uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc` | Userspace function entry |
| `uretprobe:/usr/lib/libcuda.so.1:cuMemAlloc_v2` | Userspace function return |
| `usdt:/usr/lib/jvm/.../libjvm.so:hotspot:gc__begin` | A statically-defined app probe |
| `software:page-faults:1` / `hardware:cache-misses:1000000` | Perf software / hardware event, every Nth |
| `profile:hz:99` | Timed sample on every CPU, 99 times a second |
| `interval:s:1` | Once per second, on one CPU |
| `BEGIN` / `END` | Script start / exit |
| `watchpoint:0xffffffff81c0a000:8:w` | Memory write watchpoint |

**Filters** are C expressions in `/.../`: `/pid == 1234/`, `/comm == "python3"/`, `/args.ret > 0/`, `/cgroup == 12345/`.

**Builtins** available inside an action:

| Builtin | Value |
|---|---|
| `pid`, `tid`, `uid`, `gid` | Process/thread/user ids |
| `comm` | Process name (16 bytes, `TASK_COMM_LEN`) |
| `cpu` | Current CPU number |
| `nsecs` | Monotonic nanoseconds |
| `elapsed` | Nanoseconds since the program started |
| `cgroup` | Current cgroup id (match against `/sys/fs/cgroup` inode) |
| `curtask` | Pointer to `struct task_struct` |
| `args` | Struct of probe arguments (`args.ret`, `args.filename`) |
| `arg0`…`arg9` | Positional args for kprobe/uprobe |
| `retval` | Return value in `kretprobe`/`uretprobe`/`fexit` |
| `func`, `probe` | Probed function name / full probe name |
| `kstack`, `ustack` | Kernel / user stack (as a map key or printed) |

**Aggregating functions** — assigned to a map variable (`@name`, optionally keyed `@name[k1, k2]`):

| Function | Produces |
|---|---|
| `count()` | Event count |
| `sum(v)`, `avg(v)`, `min(v)`, `max(v)` | Scalar aggregates |
| `hist(v)` | Power-of-two histogram (`[4K, 8K)` style buckets) |
| `lhist(v, min, max, step)` | Linear histogram with fixed bucket width |
| `stats(v)` | count + average + total in one |

**Other functions:** `printf()`, `str(ptr)` (char\* → string), `buf(ptr, len)`, `ksym(addr)`, `usym(addr)`, `ntop(addr)` (IP formatting), `time()`, `strftime()`, `delete(@m[key])`, `clear(@m)`, `zero(@m)`, `print(@m)`, `exit()`, `signal()`, `system()` (requires `--unsafe`).

bpftrace prints every `@` map automatically on exit, which is why most one-liners have no `END` block.

**The two idioms that cover most real work.** First, keyed counting:

```
# Which process is doing the most syscalls? Aggregated in-kernel, one probe.
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

Second, latency timing — record a timestamp on entry keyed by thread id, subtract on return, delete the key so the map does not grow without bound:

```
# Latency distribution of vfs_read(), in microseconds.
bpftrace -e '
  kprobe:vfs_read  { @start[tid] = nsecs; }
  kretprobe:vfs_read /@start[tid]/ {
      @us = hist((nsecs - @start[tid]) / 1000);
      delete(@start[tid]);
  }'
```

The `/@start[tid]/` filter matters: without it, a return that arrives after you attached but whose entry did not (the probe raced with an in-flight call) computes `nsecs - 0`, producing a nonsense multi-decade latency that lands in the top histogram bucket and ruins the picture. Every latency one-liner you write should have that guard.

A one-liner catalogue worth memorizing — each answers a fleet question:

```
# Files opened across the node, with the process that opened them
bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%-16s %-7d %s\n", comm, pid, str(args.filename)); }'

# Block I/O size distribution per device (bytes)
bpftrace -e 'tracepoint:block:block_rq_issue { @[args.dev] = hist(args.bytes); }'

# TCP retransmits, by process
bpftrace -e 'kprobe:tcp_retransmit_skb { @[comm, pid] = count(); }'

# Which cgroup is issuing reads (map the id back: find /sys/fs/cgroup -inum <id>)
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @[cgroup] = count(); }'

# Kernel stack traces for a hot function — where is this called from?
bpftrace -e 'kprobe:__kmalloc { @[kstack] = count(); }'
```

### 11. The three toolchains, and a real libbpf program

| | bpftrace | BCC | libbpf + CO-RE |
|---|---|---|---|
| You write | A DSL script | Python + embedded C | C (kernel) + C/Go/Rust (user) |
| Compiled | At runtime, in-process LLVM | At runtime, on the target | **Ahead of time, once** |
| Needs on target | bpftrace binary, BTF | LLVM/clang + kernel headers (100s of MB) | Nothing but the kernel's BTF |
| Startup | ~0.1–1 s | seconds | milliseconds |
| Portability | Good (BTF-based) | Recompiles per host — fragile | **One binary, all kernels** |
| Ceiling | Tracing only, no daemon | Full tools, heavy runtime | Anything; this is what ships |
| Use it for | 2am investigation | Its existing tool catalogue | Production agents |

**Rule of thumb: bpftrace to investigate, libbpf/CO-RE to ship, BCC only for the tools it already has** (`execsnoop`, `biolatency`, `runqlat`, `tcplife` — many now have libbpf rewrites in `bcc/libbpf-tools`, which is the version to prefer).

Here is a complete, annotated libbpf CO-RE program. It answers a real GPU-fleet question: *which cgroup's block writes are slow, and how slow?* It keeps a per-cgroup latency histogram in a per-CPU map (cheap, always-on) **and** streams only the outliers — I/Os slower than a threshold — through a ring buffer.

```c
/* biolat.bpf.c — per-cgroup block I/O latency, with outlier events.
 * Build: clang -O2 -g -target bpf -c biolat.bpf.c -o biolat.bpf.o
 *        bpftool gen skeleton biolat.bpf.o > biolat.skel.h                 */
#include "vmlinux.h"                 /* bpftool btf dump file /sys/kernel/btf/vmlinux format c */
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>

char LICENSE[] SEC("license") = "GPL";   /* required: many helpers are GPL-only */

#define MAX_SLOTS 27                 /* 2^0 .. 2^26 microseconds ≈ 67 s */

struct hist_key { __u64 cgroup_id; };
struct hist     { __u64 slots[MAX_SLOTS]; };

struct outlier {                     /* what we stream for slow I/Os only */
    __u64 cgroup_id;
    __u64 delta_us;
    __u32 dev;
    __u32 nr_sector;
    char  comm[16];
};

/* Entry timestamps, keyed by request pointer. LRU so a lost completion
 * (device error, hotplug) cannot grow this map without bound.            */
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 16384);
    __type(key, __u64);              /* struct request * as an integer   */
    __type(value, __u64);            /* start timestamp, ns              */
} start SEC(".maps");

/* The cheap plane: per-CPU histograms, no locking, summed by userspace. */
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_HASH);
    __uint(max_entries, 1024);
    __type(key, struct hist_key);
    __type(value, struct hist);
} hists SEC(".maps");

/* The expensive plane: one record per outlier, 256 KiB ring.            */
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} events SEC(".maps");

const volatile __u64 min_report_us = 10000;   /* .rodata: set from userspace
                                                 before load; the verifier
                                                 treats it as a constant   */

/* fentry, not kprobe: typed `struct request *`, ~24 ns instead of ~137. */
SEC("fentry/blk_mq_start_request")
int BPF_PROG(on_start, struct request *rq)
{
    __u64 key = (__u64)rq, ts = bpf_ktime_get_ns();
    bpf_map_update_elem(&start, &key, &ts, BPF_ANY);
    return 0;
}

SEC("fentry/blk_mq_end_request")
int BPF_PROG(on_end, struct request *rq, int error)
{
    __u64 key = (__u64)rq;
    __u64 *tsp = bpf_map_lookup_elem(&start, &key);
    if (!tsp)                        /* REQUIRED: lookup returns
                                        PTR_TO_MAP_VALUE_OR_NULL          */
        return 0;

    __u64 delta_us = (bpf_ktime_get_ns() - *tsp) / 1000;
    bpf_map_delete_elem(&start, &key);

    struct hist_key hk = {};         /* {} zeroes padding — otherwise the
                                        verifier sees NOT_INIT bytes      */
    hk.cgroup_id = bpf_get_current_cgroup_id();

    struct hist *h = bpf_map_lookup_elem(&hists, &hk);
    if (!h) {
        struct hist zero = {};
        bpf_map_update_elem(&hists, &hk, &zero, BPF_ANY);
        h = bpf_map_lookup_elem(&hists, &hk);
        if (!h)
            return 0;                /* still must re-check: the verifier
                                        has no memory of the update       */
    }

    /* log2 bucket index, computed with a bounded loop (verifier-friendly
     * since 5.3; before that this had to be fully unrolled).             */
    __u32 slot = 0, v = delta_us;
    for (int i = 0; i < MAX_SLOTS && v; i++) { v >>= 1; slot++; }
    if (slot >= MAX_SLOTS)
        slot = MAX_SLOTS - 1;        /* REQUIRED: bounds the array index,
                                        or the verifier rejects the write */
    __sync_fetch_and_add(&h->slots[slot], 1);

    if (delta_us < min_report_us)    /* the common case exits here:
                                        no ring-buffer traffic at all     */
        return 0;

    struct outlier *e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e)                          /* ring full: drop, never block      */
        return 0;
    e->cgroup_id = hk.cgroup_id;
    e->delta_us  = delta_us;
    e->dev       = BPF_CORE_READ(rq, q, disk, major);   /* CO-RE relocated */
    e->nr_sector = BPF_CORE_READ(rq, __data_len) >> 9;
    bpf_get_current_comm(&e->comm, sizeof(e->comm));
    bpf_ringbuf_submit(e, 0);        /* verifier enforces submit-or-discard
                                        on every path from reserve         */
    return 0;
}
```

And the userspace half, using the generated skeleton:

```c
/* biolat.c — loader. Build: clang -O2 biolat.c -lbpf -lelf -lz -o biolat */
#include <stdio.h>
#include <signal.h>
#include <bpf/libbpf.h>
#include "biolat.skel.h"

static volatile bool stop;
static void on_sigint(int _) { stop = true; }

static int on_event(void *ctx, void *data, size_t sz)
{
    const struct outlier *e = data;
    printf("SLOW  cgroup=%llu  %s  %llu us  %u sectors\n",
           e->cgroup_id, e->comm, e->delta_us, e->nr_sector);
    return 0;
}

int main(void)
{
    struct biolat_bpf *skel = biolat_bpf__open();     /* parse embedded ELF */
    if (!skel) return 1;
    skel->rodata->min_report_us = 20000;              /* 20 ms threshold,
                                                         baked in before load */
    if (biolat_bpf__load(skel))   return 1;           /* verify + JIT        */
    if (biolat_bpf__attach(skel)) return 1;           /* create bpf_links    */

    struct ring_buffer *rb =
        ring_buffer__new(bpf_map__fd(skel->maps.events), on_event, NULL, NULL);

    signal(SIGINT, on_sigint);
    while (!stop)
        ring_buffer__poll(rb, 200 /* ms */);          /* epoll under the hood */

    /* On exit: walk `hists` with bpf_map_get_next_key + lookup, summing the
     * per-CPU copies, one histogram per cgroup id.                          */
    ring_buffer__free(rb);
    biolat_bpf__destroy(skel);                        /* closes fds → freed  */
    return 0;
}
```

Four decisions in that listing are the lesson in miniature: **fentry over kprobe** (typed args, 5× cheaper), **per-CPU map for the always-on aggregate**, **ring buffer only for the rare outlier**, **LRU for the in-flight table** so a lost completion cannot leak memory.

### 12. eBPF as the production datapath: Cilium, Hubble, and what they see

Cilium implements pod networking, service load balancing, and network policy in eBPF rather than iptables. Two of its hooks matter for lesson 07's loose ends:

- **Socket-level load balancing.** A `cgroup/connect4` program runs *inside the `connect(2)` syscall*, before a packet exists. It looks up the ClusterIP in a BPF map of services and rewrites the destination address and port to a backend pod's address right there. The connection is established directly to the backend, so **no per-packet DNAT and no netfilter conntrack entry is needed for that translation** — which is exactly why an eBPF datapath sidesteps the conntrack-table exhaustion you reproduced in lesson 07. (Conntrack itself does not vanish; Cilium keeps its own connection-tracking state in BPF maps, sized and evicted on its own terms, and pinned under `/sys/fs/bpf/tc/globals/cilium_ct4_global`.)
- **tc ingress/egress programs** on each pod's veth handle policy enforcement and observability with the full `sk_buff` available.

**Hubble** is the observability layer on top, and it reports what `iptables -j LOG` structurally cannot because identity is resolved *in the datapath*: Cilium assigns each pod's label set a numeric **security identity**, keeps the IP→identity mapping in a BPF map, and the datapath program stamps it into every flow record. The event therefore arrives already carrying source and destination pod, namespace and service — not an ephemeral IP you must correlate later against cluster state that has since changed — plus the verdict and drop reason, because the same program made the decision.

```
$ hubble observe --namespace training --verdict DROPPED --last 3
Aug 17 09:41:02.118  training/dataloader-7f9c-x4k2:52344  training/feature-svc:8080
    to-endpoint  FORWARDED  TCP Flags: SYN
Aug 17 09:41:07.902  training/trainer-0:41022  10.0.9.44:2049
    to-stack     DROPPED    Policy denied  TCP Flags: SYN
Aug 17 09:41:08.410  training/trainer-0:41022  kube-system/coredns:53
    to-endpoint  FORWARDED  DNS Query: nfs-store.storage.svc.cluster.local. A
```

*(Representative output.)* Line two answers a whole class of incidents: a trainer's checkpoint write to an NFS server on port 2049 denied by policy — named by pod, with verdict and reason, from one command. Reconstructing that from iptables counters and packet captures is an afternoon.

### 13. Attributing GPU-adjacent host work

NVIDIA's stack (DCGM, `nvidia-smi`, CUPTI) instruments the GPU. What no GPU tool sees is the **host-side work around it** — where stalls usually live. Three angles that work on a real node:

1. **The host path under the GPU.** Data loaders, checkpoint writes and dataset reads are ordinary VFS/block/network activity; key them by cgroup id for per-pod attribution no GPU dashboard can give you:

```
# Bytes read per cgroup — which pod is saturating the dataset mount?
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args.ret > 0/ {
    @bytes[cgroup] = sum(args.ret); }'

# Resolve a cgroup id to a path (the id is the cgroup directory's inode):
#   find /sys/fs/cgroup -inum <id>
```

2. **CUDA/NCCL uprobes — and the correction the old version of this lesson got wrong.** Attaching a uprobe needs a `FUNC` symbol in the binary's `.dynsym` or `.symtab`, **not** debuginfo. Exported CUDA driver and NCCL entry points are dynamic symbols by construction — they have to be, or the application could not link against them — so this works even against a stripped, minimal image:

```
$ bpftrace -l 'uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:*' | head -5
uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuInit
uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuMemAlloc_v2
uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuMemcpyHtoD_v2
uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuLaunchKernel
uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuStreamSynchronize

# How long does the host block in cuStreamSynchronize? (host-side GPU wait)
$ bpftrace -e '
  uprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuStreamSynchronize { @s[tid] = nsecs; }
  uretprobe:/usr/lib/x86_64-linux-gnu/libcuda.so.1:cuStreamSynchronize /@s[tid]/ {
      @wait_us = hist((nsecs - @s[tid]) / 1000); delete(@s[tid]); }'
```

   Without DWARF you lose *typed* argument access — `args.stream` will not resolve and you fall back to `arg0` plus knowledge of the ABI. Without any symbols (static or inlined functions) you lose the name and are down to addresses. The container-specific gotcha: the uprobe path resolves **on the host**, so point at the library inside the container (`/proc/<pid>/root/usr/lib/...`), not the host's copy, which may be a different driver version. And budget the ~1.7 µs per call before probing `cuLaunchKernel`, which a training loop calls thousands of times a second.

3. **Driver ioctls.** Userspace talks to the NVIDIA kernel driver through `ioctl(2)` on `/dev/nvidia*` and `/dev/nvidia-uvm`; tracing `syscalls:sys_enter_ioctl` filtered to those fds gives a coarse but vendor-independent view of driver interaction rate — useful precisely when the vendor tooling is what you suspect.

The through-line: **the GPU is a device on a host, and the host is fully observable.** When DCGM says the GPU is 20% utilized, the reason is almost always on the host path — and that path is what eBPF instruments.

## Perspectives

**Kernel-mechanism view.** Three primitives and one boundary. The **verifier** proves safety at load time by abstract interpretation — register-state lattice, tnum bit-tracking, pruning — so there is no runtime sandbox to pay for. The **JIT** turns proof-carrying bytecode into ordinary kernel text, which is why a probe costs tens of nanoseconds. **Maps** are the only channel across the kernel/userspace boundary, which is why map choice determines a tool's entire cost profile. The boundary itself — a fixed helper/kfunc API — bounds the blast radius: a BPF program cannot misuse `kfree()` because it cannot call it at all. If you can explain why `if (!p) return 0;` is a proof premise and not a style rule, you have the core.

**Operator/SRE view.** The discipline is **bpftrace-first, ship-with-libbpf**: at 2am you want a one-liner with no build step and no state left behind at Ctrl-C, and you invest in a compiled CO-RE program only after a one-liner has confirmed the hypothesis and you have measured the probe's cost. The complementary habit is `bpftool prog show` / `bpftool net show` / `ls /sys/fs/bpf` as the first move on an unfamiliar box — on a Cilium or Datadog-agent node the answer is "quite a lot," and a mysterious latency profile may already be someone else's probe.

**Hardware/cost view.** Overhead here is arithmetic, not vibes: `event_rate × per_event_ns` is the CPU you are spending. A tracepoint at 15 ns on a 50 kHz path costs 0.075% of a core; a uprobe at 1.7 µs on a 100 kHz path costs 17% of a core and will show up in someone's latency SLO. On a 128-core GPU node, 0.4% of one core to keep continuous block-latency attribution running is an obviously good trade against one recurring "the GPUs are idle and nobody knows why" incident. The same arithmetic argues the other way for anything per-packet at 10M pps — which is precisely why the network side of eBPF lives at XDP and tc, where the program *replaces* work rather than adding to it.

**GPU-fleet view.** eBPF is how you close the attribution gap between "the GPU is 20% utilized" and "why." DCGM tells you the device is idle; the host tells you what the process was doing instead — blocked on an NFS read, spinning in a Python data loader, waiting on `cuStreamSynchronize` because the previous kernel launch is still running, or starved of CPU because a neighbouring pod is throttled and holding a lock. Every one of those is visible from the host with a probe and a cgroup key, and none of them is visible from the GPU. The practical constraint to plan for is privilege and symbol availability: your agent needs `CAP_BPF`+`CAP_PERFMON`, and uprobes into a container need the library path under `/proc/<pid>/root`.

**Industry-adoption view.** The **eBPF Foundation** under the Linux Foundation counts Meta, Google, Isovalent/Cisco, Microsoft and Netflix among its members — infrastructure competitors who rarely co-govern anything. "Foundation-governed, multi-vendor kernel infrastructure, and the same substrate serves networking, security and observability rather than needing three agents" is a legitimate part of the case for investing engineering time here over a proprietary agent.

## Real-world use cases

- **[Netflix — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd)** and **[Netflix — "Announcing bpftop"](https://netflixtechblog.com/announcing-bpftop-streamlining-ebpf-performance-optimization-6a727c1ae2e5)**. Netflix instrumented scheduler hooks to detect containers whose runtime latency was being degraded by co-tenants — the continuous version of the off-CPU analysis in lesson 09. Before rolling it out fleet-wide they built `bpftop`, which toggles `BPF_ENABLE_STATS` and reports per-program average runtime, events/sec and estimated CPU%, and they published the result: **under 600 ns added per `sched_*` hook**. What it shows: the overhead conversation is settled with a number, and the number is obtained from the kernel's own accounting — the exact `run_time_ns / run_cnt` calculation in §9.
- **[Cloudflare — `ebpf_exporter` probe-overhead benchmark](https://github.com/cloudflare/ebpf_exporter/tree/master/benchmark)**. A reproducible harness that hammers `getpid()` with and without probes attached. Published results (Linux 6.5-rc1, Apple M1 under QEMU): empty probe — tracepoint 15 ns/op, fentry 24 ns/op, kprobe 137 ns/op; with a map increment, 35 / 42 / 160 ns; with a more complex key, 96 / 103 / 229 ns; uprobes separately at ~1,670 ns/call. What it shows: the attach-point taxonomy in §5 has a measured cost hierarchy, and "prefer a tracepoint, then fentry" is a performance statement, not an aesthetic one.
- **[Datadog — Universal Service Monitoring](https://www.datadoghq.com/blog/universal-service-monitoring-datadog/)**. The agent hooks the kernel TCP/UDP path — connection lifecycle, bytes, retransmits — and joins that kernel truth to orchestrator identity, producing a service map with zero application instrumentation. What it shows: §12's Cilium/Hubble pattern is not vendor-specific; kernel events plus identity resolved at collection time is the general shape of zero-instrumentation observability.
- **[AWS Open Source Blog — Pixie on Kubernetes](https://aws.amazon.com/blogs/opensource/gathering-insights-on-kubernetes-applications-services-and-network-traffic-with-pixie/)**. Pixie attaches uprobes to TLS library functions (OpenSSL's `SSL_read`/`SSL_write` and equivalents) to capture HTTP/2 and gRPC payloads *after decryption, inside the process*, without terminating TLS anywhere or touching application code. What it shows: uprobes are not limited to your own binaries — hooking a library function at the right layer gives you plaintext application semantics. It also shows the cost model in action: this is a ~1.7 µs-class hook, which is why Pixie samples rather than capturing everything.

## Worked example: finding the busiest syscall, then who is behind it

**Situation.** Node `gpu-47` runs four training pods. The team reports steps are slow. `nvidia-smi` shows SM utilization oscillating between 8% and 30%. `top` shows the node at 61% CPU with an unusually high `%sy` (system) fraction — 22% — and low `%us`. High system time means the cost is inside the kernel, which means syscalls. **Which ones, and whose?**

**Step 1 — one probe, whole-node syscall census.** `raw_syscalls:sys_enter` is a single tracepoint that fires for every syscall with a numeric id, so it is the cheapest possible way to get the shape of the workload:

```
$ sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[args.id] = count(); }'
Attaching 1 probe...
^C

@[16]:    3204        # ioctl
@[232]:  18847        # epoll_wait
@[1]:    90934        # write
@[0]:   488190        # read
```

At ~15 ns per event and ~600k events over 5 seconds, this probe cost about `600000 × 15 ns = 9 ms` of CPU total — free. (Ids are x86-64 syscall numbers; `ausyscall --dump` or `/usr/include/asm/unistd_64.h` maps them. `ksym(args.id)` does *not* work — ids are not kernel symbols — so use the per-syscall tracepoints when you want names.)

Reads dominate, ~98k/second. **Step 2 — who, and how big?**

```
$ sudo bpftrace -e '
  tracepoint:syscalls:sys_exit_read /args.ret > 0/ {
      @count[comm] = count();
      @bytes[comm] = sum(args.ret);
      @sizes      = hist(args.ret);
  }'
Attaching 1 probe...
^C

@count[fluent-bit]:  12043
@count[python3]:    401200

@bytes[fluent-bit]:   49328128
@bytes[python3]:    1643212800

@sizes:
[16, 32)              210 |                                                    |
[1K, 2K)             1104 |                                                    |
[4K, 8K)           390100 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)           10800 |@                                                   |
[64K, 128K)          1029 |                                                    |
```

Read it line by line. `python3` issued 401,200 reads returning 1.64 GB total — an average of **4,095 bytes**, and the histogram confirms it: one dominant bucket at `[4K, 8K)`. The `/args.ret > 0/` filter drops errors (`-EAGAIN`) and EOFs so they do not pollute the histogram's low bucket. Crucially, `hist()` aggregated **in the kernel**: 401,200 events became one 27-slot map, and nothing was copied per event. `strace -c` on the same process would have stopped it twice per syscall — at 98k syscalls/second that is 196k ptrace stops per second, and the process would have effectively halted.

**Step 3 — is this hitting the disk, or the page cache?** A 4 KiB read loop is only expensive if it reaches the device:

```
$ sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_read { @in_read[tid] = 1; }
  tracepoint:block:block_rq_issue /@in_read[tid]/ { @io[comm] = count(); }
  tracepoint:syscalls:sys_exit_read { delete(@in_read[tid]); }'
^C
@io[python3]: 96820
```

96,820 block requests issued from inside `read()` in 5 seconds — ~19k IOPS, roughly one device I/O per four syscalls. The reads are **missing the page cache**: the working set does not fit, or the access pattern defeats readahead.

**Step 4 — which cgroup, i.e. which pod?**

```
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_read { @[cgroup] = count(); }'
^C
@[7834]:    9021
@[9112]:  489304

$ sudo find /sys/fs/cgroup -inum 9112
/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/
  kubepods-burstable-pod3f0a1b2c_.../cri-containerd-9a1c...scope
```

**Verdict, in one line:** *the `dataloader` container in pod `3f0a1b2c` is issuing ~98k 4 KiB reads/second, ~19k of which reach the NVMe, saturating the device queue and starving the GPU of batches; the fix is a larger read block size / prefetch depth in the loader (or a dataset format with sequential access), not more CPU and not a bigger node.*

**What this cost.** Four probes, ~30 seconds of wall-clock, no redeploy, no restart of the job, and — at tracepoint prices — under 50 ms of aggregate CPU. Without eBPF the available evidence would have been "high `%sy`, high `iowait`, disk busy," which is where investigations of this shape stall for a day.

## Practice (feeds the deliverable toolkit)

**Feeds:** [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — commit these snippets directly into its diagnostic toolkit directory.

Prerequisites on your lab box: a kernel with BTF (`ls /sys/kernel/btf/vmlinux` must exist), `bpftrace` and `bpftool` installed, and `sudo`. Generate load with something real — a `dd`/`fio` loop, a small training script, or `stress-ng`.

**A. Orientation (5 min).** Inventory the node before attaching anything:

```
sudo bpftool prog show                 # what is already loaded, and by whom
sudo bpftool map show                  # and its state
sudo bpftool net show                  # tc/XDP attachments per interface
ls /sys/fs/bpf                         # pinned objects that survive their loader
bpftrace -l 'tracepoint:block:*'       # what static instrumentation exists here
```

**B. Run and read these four (30 min).** Do not just run them — write down what each output means for *your* box:

```
# 1. Which process makes the most syscalls? (find the noisy neighbour)
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# 2. Read-size distribution — small-and-frequent (fix: buffering/readahead) or large?
sudo bpftrace -e 'tracepoint:syscalls:sys_exit_read /args.ret>0/ { @ = hist(args.ret); }'

# 3. Everything exec'd node-wide (unexpected shell-outs, cron surprises, supply chain)
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve {
    printf("%-16s %-6d %s\n", comm, pid, str(args.filename)); }'

# 4. Page faults by process — who is churning memory before the OOM killer notices?
sudo bpftrace -e 'software:page-faults:1 { @[comm] = count(); }'
```

**C. Write a latency one-liner from scratch (20 min).** Time `vfs_read` with the entry/return idiom from §10 — including the `/@start[tid]/` guard and the `delete()` — then convert it to `fentry`/`fexit` and confirm both give the same distribution (`bpftrace -l 'fentry:vfs_read'` returns nothing on pre-5.5 kernels).

**D. Measure your own overhead (15 min).** Attach probe B1, then:

```
sudo sysctl -w kernel.bpf_stats_enabled=1
sleep 10
sudo bpftool prog show | grep -A2 run_time_ns
sudo sysctl -w kernel.bpf_stats_enabled=0
```

Compute `run_time_ns / run_cnt` (average per-event cost) and `run_cnt/10 × avg_ns` (CPU per second). **Write the number down** — this decides whether a tool can be left on.

**E. Stretch: cgroup attribution.** Re-key a counting one-liner by `cgroup` instead of `comm` and resolve the id with `find /sys/fs/cgroup -inum <id>` — a pod on a Kubernetes node, a systemd slice on a plain box (lesson 10).

**Acceptance (into the deliverable's diagnostic toolkit):** commit **3+ working bpftrace snippets**, each with a one-line comment stating the *fleet question it answers*, plus the measured overhead from task D recorded alongside them:

- `# Which container/process is generating the most syscalls right now? (noisy neighbour)`
- `# Are reads small-and-frequent (fix: buffering/readahead) or large? (I/O sizing)`
- `# What is being exec'd across the node? (unexpected shell-outs, supply chain)`
- `# Which process is churning memory via page faults? (allocation pressure before OOM)`

Each snippet must run clean on your kernel, and its output must be legible without you at the keyboard — a colleague should be able to run it during an incident and know what they are looking at.

## Common pitfalls

1. **Treating eBPF as "a faster strace."** The difference is architectural, not syntactic. `strace` stops the target twice per syscall via ptrace and context-switches to the tracer for every event; eBPF runs a few dozen JITed instructions in the same context and aggregates into a map. That is why one is safe on a production node under load and the other can slow a syscall-heavy process by an order of magnitude. If your eBPF tool *does* emit one userspace event per kernel event, you have rebuilt strace's cost model with extra steps — that is pitfall 5.
2. **Assuming a kprobe is as durable as a tracepoint.** kprobe targets are internal function names and signatures; they change or vanish between releases with no deprecation window, and your tool fails at *attach* time on the next kernel upgrade — usually in the middle of the upgrade, on some nodes and not others. Prefer a tracepoint; on 5.5+ prefer fentry over kprobe for typed args and 5× lower cost; treat any kprobe as version-pinned and check `available_filter_functions` on the target kernel before rollout.
3. **Confusing loaded, attached, and pinned.** Loading verifies and JITs; attaching binds to a hook; pinning into `/sys/fs/bpf` is what keeps either alive after the loader exits. A program that "was working until I closed my terminal" is an unpinned object whose last fd closed. Conversely, a *pinned* program from a crashed agent keeps running and keeps consuming CPU until someone unlinks the pin — check `bpftool prog show` timestamps against process start times when overhead appears from nowhere.
4. **Assuming an unprivileged process can load programs.** Tracing needs `CAP_BPF` + `CAP_PERFMON` (5.8+) or root, network attach needs `CAP_NET_ADMIN`, and `kernel.unprivileged_bpf_disabled` is on by default nearly everywhere. Design the agent's security context up front; discovering this at deploy time costs a sprint.
5. **Streaming when you should aggregate.** Emitting every event "in case the detail is useful later" turns a nanosecond-scale probe into a megabytes-per-second pipeline plus a userspace parser. Default to `PERCPU_HASH`/`PERCPU_ARRAY` counters and histograms; use `RINGBUF` only when the individual record *is* the answer (execs, outliers above a threshold, policy drops).
6. **Ignoring failure modes inside your own probe.** `bpf_probe_read_kernel()` returning `-EFAULT`, `bpf_ringbuf_reserve()` returning NULL on a full ring, kretprobes dropping returns past `maxactive` — each fails *quietly and plausibly*, leaving output that looks fine and is wrong. Instrument your instrumentation: keep a dropped-event counter in a map and print it.
7. **Building the compiled tool first.** A libbpf program is a day of work including the verifier fight; a one-liner is ninety seconds. Confirm the hypothesis first, then decide whether it deserves to be permanent.

## Self-check

- **Why is eBPF safe to run in the kernel — what does the verifier actually prove, and what does it forbid?**
  **Answer:** Statically, at load time, before the program executes. Pass 1 is a DFS over the instruction graph rejecting unbounded loops, unreachable code, and out-of-range jumps. Pass 2 abstractly interprets every reachable path, carrying a type per register and per stack slot (`NOT_INIT`, `SCALAR_VALUE`, `PTR_TO_CTX`, `PTR_TO_STACK`, `PTR_TO_MAP_VALUE[_OR_NULL]`, `PTR_TO_PACKET`, `PTR_TO_BTF_ID`) plus tnum bit-tracking and min/max bounds for scalars. It proves: no read of uninitialized state; every dereference in bounds (a `_OR_NULL` pointer is unusable until you branch on NULL, a packet pointer until compared against `data_end`); every array index bounded; termination. It forbids arbitrary kernel calls (helpers and kfuncs only), stack beyond 512 bytes, and blocking outside `BPF_F_SLEEPABLE` types. State pruning — skipping a branch whose state is a subset of one already verified — keeps the analysis inside the 1,000,000-instruction `BPF_COMPLEXITY_LIMIT_INSNS` budget; exceeding it gives `BPF program is too large. Processed 1000001 insn`, which is about path count, not source size. The JIT then compiles to native code, so the proof costs nothing at runtime.

- **You need to trace a kernel function on a 6.x kernel. Rank tracepoint / fentry / kprobe / uprobe and justify with mechanism and cost.**
  **Answer:** Tracepoint first (~15 ns): a compiled-in static call site patched from a no-op jump, kernel-maintained stable ABI, no trap. Then fentry (~24 ns, 5.5+): a BPF trampoline at the function's `-mfentry` patch site — no trap, and arguments arrive BTF-typed, so `req->__data_len` directly instead of `bpf_probe_read_kernel`; `fexit` also sees the return value with no instance pool to exhaust. Then kprobe (~137 ns): a breakpoint whose trap-and-return dominates, args only by register convention. Uprobe last (~1,670 ns): ~10× a kprobe, ~100× a tracepoint, and the only one that can measurably slow the traced application. Both kprobe and fentry are *unstable* attach points (internal function names), so prefer a tracepoint whenever one exists. (Costs: Cloudflare `ebpf_exporter` benchmark, Linux 6.5-rc1 — ordering durable, absolutes hardware-specific.)

- **A tracer must report block-I/O latency on a node doing 1M IOPS. Design the map layer and justify it.**
  **Answer:** In-flight timestamps in a `BPF_MAP_TYPE_LRU_HASH` keyed by request pointer, so a completion lost to a device error cannot grow the map without bound. Latency buckets in a `BPF_MAP_TYPE_PERCPU_HASH` keyed by cgroup id — one lock-free increment per completion, no cache-line bouncing; userspace sums the per-CPU copies once a second. `BPF_MAP_TYPE_RINGBUF` only for outliers above a threshold. The arithmetic: streaming 32-byte events at 1M/s is 32 MB/s plus a reserve/submit pair per event plus userspace parsing — about a core; the aggregate moves 27 buckets × N CPUs per second, five orders of magnitude less, for the same distribution. Watch memory: per-CPU maps cost `value_size × max_entries × N_CPUs`, charged to the loader's memory cgroup since 5.11.

- **What is BTF, and how exactly does CO-RE use it?**
  **Answer:** BTF is compact type metadata — struct layouts, member names and offsets, enums, function prototypes — built into the kernel with `CONFIG_DEBUG_INFO_BTF=y` and exposed at `/sys/kernel/btf/vmlinux`; `bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h` reconstructs C headers for the running kernel. CO-RE has clang emit field accesses as placeholders plus relocation records in `.BTF.ext` ("this immediate is the offset of `pid` in `struct task_struct`"). At load time libbpf reads the *target* kernel's BTF and patches each instruction with the offset that kernel really uses, before `BPF_PROG_LOAD`. Relocations also cover field existence, field size, type existence and enum values, so `bpf_core_field_exists()` lets one binary branch around a field a kernel lacks. Result: one binary, kernels you never tested, no on-target compiler or headers. It fails if a field is *renamed* (load is refused rather than reading wrong bytes) or if the kernel exports no BTF at all (BTFHub ships pre-generated blobs for those).

- **Why can't eBPF programs normally block, and what is the exception?**
  **Answer:** Most program types run in atomic, interrupt-like context — inside a kprobe trap, a tracepoint call site, a softirq — where there is no scheduler-safe way to suspend, so the verifier forbids sleeping helpers. The exception is program types flagged `BPF_F_SLEEPABLE` (`fentry.s`, `lsm.s`, sleepable uprobes; 5.7 onward), which attach where sleeping is safe and may call helpers such as `bpf_copy_from_user()` that can fault in a userspace page, at the price of extra RCU/trampoline bookkeeping. Separately: even non-sleepable programs get a fault-*tolerant* read — `bpf_probe_read_kernel()` returns `-EFAULT` on a non-present page rather than faulting.

- **You attach a uprobe to `ncclAllReduce` in a minimal GPU container image and it fails. What are the possible causes, in order?**
  **Answer:** (1) **Wrong path.** The uprobe path is resolved on the host, so you must point at the library inside the container's mount namespace — `/proc/<pid>/root/usr/lib/.../libnccl.so.2` — not the host's copy, which may be a different version or absent. (2) **Symbol not in the symbol tables.** Uprobes resolve `FUNC` symbols from `.dynsym` or `.symtab`, *not* debuginfo; exported entry points like `ncclAllReduce` are dynamic symbols and normally present even in a stripped image, but static or inlined functions are not. Check with `bpftrace -l 'uprobe:<path>:*'` or `readelf --dyn-syms`. (3) **You wanted typed arguments.** `args.<name>` needs DWARF, which stripped images do lack; fall back to `arg0`/`arg1` and the ABI. (4) **Privilege** — `CAP_BPF`+`CAP_PERFMON`, and the tracer needs to see the target's filesystem (`hostPID` and the right mounts, if it runs as a pod). Also plan for cost: at ~1.7 µs per call, a uprobe on a function in the training inner loop is a workload-affecting change, not a passive observation.

## Connections & what's next

This lesson is the mechanism behind lesson 07's Cilium datapath (the `cgroup/connect4` socket-LB hook and Cilium's BPF conntrack maps) and the substrate under lesson 09's off-CPU analysis (`sched:sched_switch` hooking). More broadly it is the module's connective tissue: cgroups (03), PSI (04), memory and reclaim (05), NUMA (06) and the network datapath (07) are all now watchable live rather than inferred from `/proc` — a probe on `try_charge` or a tracepoint on `sched_switch` turns any of those abstractions into a trace with timestamps and stacks. It is also the deliverable's foundation: the snippets you commit here are the toolkit's core.

Next: **[09 — perf / ftrace / USE method](09-perf-ftrace-use.md)**, which adds the two instruments eBPF does not replace — statistical CPU profiling with `perf` and the kernel's built-in exhaustive tracer, ftrace — and assembles PSI, eBPF, perf and ftrace into the USE-method flow that is the actual thing being tested when an interviewer says "a node is slow, walk me through it."

## References & further reading

**Primary sources**
- [bpftrace one-liner tutorial](https://bpftrace.org/tutorial-one-liners) — twelve worked one-liners, each teaching one language feature; the fastest path from zero to useful. Work it before the Practice section.
- [bpftrace reference guide (man/adoc)](https://github.com/bpftrace/bpftrace/blob/master/man/adoc/bpftrace.adoc) — the authoritative list of probe types, builtins, and functions; source for §10's tables.
- [ebpf.io](https://ebpf.io/) — the project's conceptual home: architecture, the verifier, maps, CO-RE, and the eBPF Foundation's membership.
- [Kernel docs — `Documentation/bpf/verifier.rst`](https://docs.kernel.org/bpf/verifier.html) — the register-state lattice, pointer rules, pruning, and the error strings reproduced in §4.
- [Kernel docs — `Documentation/bpf/ringbuf.rst`](https://docs.kernel.org/bpf/ringbuf.html) — why the MPSC ring replaced per-CPU perf buffers; the reserve/commit protocol and self-pacing notification of §6.
- [`bpf(2)` man page](https://man7.org/linux/man-pages/man2/bpf.2.html) — syscall commands, program types, map types (§3).
- [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) — minimal, complete CO-RE examples with skeleton generation; the scaffolding for §11's program.

**Real-world engineering blogs**
- [Netflix — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd) — what it shows: continuous scheduler-hook instrumentation in production, with per-hook overhead bounded to under 600 ns *before* rollout.
- [Netflix — "Announcing bpftop"](https://netflixtechblog.com/announcing-bpftop-streamlining-ebpf-performance-optimization-6a727c1ae2e5) — what it shows: how the overhead number is obtained — `BPF_ENABLE_STATS`, `run_time_ns / run_cnt`, per-program CPU% — i.e. the measurement discipline of §9.
- [Cloudflare — `ebpf_exporter` benchmark](https://github.com/cloudflare/ebpf_exporter/tree/master/benchmark) — what it shows: reproducible per-probe-type overhead numbers (tracepoint 15 ns, fentry 24 ns, kprobe 137 ns empty; uprobe ~1,670 ns), the basis for §5's cost hierarchy.
- [Datadog — Universal Service Monitoring](https://www.datadoghq.com/blog/universal-service-monitoring-datadog/) — what it shows: hooking the kernel TCP/UDP path to build a zero-instrumentation service map from kernel truth plus orchestrator identity.
- [AWS Open Source Blog — Pixie on Kubernetes](https://aws.amazon.com/blogs/opensource/gathering-insights-on-kubernetes-applications-services-and-network-traffic-with-pixie/) — what it shows: uprobes on TLS library functions capturing decrypted HTTP/2 and gRPC without touching application code or terminating TLS.

**Deeper dives**
- Brendan Gregg — [bpftrace intro](https://www.brendangregg.com/blog/2019-08-19/bpftrace.html) and [BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html) — the canonical tool catalogue, organised by subsystem; the book to keep on the desk.
- [Cilium and Hubble documentation](https://docs.cilium.io/) — the production datapath: socket-LB, the identity model behind Hubble's pod-labelled flows, and the pinned map layout you saw under `/sys/fs/bpf/tc/globals/`.
