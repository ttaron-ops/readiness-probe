---
lesson: "A03.8"
title: "Continuous profiling and eBPF"
module: "A-03"
concept: "on-CPU vs off-CPU, fleet-wide eBPF"
status: not-started
est_time: "4 hrs"
prev: "07-slos-and-alerting.md"
next: "09-gpu-and-ml-observability.md"
artifacts: ["offcpu-diagnosis.md"]
sources: 14
---

# A03.8 · Continuous profiling and eBPF

> **Concept.** The staff distinction is on-CPU vs off-CPU: real latency usually hides in the off-CPU stacks a CPU profiler never samples — and eBPF makes capturing it cheap enough to run continuously across the whole fleet.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

[07 · SLOs and alerting](07-slos-and-alerting.md) ends with a burn-rate alert firing correctly. This lesson is the next rung on the escalation ladder: an alert tells you a symptom is happening, not why. Profiling — and specifically the on-CPU/off-CPU split, run continuously via eBPF — is how you go from "the goodput SLI is burning budget" to "this specific node is parked in `cudaStreamSynchronize` waiting on the slowest NCCL peer." It's also the last stop before [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md), which applies the same machinery as one input to fleet-wide straggler detection alongside DCGM and NCCL signals.

**Scope, stated precisely, because two other lessons own the neighbouring ground.** The eBPF machine model — the verifier, maps, attach points, BTF/CO-RE, the load path — is `modules/01b-linux-internals` lesson 08. `perf`'s PMU sampling, flame-graph construction and reading rules, the `sched_switch` off-CPU mechanism, and stack-unwinding fundamentals are `modules/01b-linux-internals` lesson 09. **This lesson assumes all of that and never re-derives it.** What it adds is the thing neither covers: *profiles as a continuously collected, fleet-wide, queryable telemetry signal* — its data model, its cardinality, its storage cost, its symbolization pipeline, its overhead budget as a function of sampling frequency, and the comparison queries that only exist once you have thousands of nodes' profiles in one store.

Everything below is checked against **OpenTelemetry eBPF Profiler** (`main`, August 2026 — `README.md`, `cli_flags.go`, `doc/internals.md`), **Parca Agent** (`main` — `README.md` / `dist/help.txt`), **Grafana Alloy `pyroscope.ebpf`** (`main` — `docs/sources/reference/components/pyroscope/pyroscope.ebpf.md`), **Grafana Pyroscope** (`main` — `docs/sources/configure-server/reference-configuration-parameters/index.md`), the **pprof** profile schema (`google/pprof`, `proto/profile.proto`) and the **OpenTelemetry profiles** schema (`open-telemetry/opentelemetry-proto`, `opentelemetry/proto/profiles/v1development/profiles.proto`), all read from the upstream repositories because the rendered doc sites are unreachable from this environment.

## Why this matters

A senior engineer reaches for `perf`, gets a flame graph, and finds the hot function. That skill does not scale in three separate ways, and each failure is a different kind of expensive.

**It does not scale in time.** The interesting regression happened at 03:40 and resolved itself by 04:10. Nobody was logged in. `perf record` is a decision to capture *now*, and by the time a human decides, the evidence is gone. Continuous profiling makes the past queryable, which converts profiling from an interactive skill into a *data* problem — and data problems are the ones a platform team can own.

**It does not scale in breadth.** "Which of the 4,000 nodes regressed" is not answerable by SSH-ing to one of them. It is answerable by a query over a store keyed on labels, which is only possible if profiles from every node are already there, already symbolized, already tagged with the same label schema your metrics use.

**It does not scale in comparison.** The most valuable profiling question is almost never "where does time go" — it is "**what changed**": between this deploy and the last, between the slow node and its 400 peers, between the driver rollout and the week before. A comparison needs two populations, and one of them is always in the past.

The cost of not having it is measured in the same currency as lesson 07: a 3% CPU regression that ships in a release and is never noticed is, on a 4,000-node fleet, roughly 120 nodes' worth of hardware burned continuously. Nobody pages for 3%. A differential flame graph finds it in one query.

## What's new here (calibration)

- **Skip (owned by `modules/01b-linux-internals`):** the eBPF verifier, maps and attach points; `perf`'s PMU/NMI sampling loop; how a flame graph is folded and how to read it; the `sched_switch` off-CPU mechanism; frame pointers vs DWARF as a general topic.
- **New: the profile as a data structure.** pprof's string/location/function tables and OTel's `ProfilesDictionary`, and the byte arithmetic showing why a naive representation is ~20× larger than an index-based one. This is what makes fleet-wide continuous collection affordable at all.
- **New: the pipeline with an overhead budget per stage**, with the real defaults — 19–20 Hz sampling, 5 s aggregation, 15 s collection interval, 5 s ± 20% reporter interval, 64-bit in-kernel trace hashes promoted to 128-bit in userspace, backend symbolization keyed on a content-derived file ID.
- **New: probabilistic profiling as a fleet-level sampling knob** (`--profiling-probabilistic-threshold` / `--off-cpu-threshold`), and the arithmetic for when host-sampling beats frequency-reduction.
- **New: the cardinality and cost model for a profile store** — Pyroscope's real limits (`max_global_series_per_tenant: 5000`, `max_profile_size_bytes: 4194304`, `max_profile_stacktrace_samples: 16000`, `max_profile_stacktrace_depth: 1000`, `ingestion_rate_mb: 4`, `max_query_lookback: 1w`) and a worked $/month for a 4,000-node fleet against real S3 and EBS prices.
- **New: off-CPU profiling's cost model at fleet scale** — why its cost scales with *context-switch rate* rather than sample rate, and why every production agent ships it disabled by default behind a probability knob.
- **New: symbolization as the operational hard part** — the agent/backend split, the debuginfo store, and the unwind-failure metrics you must alert on or your flame graphs quietly become `[unknown]`.

## Core concepts

### 1. A profile is a signal with a shape — and the shape looks like Prometheus

Lesson 01 framed every signal by what it builds an index over. Do it for profiles.

**One profile is:** a label set, a time window, a sample type, and a map from *stack* → *value*. Written as a series identity, that is almost exactly a Prometheus series:

```
   metric:   series identity = {__name__, label₁…labelₙ}          value = float64 per timestamp
   profile:  series identity = {__name__, label₁…labelₙ, __type__} value = map[stack]→int64 per window
```

The `__name__` for a CPU profile from Alloy's eBPF component defaults to `process_cpu`, and the label set carries `service_name`, `__container_id__`, plus whatever you relabel in. The consequence is immediate and is the reason this lesson sits inside a metrics-heavy module: **every cardinality rule from lesson 01 applies unchanged, and one new dimension is added — the sample type.**

A single Go process emits `cpu`, `alloc_objects`, `alloc_space`, `inuse_objects`, `inuse_space`, `goroutines`, `mutex_count`, `mutex_duration`, `block_count`, `block_duration` — ten types. Ten types × your label set is your series count, which is why Pyroscope ships `sample_type_relabeling_rules` operating on the synthetic `__type__` and `__unit__` labels: dropping `alloc_*` from memory profiles is a first-class cardinality lever, exactly like dropping a label on a metric.

The difference from a metric is the *value*: not a float, but a map with hundreds to thousands of entries. That is what §2 has to make cheap.

### 2. The profile data model — where the compression actually comes from

Take one realistic CPU profile: a 5-second window, 19 Hz, 64 CPUs ⇒ up to `19 × 5 × 64 = 6,080` samples, collapsing to perhaps 800 unique stacks, average depth 30 frames, average symbol name 40 bytes.

**Naive representation — every frame as a string, in every stack:**

```
   800 stacks × 30 frames × (40 B name + 8 B line/offset)  = 1,152,000 B  ≈ 1.13 MiB
   per node per 5 s  ⇒  1.13 MiB × 12 × 60 × 24            ≈ 19.5 GiB / node / day
   × 4,000 nodes                                            ≈ 78 TB / day        ✗ absurd
```

**Index representation — the one both real formats use:**

pprof (`google/pprof`, `proto/profile.proto`) stores a `Profile` as parallel tables plus samples that reference them by index:

| Table | Holds | Referenced by |
|---|---|---|
| `string_table []string` | every name, filename, label key/value, unit — **once** | everything, by `int64` index |
| `function []Function` | `{name, system_name, filename, start_line}` — all string indices | `Line.function_id` |
| `location []Location` | `{mapping_id, address, []Line}` — a PC, and the (possibly inlined) lines at it | `Sample.location_id[]` |
| `mapping []Mapping` | `{memory_start, memory_limit, file_offset, filename, build_id}` per loaded object | `Location.mapping_id` |
| `sample []Sample` | `{[]location_id, []value, []Label}` | — |

and the profile header carries `sample_type[]` (e.g. `("cpu","nanoseconds")`), `period_type`, `period`, `time_nanos`, `duration_nanos`.

The OpenTelemetry profiles schema (`opentelemetry-proto`, `profiles/v1development/profiles.proto`) takes the same idea one level further: the tables are hoisted out of the individual profile into a **`ProfilesDictionary`** shared by every profile in the request — `mapping_table`, `location_table`, `function_table`, `link_table`, `string_table`, `attribute_table`, and a `stack_table`. A `Sample` there is `{stack_index, attribute_indices[], link_index, values[], timestamps_unix_nano[]}` — an entire call stack is **one int32**.

Redo the arithmetic with indices, for the same 800-stack profile:

```
   string table:  ~2,000 unique symbols × 40 B                     =  80,000 B
   function tbl:  2,000 × 4 int32                                  =  32,000 B
   location tbl:  2,500 × ~12 B                                    =  30,000 B
   stack table:   800 stacks × 30 frames × 4 B (location index)    =  96,000 B
   samples:       800 × (4 B stack index + 8 B value)              =   9,600 B
                                                                    ──────────
   uncompressed                                                    ≈ 247,600 B  ≈ 242 KiB
   after gzip/zstd (symbol text compresses ~4–6×)                   ≈  50–70 KiB
                                                                    ──────────
   vs the naive 1.13 MiB                                            ≈ 17–23× smaller
```

**And the tables amortise across time, which is the bigger win.** The symbol table for a given binary is *the same in every profile that binary appears in*. A store that deduplicates symbols across profiles (which is what Parca's columnar store and Pyroscope's symbol database both do) pays the 80 KB symbol cost once per build, not once per 5-second window. The per-window incremental cost collapses to the stack table and the samples — on the order of a few KB.

That amortisation is the entire economic basis of continuous profiling. **Profiles are cheap not because sampling is cheap, but because the interesting part of a profile is almost entirely a repeat of the previous one.**

### 3. The pipeline, with overhead at every stage

```
   CONTINUOUS PROFILING PIPELINE — ONE NODE TO ONE FLAME GRAPH
   ═══════════════════════════════════════════════════════════════════════════

   ┌─ NODE (DaemonSet, one agent per host, privileged) ───────────────────────┐
   │                                                                           │
   │  ① TRIGGER          perf_event on each CPU, 19–20 Hz                      │
   │     cost: one NMI + register read per sample                              │
   │     19 Hz × 64 CPUs = 1,216 samples/s                                     │
   │                    │                                                      │
   │                    ▼                                                      │
   │  ② UNWIND (in kernel, eBPF)                                               │
   │     native: .eh_frame-derived unwind tables preloaded into BPF maps       │
   │             → NO frame pointers and NO host debug symbols required        │
   │     interp: per-runtime unwinders (Python, JVM, Ruby, PHP, V8, .NET…)     │
   │     cost: O(depth); dominant CPU term. Bounded by MAX_FRAME_UNWINDS       │
   │           (overflow counted in UnwindErrStackLengthExceeded_total)        │
   │                    │  raw trace = [ (file_id₆₄, offset) … ]               │
   │                    ▼                                                      │
   │  ③ HASH + DEDUP (in kernel → map)                                         │
   │     64-bit trace hash in BPF (kernel memory is precious)                  │
   │     repeat stacks increment a counter instead of crossing to userspace    │
   │     cost: ~0. THIS is why eBPF profiling is cheap, not the sampling       │
   │                    │                                                      │
   │                    ▼                                                      │
   │  ④ AGENT (userspace)                                                      │
   │     64-bit BPF hash → 128-bit userland hash, cached                       │
   │     symbolizes JIT/interpreted frames HERE (symbols only exist in-process)│
   │     aggregates over --profiling-duration (Parca 5s) /                     │
   │        collect_interval (Alloy 15s)                                       │
   │     attaches labels: node, service_name, container_id, pid?, thread_comm? │
   │     cost: proportional to UNIQUE stacks, not to samples                   │
   │                    │                                                      │
   │                    ▼  ⑤ WIRE: pprof/OTLP, dictionary-encoded, gzip        │
   │                       reporter interval 5s ±20% jitter (OTel) /           │
   │                       batch write 10s (Parca).  Jitter matters: 4,000     │
   │                       agents on the same 5s boundary is a thundering herd │
   └───────────────────────┬───────────────────────────────────────────────────┘
                           │
   ┌─ BACKEND ─────────────▼───────────────────────────────────────────────────┐
   │  ⑥ SYMBOLIZATION (native frames only — deliberately NOT on the node)      │
   │     key: file_id = SHA256( file[:4096] ‖ file[-4096:] ‖ be64(len) )[:16]  │
   │          (kernels: FNV128 of the GNU build ID)                            │
   │     look up debuginfo by file_id → (function, file, line, inline frames)  │
   │     cost: once per (build, address); cached forever after                 │
   │                    │                                                      │
   │  ⑦ STORE          columnar, label-indexed, symbol-deduplicated            │
   │     retention tiers; query by label selector + time range                 │
   │                    │                                                      │
   │  ⑧ QUERY          merge N profiles → fold → render / diff                 │
   │     cost ∝ (profiles matched × unique stacks), NOT ∝ samples              │
   └───────────────────────────────────────────────────────────────────────────┘
```

Three design decisions in that diagram are worth stating as principles, because each is counter-intuitive until you see the mechanism:

**In-kernel aggregation is the whole trick, not eBPF per se.** A profiler that shipped every sample to userspace would cost the same regardless of the technology. The reason 1,216 samples/s is affordable is that identical stacks collapse to a counter increment inside the kernel and never cross the boundary. Corollary: **workloads with pathologically diverse stacks (deep recursion, heavy JIT churn) are more expensive to profile than hot loops**, and that is the one workload shape where always-on profiling can surprise you.

**Symbolization is deliberately deferred to the backend for native code.** Production binaries are stripped; shipping debuginfo to every node to symbolize locally would cost gigabytes per node and make adoption a fight with the release pipeline. Sending `(file_id, offset)` pairs and resolving centrally moves the debuginfo requirement to one place. The OTel profiler's internals document is explicit that this is a deployment-friction decision as much as a disk-space one.

**The file ID is content-derived, not path-derived.** `SHA256(head 4 KiB ‖ tail 4 KiB ‖ length)` truncated to 16 bytes identifies a binary regardless of where it is mounted, what the container image is called, or whether the build was tagged. That is what lets one symbol upload serve every node running that build — and it is why "the same binary" means byte-identical, so a rebuild that changes only a timestamp invalidates the cache.

### 4. Overhead as a function of sampling frequency — derive it, then measure it

The claim "under 1% CPU" is worth exactly nothing without the model behind it. Build the model, then say how to check it on your own hardware.

**Per-sample cost** has three terms: the interrupt and register capture (fixed, sub-µs), the unwind (proportional to stack depth and to unwinder complexity), and the map update (fixed, small). Using the frame-pointer figure derived in `01b/09` (~1–2 µs per sample) as the floor, and noting that `.eh_frame`-table unwinding in eBPF walks a preloaded binary-search structure per frame rather than following a linked list of frame pointers:

```
   OVERHEAD MODEL — one node
   ═══════════════════════════════════════════════════════════════════════════
     C_sample  = per-sample CPU cost                        [µs]
     f         = sampling frequency                         [Hz]
     N         = CPU count
     overhead  = C_sample × f × N  /  (N × 1e6 µs/s)  =  C_sample × f / 1e6

     ⇒ OVERHEAD IS INDEPENDENT OF CORE COUNT. Each core samples itself.

   Worked, at f = 19 Hz:
     C_sample = 2 µs  (shallow stacks, frame pointers)   → 0.0038 %
     C_sample = 10 µs (deep stacks, .eh_frame unwinding) → 0.019  %
     C_sample = 50 µs (pathological: depth 1000 + interp) → 0.095  %

   Same at f = 99 Hz:  0.020 % / 0.099 % / 0.495 %
   Same at f = 997 Hz: 0.20 % / 1.0 % / 4.95 %          ← the 1 % envelope breaks

   ⇒ The published "under 1 % CPU" envelope (OTel eBPF profiler README states
     1 % CPU and 250 MB memory as their upper testing limits) is a statement
     about the DEFAULT frequency, not about the technology. At 19–20 Hz you
     have roughly 1.5–2 orders of magnitude of headroom; at 1 kHz you do not.
```

**Why 19 Hz and not 20 or 100?** Two reasons, one statistical and one arithmetic. First, a prime number of samples per second is co-prime with essentially every periodic activity in the system (timer ticks, 10 ms poll loops, 1 s housekeeping), so samples do not phase-lock onto or systematically miss periodic work — the same reasoning behind `perf -F 99`. Second, and specific to continuous profiling: **the aggregation window does the work that frequency does in an interactive profile.** In an interactive `perf` session you need statistical power inside 30 seconds, so you sample at 99–999 Hz. In a continuous system you are going to merge an hour, or a day, or the whole fleet — so you can afford a low rate per node and buy statistical power by *summing*.

Make that concrete with the sampling-error formula from `01b/09` (`relative SE = sqrt((1−p)/(N·p))` for a frame with true share `p`):

```
   STATISTICAL POWER: ONE NODE INTERACTIVELY vs THE FLEET CONTINUOUSLY
   ═══════════════════════════════════════════════════════════════════════════
   (a) perf, one node, 99 Hz, 64 CPUs, 30 s, node 50 % busy
         N ≈ 99 × 64 × 30 × 0.5              ≈  95,000 samples
         a p = 0.1 % frame:  SE = sqrt(0.999/(95,000×0.001)) ≈ 10.3 %  → noise

   (b) continuous, one node, 19 Hz, 64 CPUs, 1 hour, 50 % busy
         N ≈ 19 × 64 × 3600 × 0.5            ≈ 2,190,000 samples
         a p = 0.1 % frame:  SE ≈ 2.1 %                        → usable

   (c) continuous, 4,000 nodes, 19 Hz, 5 minutes, 50 % busy
         N ≈ 19 × 64 × 300 × 0.5 × 4,000     ≈ 730,000,000 samples
         a p = 0.001 % frame: SE = sqrt(1/(7.3e8 × 1e-5)) ≈ 1.2 % → RESOLVABLE

   ⇒ The fleet store resolves frames FOUR ORDERS OF MAGNITUDE narrower than an
     interactive capture can, at a twentieth of the per-node sampling rate.
     That is the real argument for continuous profiling, and it is arithmetic,
     not preference.
```

**Then measure `C_sample` yourself instead of trusting the model.** Both agents expose their own runtime: the OTel profiler takes `-pprof <addr>` and serves Go pprof endpoints; Parca Agent serves metrics on `--http-address` (default `127.0.0.1:7071`); Alloy's component exports ~213 unwinder metrics including `UnwindNativeAttempts_total`, `UnwindNativeFrames_total`, and per-runtime attempt/frame counters. `UnwindNativeFrames_total / UnwindNativeAttempts_total` is your fleet's mean stack depth, which is the parameter `C_sample` is most sensitive to — a fleet averaging 12 frames and one averaging 90 do not have the same overhead, and now you know which you have.

### 5. Probabilistic profiling: sampling hosts instead of sampling faster

At some fleet sizes even 0.02% per node is a number someone will argue about, and the reflex — lower the frequency — is the wrong lever, because it degrades every profile equally. Both agents ship a better one.

`--profiling-probabilistic-threshold=N` with `--profiling-probabilistic-interval=D` (Parca; identical flags in the OTel profiler, `ProbabilisticThresholdMax = 100`): every interval `D`, the agent draws a random number in `[0, 100)`; if `N` is greater, it profiles for that whole interval, otherwise it does nothing. With `N = 10` and `D = 1m`, each host profiles roughly 10% of the minutes.

**Why this beats reducing frequency, in one calculation.** Suppose you want to cut profiling cost 10×:

| Strategy | Per-node overhead | Fleet samples/5 min | Per-node profile quality |
|---|---:|---:|---|
| `f = 19 Hz`, always on | 0.019% | 7.3 × 10⁸ | full |
| `f = 1.9 Hz`, always on | 0.0019% | 7.3 × 10⁷ | **10× worse on every node** |
| `f = 19 Hz`, 10% of intervals | 0.0019% (mean) | 7.3 × 10⁷ | **full, on 10% of nodes** |

Both bottom rows collect the same total samples. The difference is *where the loss lands*. Frequency reduction degrades every per-node profile uniformly — which is exactly wrong, because the per-node profile is what you need when you have already identified a suspect node. Host sampling keeps per-node profiles intact and loses coverage instead, which the fleet-aggregate query barely notices (a 10% host sample of a homogeneous fleet estimates the aggregate to within `1/sqrt(400) = 5%` at 4,000 nodes).

**The failure mode to name:** host sampling is *not* safe when the fleet is heterogeneous in the dimension you are investigating. If only 20 of 4,000 nodes run the new driver, a 10% host sample gives you 2 of them, and your "regression after driver rollout" query has a sample size of two. The rule: **probabilistic profiling for the steady-state baseline; force it to 100 on the cohort you are actively investigating.** That is a label-selector-driven config change, not a fleet-wide one.

### 6. Off-CPU profiling: why its cost model is completely different

On-CPU cost scales with *your chosen* sample rate. Off-CPU cost scales with something the workload chooses: the context-switch rate. That single difference is why every production agent ships off-CPU disabled.

```
   OFF-CPU COST, DERIVED
   ═══════════════════════════════════════════════════════════════════════════
   Capture happens on scheduler transitions, not on a timer:
     events/s = context switches/s  (per node, all CPUs)

   A busy 64-core node running a serving workload: 20,000–100,000 ctx sw/s
   (read yours: `vmstat 1` column `cs`, or /proc/stat `ctxt` delta)

   At 50,000 ctx sw/s with a 5 µs capture+unwind cost:
     50,000 × 5 µs = 250,000 µs/s = 0.25 s of CPU per second across the node
                   = 25 % of ONE core = 0.39 % of a 64-core node
   At 200,000 ctx sw/s (chatty microservice, heavy epoll):
                   = 1.0 s/s = 1 whole core = 1.6 % of the node   ← unacceptable
                     …and it is worst exactly when the node is busiest.

   ⇒ On-CPU overhead is a CONSTANT you choose.
     Off-CPU overhead is a VARIABLE the workload chooses, correlated with load.
```

Hence the design both agents converged on: `--off-cpu-threshold` is a **probability in [0, 1], defaulting to 0 (disabled)** in the OTel profiler, and `off_cpu_threshold` defaults to `0` in Alloy's `pyroscope.ebpf`. Setting it to `0.01` records one in a hundred off-CPU events, restoring a fixed expected cost:

```
   expected cost = ctx_sw/s × threshold × C_capture
   at 200,000 × 0.01 × 5 µs = 10,000 µs/s = 0.016 % of a 64-core node
```

Two consequences for how you *read* the result, which matter more than the config:

- **A probability-sampled off-CPU profile is unbiased in event count, not in duration.** Each recorded event carries its own blocked duration, so summing durations over a 1% sample and scaling by 100 estimates total blocked time correctly *in expectation* — but a rare, very long block (a 30-second NFS stall that happens twice an hour) is likely to be missed entirely. If you are hunting rare-and-long rather than common-and-short, raise the threshold temporarily on the suspect cohort instead of reasoning about the sampled aggregate.
- **The idle-thread problem gets worse in aggregate.** `01b/09` names the trap for one node: threads parked in `epoll_wait` dominate an off-CPU profile and mean nothing. Across 4,000 nodes that noise is 4,000× larger, and it will bury the signal unless you filter — by task state (uninterruptible sleep only), or by restricting to the pid/cgroup on the critical path. **At fleet scale, an unfiltered off-CPU profile is not a weak signal; it is no signal.**

### 7. Symbolization: the part that actually breaks

Unwinding gives you `(file_id, offset)`. Turning that into `torch::autograd::Engine::execute` is a separate system with its own failure modes, and in practice it is where fleet rollouts stall.

**The split, and why it is drawn where it is:**

| Frame kind | Symbolized where | Why |
|---|---|---|
| Native (C/C++/Rust/Go) | **backend**, from `(file_id, offset)` | production binaries are stripped; debuginfo is large and per-build |
| JIT (JVM, V8, .NET) | **agent**, at capture time | the symbol only exists inside the running process's memory |
| Interpreted (Python, Ruby, PHP, Perl) | **agent**, via runtime-specific unwinders | frames are interpreter data structures, not machine stacks |
| Kernel | backend, via FNV128 of the GNU build ID | one build ID per kernel; trivially cacheable |

**The debuginfo store is a real subsystem.** Parca Agent's flags spell out its shape: `--debuginfo-strip` (upload only what symbolization needs, not the whole binary), `--debuginfo-compress` (compress DWARF sections before upload), `--debuginfo-upload-queue-size=4096`, `--debuginfo-upload-max-parallel=25`, `--debuginfo-upload-timeout-duration=2m`, `--debuginfo-upload-cache-duration=5m`, and `--debuginfo-directories=/usr/lib/debug` for locally available symbols.

Size that queue against your release cadence, because it is the thing that overflows:

```
   DEBUGINFO VOLUME, WORKED
   ═══════════════════════════════════════════════════════════════════════════
   distinct binaries in the fleet             ≈   300 (services + sidecars + base)
   deploys per binary per day                 ≈     2
   ⇒ new file_ids per day                     ≈   600
   stripped+compressed debuginfo per build    ≈ 5–50 MB (language-dependent;
                                                 Go binaries are the big ones)
   ⇒ upload volume                            ≈ 3–30 GB/day  ← ONE-TIME per build
   ⇒ store growth at 90-day symbol retention  ≈ 270 GB – 2.7 TB
      at S3 Standard $0.023/GB-month           ≈ $6 – $62 / month

   The queue (4,096 entries) drains at 25 parallel × ~2 s ≈ 12/s, so 600/day
   is nowhere near it — UNTIL a fleet-wide base-image bump makes every node
   discover the same 300 new binaries within minutes. That burst is the
   overflow case, and the flag to notice is that the queue DROPS on overflow.
```

**How you find out it is broken:** you do not, unless you instrument for it. The symptom is a flame graph with `[unknown]` plateaus, and `[unknown]` is indistinguishable from "a function called `[unknown]` is hot" to anyone reading the picture casually. Alloy exposes exactly the counters to alert on:

```promql
# Native unwinding is failing more than 1 % of the time on some node.
  rate(UnwindErrStackLengthExceeded_total[10m])
/ rate(UnwindNativeAttempts_total[10m])          > 0.01

# Python symbolization is failing — usually a runtime version the agent
# does not have an offset table for. Silent, and it kills exactly the
# workloads (dataloaders, training entrypoints) you most need to see.
  rate(PythonSymbolizationFailures_total[10m])
/ ( rate(PythonSymbolizationSuccesses_total[10m])
  + rate(PythonSymbolizationFailures_total[10m]) ) > 0.05
```

Both agents also offer an explicit "tell me when you failed" mode — `--profiling-enable-error-frames` (Parca) / `-send-error-frames` (OTel profiler) — which emits synthetic frames describing the unwind failure. Turn it on during rollout, off in steady state (it inflates cardinality), and treat the resulting error-frame share as the acceptance criterion for "this agent works on this kernel and this runtime."

### 8. Cardinality and cost of the profile store

Profiles land in the same trap as every other signal in this module. Pyroscope's defaults name the walls explicitly (`docs/sources/configure-server/reference-configuration-parameters/index.md`):

| Limit | Default | What it protects |
|---|---:|---|
| `max_global_series_per_tenant` | **5000** | active profile series per tenant, cluster-wide |
| `max_label_names_per_series` | **30** | label-schema sprawl |
| `ingestion_rate_mb` | **4** MB/s | per-tenant ingest |
| `ingestion_burst_size_mb` | **2** MB | should be ≥ the largest single profile |
| `max_profile_size_bytes` | **4194304** (4 MiB) | one uncompressed profile |
| `max_profile_stacktrace_samples` | **16000** | samples per profile |
| `max_profile_stacktrace_depth` | **1000** | stacks are **truncated**, not rejected |
| `max_profile_stacktrace_sample_labels` | **100** | per-sample labels |
| `max_profile_symbol_value_length` | **65535** | symbols are **truncated**, not rejected |
| `max_query_lookback` | **1w** | queries are silently *clamped*, not failed |
| `max_query_length` | **1d** | single-query span |

Three of those have failure modes that look like a bug in your dashboard rather than a limit:

- **`max_profile_stacktrace_depth: 1000` truncates.** Deeply recursive code loses its root, so stacks that should merge into one tower fragment into many. Symptom: a flame graph that is unusually wide and shallow for code you know is deep.
- **`max_profile_symbol_value_length: 65535` truncates.** Heavily templated C++ symbols really do exceed this. Symptom: two distinct functions rendering as the same frame because their names were cut at the same prefix. (Alloy's `demangle` mode — `none` by default, up to `full` — changes symbol length dramatically, so this limit interacts with a knob most people set for readability.)
- **`max_query_lookback: 1w` clamps rather than errors.** A query for 30 days silently returns 7. Symptom: "the regression started exactly a week ago" for every regression.

**Now the cost model.** Size a 4,000-node fleet, using §2's amortised per-window figure:

```
   PROFILE STORE SIZING — 4,000 NODES
   ═══════════════════════════════════════════════════════════════════════════
   assumptions (state and re-measure yours):
     19 Hz, 64 CPUs, 15 s collect interval  ⇒ 5,760 profiles/node/day
     per-window compressed payload after symbol dedup       ≈ 6 KB
       (the 50–70 KiB from §2 is the FIRST profile for a build; steady state
        is the stack table + samples only)

   per node/day   5,760 × 6 KB                              ≈  34.6 MB
   fleet/day      × 4,000                                   ≈ 138 GB/day
   fleet/month                                              ≈   4.1 TB
   fleet/month at 15-day retention                          ≈   2.1 TB resident

   INGEST CHECK against Pyroscope's per-tenant limit:
     138 GB/day ÷ 86,400 s = 1.6 MB/s  →  under ingestion_rate_mb: 4 ✓
     …but that is ONE tenant. Split GPU nodes and service nodes into two
     tenants and each is 0.8 MB/s, with independent blast radius. Do that.

   COST, at verified list prices (us-east-1, AWS pricing API, Aug 2026):
     object storage, S3 Standard $0.023/GB-mo:
        2.1 TB × 1000 × 0.023                               ≈ $48 / month
     hot cache on gp3 at $0.08/GB-mo, 3 days resident (414 GB):
        414 × 0.08                                          ≈ $33 / month
     query/compute: 3 × r7i.4xlarge (16 vCPU / 128 GiB) @ $1.0584/h
        3 × 1.0584 × 730                                    ≈ $2,318 / month
                                                             ─────────────
     ⇒ TOTAL ≈ $2,400 / month, of which 98 % IS COMPUTE, NOT STORAGE.
```

**Read that last line, because it inverts the intuition every other signal in this module builds.** Metrics are priced in RAM per active series (lesson 01/03); logs are priced in bytes stored and bytes scanned (lesson 06). **Profiles are priced in query-time merge compute** — because answering "show me the fleet flame graph for the last hour" means merging ~4 million profiles, and the store's job is to make that merge fast, not to hold bytes cheaply. It is why profile backends are aggressive about pre-merging and why `max_query_length: 1d` exists.

Position it on the module's signal-cost gradient with real numbers, per node per day:

| Signal | Volume/node/day | What it answers | Priced in |
|---|---:|---|---|
| Metrics (DCGM, ~30 series/GPU) | ~0.5 MB stored | "is the ratio moving" | RAM per active series |
| **Profiles (19 Hz, all processes)** | **~35 MB stored** | **"which code, on which node, and how did it change"** | **query-time merge CPU** |
| Traces (tail-sampled 3%) | ~250 MB | "where did this one request go" | buffering to decide + storage |
| Logs (access + app) | ~10–50 GB | "what exactly happened to this event" | bytes ingested and scanned |

**Profiles are roughly 70× the cost of metrics and ~300–1,500× cheaper than logs, for a class of question neither can answer at all.** That ratio, not enthusiasm, is the case you make in a design review.

### 9. The query that matters is a diff

Everything above exists to make one operation cheap: comparing two populations of profiles selected by label and time. Three canonical shapes, and they cover nearly every real use:

```
   ① REGRESSION ACROSS TIME (same cohort, two windows)
        A = { service_name="inference-gateway" }  @  [now-1h, now]
        B = { service_name="inference-gateway" }  @  [now-8d, now-8d+1h]
        render: flamegraph(A) − flamegraph(B), coloured by delta
        answers: "what did last week's release change?"

   ② OUTLIER WITHIN A COHORT (two populations, same window)
        A = { pool="train-a100", node="gpu-0417" }     @ [now-30m, now]
        B = { pool="train-a100", node!="gpu-0417" }    @ [now-30m, now]
        answers: "what is THIS node doing that its 511 peers are not?"

   ③ COHORT SPLIT BY A DEPLOY DIMENSION (two populations, same window)
        A = { pool="train-a100", driver_version="550.x" }
        B = { pool="train-a100", driver_version="535.x" }
        answers: "did the driver rollout change where time goes?" — and the
        canary cohort is the reason `driver_version` must be a PROFILE LABEL,
        decided before the rollout, not after.
```

**The statistical caveat that makes a diff trustworthy** is the same `SE = sqrt((1−p)/(N·p))` from §4, now applied to a *difference*. The variance of a difference of two independent estimates adds, so the standard error of `p_A − p_B` is `sqrt(SE_A² + SE_B²)`. Worked, for shape ②:

```
   A: one node, 30 min  → N_A ≈ 19 × 64 × 1800 × 0.5      ≈  1.09e6 samples
   B: 511 nodes, 30 min → N_B ≈ 1.09e6 × 511              ≈  5.6e8  samples

   a frame at p = 2 % in A vs 1 % in B:
     SE_A = sqrt(0.98/(1.09e6 × 0.02)) = 0.67 %  (relative) → 0.0134 % absolute
     SE_B ≈ negligible
     difference 1.0 % ± 0.013 %  ⇒  ~75 σ.  Real.

   a frame at p = 0.05 % in A vs 0.03 % in B:
     SE_A = sqrt(1/(1.09e6 × 0.0005)) = 4.3 % relative → 0.0021 % absolute
     difference 0.02 % ± 0.002 %   ⇒  ~9 σ.  Also real, but you are now
     chasing 0.02 % of one node's CPU. Ask whether it can possibly matter
     before you spend an afternoon on it.
```

**The rule: the fleet side of a diff is essentially noise-free, so all the uncertainty lives in the small side.** That is a gift — it means a single suspect node can be compared against its cohort with confidence after half an hour, which is exactly the workflow §12 and lesson 09 need.

### 10. The second consumer: profiles as build-system input

Continuous profiling has two distinct payoffs, and conflating them undersells the investment.

1. **Observability.** A queryable, time-indexed store of where every node spends its cycles and its waits, for humans doing triage. Consumer: on-call.
2. **Compiler feedback.** The same pprof data fed into profile-guided optimisation — PGO/FDO at compile time, BOLT at link time — closing the loop from "where does the fleet spend cycles" to "emit a binary that spends fewer." Consumer: the build pipeline.

The second one is mechanically straightforward *because* of §2: PGO consumers want exactly a merged pprof CPU profile, which is what the store already holds. Go's PGO, for instance, consumes a `default.pgo` pprof file checked into the main package; the fleet store's "merge the last week for `service_name=X`" query produces it directly. Meta's Strobelight is the flagship public example of running this loop at scale across their largest services and feeding BOLT/FDO with it.

The ROI argument is different in kind from the triage one: it is a **continuous, measurable percentage of fleet CPU**, not an occasional faster incident. On a 4,000-node fleet at ~$1.06/hour for an r7i.4xlarge-class node, 1% of fleet CPU is `4,000 × 1.0584 × 730 × 0.01 ≈ $30,900/month`. That is the sentence that funds the platform, and it is available only if the profile store exists and is trusted. **Design for it explicitly**: retain merged per-service CPU profiles longer than raw fleet profiles, and make "produce the PGO profile for service X over window W" a supported, documented query rather than an archaeology exercise.

### 11. Deploying it: privileges, kernels, and the rollout that does not break the fleet

Fleet-wide continuous profiling means a privileged DaemonSet on every node. That is a real trust and operations boundary, and it is where a design review should spend its scepticism.

**Kernel floor and heterogeneity.** The OTel eBPF profiler states its minimum supported kernel as **5.10 or greater** on current commits (earlier commits supported 5.4 and 4.19), and maintains the floor at the lowest version shipped by actively maintained major distributions. A heterogeneous fleet almost certainly has three or four kernel versions in it; the agent must be validated on each, and `-no-kernel-version-check` exists precisely because distributions backport eBPF features so the version string lies in both directions. **Roll out by kernel version, not by percentage of fleet** — a 5% random canary across four kernel versions tests each one badly.

**Privileges.** The agent needs `CAP_BPF` + `CAP_PERFMON` on modern kernels (`CAP_SYS_ADMIN` on older ones), access to `/sys/fs/bpf` (`--bpffs-root`, default `/sys/fs/bpf/`), and the host PID namespace to resolve processes. The verifier bounds what the loaded programs can do — that is `01b/08`'s subject — but the verifier is not the whole threat model: **the agent is a userspace process reading other processes' stacks, and stacks contain addresses, symbol names, and sometimes arguments.** Two flags encode exactly that concern, and both are worth knowing before someone asks: Parca's `--metadata-enable-process-cmdline` is deprecated with an explicit warning that it "may expose sensitive information like secrets," and the OTel profiler's `-env-vars` opt-in list means environment variables reach your profile store only if you name them. Default-off, opt-in by name, and reviewed — the same discipline lesson 06 applies to log fields.

**Resource containment.** eBPF maps are preallocated kernel memory: `--bpf-map-scale-factor=0` corresponds to 4 GB of coverable executable address space, each increment doubling map size. Set container memory limits above the agent's map footprint plus its symbol caches, or the OOM killer will remove your observability at exactly the moment the node is under pressure. The published envelope to size against is **≤1% CPU and ≤250 MB memory** (the OTel profiler's own stated upper limits in testing).

**A rollout that produces evidence, in order:**

```
   1. one node per kernel version, 24 h. Accept if: agent CPU < 0.1 %, RSS
      stable, error-frame share < 2 %, and a known-hot function appears where
      you expect it in the flame graph (validate against a `perf` capture).
   2. one non-critical service's nodes, 72 h. Accept if: no change in that
      service's own SLI burn rate (lesson 07) attributable to the agent.
   3. 10 % of the fleet, one week, with probabilistic profiling at 100 (full).
      Watch ingest against `ingestion_rate_mb` and the debuginfo upload queue.
   4. full fleet, probabilistic threshold tuned to the ingest budget.
   5. THEN turn on off-CPU at a threshold derived from §6, on one cohort.
      Never enable off-CPU fleet-wide as part of the initial rollout.
```

### 12. GPU tie: profiling the host half of a GPU problem

A GPU node's CPU profile is not about the GPU — and that is precisely why it is useful. The GPU's own counters (module 05's DCGM material) tell you the device was busy or idle; the host profile tells you **why the host failed to keep it fed**, which is the actual cause in a large fraction of goodput incidents.

Three fleet-scale patterns worth naming, each mapping to a §9 query shape:

- **The starved dataloader.** On-CPU profile of the training pod shows wide plateaus in JPEG decode, `memcpy`, or Python bytecode; the GPU shows low `SM_ACTIVE`. This is a *host* bottleneck presenting as a GPU utilisation problem, and it is invisible to every GPU metric. Query shape ②: compare the suspect node's dataloader stacks against the pool's.
- **The synchronisation straggler.** Off-CPU profile shows a wide plateau under `cudaStreamSynchronize` / `ncclCommWaitUntilDone` on *most* ranks — because they are all waiting for one slow rank. The important read is inverted from the intuitive one: **the node with the *least* off-CPU sync wait is the straggler**, because everyone else is waiting on it. That single sentence is the reason this signal composes with lesson 09's per-rank step-time query rather than replacing it.
- **The driver/kernel regression.** Query shape ③, split by `driver_version`, over the same workload. This only works if `driver_version` is a profile label — which means it must be relabelled in at agent config time, before the rollout, from a node label. Add it to the label schema now, not during the incident.

Parca Agent carries a GPU-specific flag worth knowing about: **`--instrument-cuda-launch`**, which instruments calls to `cudaLaunchKernel`. That attributes host-side launch activity to the stack that issued it — the bridge between "which Python function" and "which kernel launch," on the host side of the boundary. Treat it the way §6 treats off-CPU: an extra hook whose cost scales with the workload's launch rate (a decode loop can issue tens of thousands of launches per second), so measure before enabling it fleet-wide.

**Where this stops.** Host profiling cannot see inside a kernel: SM stall reasons, warp divergence, memory-pipe saturation are device-side and belong to Nsight Compute / the PyTorch profiler / DCGM PROF counters. The escalation ladder is *continuous fleet profiling finds the node or job → device-side tooling explains the kernel*, and knowing the boundary is what keeps you from running a 4,000-node fleet's worth of Nsight captures.

## Perspectives

**Data-model.** Continuous profiling is affordable because a profile is 90% a repeat of the previous profile, and both real schemas — pprof's string/function/location tables, OTel's shared `ProfilesDictionary` — are built entirely around exploiting that. If you remember one implementation detail, make it that a call stack in the OTel model is a single `int32` index into a `stack_table`. Everything else in the cost model follows from that.

**Statistical.** The interesting inversion is that continuous profiling samples *slower* per node than interactive profiling (19 Hz vs 99–999 Hz) and yet resolves far narrower frames, because it aggregates over time and over the fleet. `N` grows by four orders of magnitude; `SE ∝ 1/sqrt(N)` does the rest. Anyone arguing that 19 Hz is "too low to be useful" is comparing against the wrong denominator.

**Economics.** Profiles invert this module's usual cost story: they are cheap to store and expensive to *query*, because the query is a merge over millions of small objects. That changes the levers — pre-merge, cap query span, tier retention by aggregation level — and it explains product decisions (`max_query_length: 1d`) that look arbitrary until you know what the query does.

**Operational trust.** The verifier bounds what in-kernel code can do; it says nothing about a privileged userspace agent reading every process's stacks, symbol names, command lines and (opt-in) environment variables. The defaults in both agents are conservative for exactly this reason. Treat the profile store's access control the same way you treat logs: it contains production internals, and "it's just stack traces" stops being true the moment someone enables cmdline capture.

**Investment case.** The compiler-feedback loop (PGO/FDO/BOLT) makes continuous profiling a line item with a percentage attached rather than a debugging convenience. On a 4,000-node fleet, 1% of CPU is ≈ $31k/month at list prices. That is a different conversation than "it helps during incidents," and it is available only because the same data serves both consumers.

## Real-world use cases

- **Meta — Strobelight** (https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/, open-sourced at https://github.com/facebookincubator/strobelight). Fleet-wide eBPF profiling wired into FDO/BOLT for their largest services, with reported CPU-cycle reductions up to 20% on targeted services. **What it shows:** the two-consumer model of §10 in production — the same profile data serving triage and the build system, with the compiler loop carrying the measurable ROI. *(Cited from prior reading; the domain is unreachable from this environment.)*

- **The OpenTelemetry eBPF Profiler's donation and design** (`open-telemetry/opentelemetry-ebpf-profiler`, `README.md` + `doc/internals.md`). A whole-system, multi-runtime profiler that unwinds native code via `.eh_frame` with **no frame pointers and no debug symbols on the host**, symbolizes native frames in the backend against content-derived file IDs, and now ships as an OTel Collector receiver. **What it shows:** the industry converged on backend symbolization and on removing the frame-pointer prerequisite, because the prerequisite was what blocked adoption — the "you must rebuild your fleet with `-fno-omit-frame-pointer`" era is ending, and a design doc written against that assumption is dated.

- **Alloy's `pyroscope.ebpf` metric surface** (`grafana/alloy`, `docs/sources/reference/components/pyroscope/pyroscope.ebpf.md`): ~213 exported counters including `UnwindNativeAttempts_total`, `UnwindErrStackLengthExceeded_total`, and per-runtime `*SymbolizationFailures_total`. **What it shows:** a mature agent treats *its own unwinding accuracy* as a first-class telemetry signal, because silent symbolization failure is the dominant real-world failure mode and it is invisible in the artefact users actually look at.

- **Off-CPU shipped disabled by default, behind a probability** (`off_cpu_threshold: 0` in Alloy; `-off-cpu-threshold` default 0 in the OTel profiler, documented as "the probability for an off-cpu event being recorded"). **What it shows:** independent confirmation of §6's cost derivation — the projects that would most like to advertise off-CPU profiling ship it off, because its cost is a function of the workload rather than of configuration.

- **eBPF Foundation / Polar Signals — cross-zone traffic attribution** (https://ebpf.foundation/case-study-polar-signals-uses-ebpf-to-monitor-internal-cross-zone-network-traffic-on-kubernetes-reducing-these-operating-costs-by-50/): eBPF used to attribute cross-zone network cost per process, cutting it by half. **What it shows:** the substrate generalises past CPU stacks — always-on, per-process attribution of *any* expensive-to-instrument resource. CPU cycles are simply the first thing everyone points it at. *(Cited from prior reading; not fetched here.)*

- **Brendan Gregg — AI Flame Graphs / Doom GPU Flame Graphs** (https://www.brendangregg.com/blog/2024-10-29/ai-flame-graphs.html, https://www.brendangregg.com/blog/2025-05-01/doom-gpu-flame-graphs.html): mixed-mode flame graphs spanning CPU launch stacks into GPU code, then validated on a non-AI GPU workload. **What it shows:** the CPU→GPU stack boundary is crossable and the technique is general-purpose rather than AI-specific — the direction §12's host-side attribution is heading. *(Cited from prior reading; not fetched here.)*

## Worked example

**Scenario.** The goodput burn-rate alert from lesson 07 fired at 02:14 for the `train-a100` pool (512 GPUs, 64 nodes × 8). Utilisation dashboards are green. You have a continuous profiling store covering the fleet. Find the cause without SSH-ing anywhere.

### Step 1 — establish what changed, before looking at any flame graph

```
   Query the store's own metadata first — it is a metrics question, not a
   profiling one, and it is 10 seconds of work:

     count by (driver_version) (
       count by (node, driver_version) (up{job="parca-agent", pool="train-a100"})
     )
   → 535.x : 52 nodes
     550.x : 12 nodes      ← a rollout is in progress. Note the cohort sizes.
```

**This is the moment probabilistic profiling would have hurt you** (§5): at a 10% host sample the 550.x cohort would contribute ~1 node. Check, and if you are host-sampling, pin the cohort to full-rate now and wait 15 minutes rather than analysing 1 node.

### Step 2 — cohort diff, shape ③

```
   A = { pool="train-a100", driver_version="550.x" }  @ [now-30m, now]
   B = { pool="train-a100", driver_version="535.x" }  @ [now-30m, now]

   Representative merged on-CPU result (top deltas, share of on-CPU samples):

     frame                                            A        B      Δ
     ─────────────────────────────────────────────────────────────────────
     libcuda.so.1 ‹unknown +0x2f1a40›                12.4 %    0.9 %  +11.5
       ← [unknown] IN THE 550.x COHORT ONLY
     torch::autograd::Engine::execute                 8.1 %    8.4 %   −0.3
     c10::cuda::CUDACachingAllocator::malloc          3.2 %    3.1 %   +0.1
     PIL._imaging.decode_jpeg                         6.6 %    6.7 %   −0.1

   Sample counts: N_A ≈ 19 × 64 × 1800 × 0.6 × 12  ≈ 1.58e7
                  N_B ≈ 19 × 64 × 1800 × 0.6 × 52  ≈ 6.83e7
   SE on the 12.4 % frame: sqrt(0.876/(1.58e7 × 0.124)) = 0.067 % relative
   ⇒ the +11.5 pp delta is overwhelming. Not noise.
```

**Read the `[unknown]` correctly.** It is not a finding about the workload; it is a finding about the *pipeline*: the new driver shipped a `libcuda.so.1` whose `file_id` has no debuginfo in the symbol store, so its frames are unresolved. Two facts follow immediately: (a) the CPU time is real and it is inside the CUDA userspace driver, and (b) your symbol store needs an entry for the new build before the flame graph becomes readable. Both matter; only the first is the incident.

### Step 3 — cross-check off-CPU, and read the inversion

Off-CPU is enabled on this pool at `off_cpu_threshold = 0.01` (§6). Query the same cohorts:

```
   off-CPU time attributed to cudaStreamSynchronize / nccl wait:

     driver_version="535.x"  (52 nodes)  : 41 % of off-CPU time  [expected —
                                            these ranks are WAITING]
     driver_version="550.x"  (12 nodes)  :  6 % of off-CPU time  [they are not
                                            waiting; everyone waits for THEM]

   ⇒ The 12 nodes on the new driver are the stragglers. The 52 healthy nodes
     show MORE synchronisation wait precisely because they are blocked on the
     slow cohort at every collective. §12's inversion, in the data.
```

### Step 4 — quantify it in the currency lesson 07 uses

```
   Step time before rollout (from the training job's own metrics): 412 ms
   Step time now:                                                  611 ms
   goodput_ratio = 412/611                                       = 0.674
   wasted fraction                                               = 0.326

   The job is synchronous, so ALL 512 GPUs run at the slow cohort's pace:
     wasted GPU-h per hour = 512 × 0.326                         = 167 GPU-h/h
   Against the 15.4 GPU-h/h sustainable rate derived in lesson 07 §12:
     burn rate = 167 / 15.4                                      = 10.8 ×
   ⇒ below the 14.4× fast tier, ABOVE the 6× slow tier. The page came from
     the slow tier, which is exactly the coverage gap the multi-tier design
     exists to close (lesson 07 §4). A single-tier 14.4× alert would have
     let this run for days.

   Cost, at the module's $2.50/GPU-hour rate-card snapshot:
     167 GPU-h/h × $2.50 = $418/hour  ≈ $10,000/day
```

### Step 5 — the counterfactual, and what it cost to have this capability

```
   WITHOUT the fleet profile store:
     · notice → SSH to a node → is it the right node? (12 of 64 are affected;
       a random pick is wrong 81 % of the time)
     · perf record on one node → symbols missing for the new libcuda anyway
     · repeat until you happen to hit an affected node
     · realistic time-to-cause: hours, and only if someone is awake

   WITH it: two label-scoped queries, ~5 minutes, at 02:20.

   WHAT IT COSTS to have it (§8, this fleet's 64 nodes rather than 4,000):
     ingest   64 × 34.6 MB/day                          ≈ 2.2 GB/day
     15-day resident                                     ≈  33 GB
     S3 Standard $0.023/GB-mo                            ≈ $0.76/month
     query tier (shared with the rest of the estate)     ≈ amortised
     agent CPU 0.019 % × 64 nodes × 64 cores             ≈ 0.78 core-equivalents
                                                          across the pool
   ⇒ The agent's own CPU cost across this pool is ~0.8 cores. The incident it
     just resolved was burning $418/hour. The ratio is not close.
```

## Practice

<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Add the profiling layer to the fleet observability design. The deliverable is a section of the design doc plus deployable config; the numbers are what gets marked.

1. **Choose the agent and justify it against your stack.** Parca, Pyroscope via Alloy's `pyroscope.ebpf`, or the OTel eBPF Profiler as a Collector receiver. Decide on three axes: does it fit your existing storage/query plane, does it need frame pointers on your fleet's binaries, and does it symbolize your runtimes (Python and the JVM specifically). State the kernel floor and enumerate the kernel versions actually present in your fleet.
2. **Write the overhead budget.** Using §4's model, state your sampling frequency, your measured or assumed `C_sample`, the resulting per-node overhead, and the fleet total in core-equivalents. Then state how you will *measure* rather than assume it — name the agent metric you will read for mean stack depth and the one you will read for unwinding failures.
3. **Decide probabilistic profiling.** Pick a threshold and interval, show the arithmetic for the resulting ingest, and write the rule for when the cohort under investigation gets pinned to 100 (§5). Name the fleet-heterogeneity case where host sampling would have misled you.
4. **Decide off-CPU explicitly.** Measure or estimate your context-switch rate (`vmstat 1`, column `cs`), compute the unsampled cost from §6, choose a threshold that bounds it under a stated budget, and state which cohorts have it enabled. "Off-CPU on everywhere" is a wrong answer; so is "off-CPU nowhere."
5. **Design the label schema, with a cardinality budget.** Enumerate every profile label, its distinct-value count, and the resulting series count against `max_global_series_per_tenant: 5000`. Include `driver_version` and whichever rollout dimensions you will need to diff on — the point of the exercise is that they must exist *before* the incident. Decide the tenant split and justify it by blast radius.
6. **Size and cost the store.** Per-node/day, fleet/day, retention, resident bytes, and the monthly figure at stated prices, split into storage and query compute. Compare it against the metrics and logs figures from lessons 01/03 and 06 and place profiles on the module's cost gradient with your own numbers.
7. **Write the symbolization plan.** Estimate distinct builds/day, debuginfo size per build, upload volume, and symbol retention. Name what happens on a fleet-wide base-image bump. Write the two alerting rules (unwind failure ratio, symbolization failure ratio) with thresholds you can defend.
8. **Write the three diff queries** of §9 as label selectors against your own schema, and for the outlier query compute the standard error of the difference at the cohort sizes you actually have. State the smallest frame-share delta your fleet can resolve in 30 minutes.
9. **Write the rollout plan** as the five-step ladder in §11, with an explicit acceptance criterion per step, including the "compare against a `perf` capture" validation on step 1.
10. **Name the second consumer.** Either the PGO/BOLT loop or a cost-attribution use, with the arithmetic for what 1% of fleet CPU is worth on your fleet at your instance prices. This is the paragraph that justifies the platform.

**Acceptance criteria:** every overhead and cost number traceable to a stated assumption; a label schema with a cardinality count against a real limit; off-CPU decided with arithmetic rather than by default; at least two alerts on the profiler's own accuracy; and a rollout plan ordered by kernel version, not by percentage.

## Common pitfalls

- **"eBPF is low-overhead, so continuous profiling is free."** The overhead is `C_sample × f`, and both terms are yours to get wrong. Symptom: a fleet-wide agent that is invisible at 19 Hz and costs 1% at 997 Hz, or that is fine on a service fleet and expensive on a deep-stack JVM fleet. Mechanism: cost is dominated by unwinding, which is O(stack depth) — measure `UnwindNativeFrames_total / UnwindNativeAttempts_total` before assuming.
- **Enabling off-CPU profiling fleet-wide "because it's the interesting signal."** Its cost tracks context-switch rate, which tracks load, so it is most expensive exactly when the fleet is busiest. Symptom: an agent whose CPU use correlates with incidents. Fix: the probability threshold, derived from your measured `cs` rate, on named cohorts only.
- **Treating `[unknown]` as a workload finding.** It is a pipeline finding: a `file_id` with no debuginfo, an unsupported runtime version, or a stack that exceeded `max_profile_stacktrace_depth: 1000`. Symptom: a wide `[unknown]` plateau that never resolves and that different engineers interpret differently. Fix: alert on the unwind/symbolization failure ratios; treat error-frame share as a rollout acceptance criterion.
- **Discovering you cannot slice by the dimension you need.** `driver_version`, `image_tag`, `model_arch` must be profile labels *before* the rollout you want to diff. Symptom: an incident where the diff query cannot be written at all. Mechanism: profile labels are attached at collection time and cannot be back-filled — same property as metric labels, same discipline.
- **Assuming the flame graph's `[unknown]`-free appearance means unwinding works.** Frame-pointer-based agents on a stripped-Python fleet produce *short, plausible-looking* stacks rather than obviously broken ones. Symptom: flame graphs where the interesting service is suspiciously shallow. Fix: validate against a known-hot function from a `perf` capture on step 1 of the rollout.
- **Ignoring `max_query_lookback: 1w`, which clamps rather than errors.** Symptom: every regression appears to have started exactly seven days ago. Mechanism: the query frontend silently narrows the range to the allowed window.
- **Building the store with no second consumer and no diff workflow.** A profile store nobody queries is a very expensive archive with a DaemonSet attached. Symptom: it is the first thing cut in a cost review, and nobody can produce a number to defend it. Fix: ship one recurring diff (weekly release regression report) and one build-pipeline consumer.
- **Confusing "the node with the most sync wait" with the straggler.** In a synchronous collective the *waiters* accumulate off-CPU time; the straggler is the one that is not waiting. Symptom: an investigation that repeatedly lands on healthy nodes. This one inverts the intuition and it costs people whole afternoons.

## Self-check

- **Why does a continuous profiler sample at 19 Hz when `perf` sessions use 99–999 Hz, and what makes that a better trade rather than a worse one?** — Statistical power comes from total sample count `N`, and `SE = sqrt((1−p)/(N·p))`. A continuous system aggregates over hours and over thousands of nodes, so `N` is four or more orders of magnitude larger than an interactive 30-second capture even at a 5× lower rate: 95,000 samples for one node in 30 s at 99 Hz, versus ~7.3 × 10⁸ for 4,000 nodes over 5 minutes at 19 Hz. The lower rate buys a 5× lower per-node overhead, and the aggregation buys back the resolution — down to frames at 0.001% of fleet CPU. 19 rather than 20 is additionally co-prime with common periodic system activity, so samples do not phase-lock onto timer ticks.

- **Where does a profile's compression actually come from, and why does that make continuous collection viable?** — From indexing, not from a compression algorithm. Both pprof and the OTel schema store symbols once in a `string_table` and reference them by integer; the OTel model goes further and makes a whole call stack a single `int32` index into a shared `stack_table`, with all tables hoisted into a per-request `ProfilesDictionary`. That takes a naive 1.13 MiB profile to ~242 KiB before compression and ~50–70 KiB after. The bigger effect is across *time*: the symbol tables for a given build are identical in every profile, so a deduplicating store pays them once per build and the incremental per-window cost falls to a few KB. Continuous profiling is affordable because consecutive profiles are almost entirely repeats.

- **Your fleet's nodes average 200,000 context switches per second. What does enabling off-CPU profiling cost, and what do you do about it?** — At ~5 µs to capture and unwind per scheduler transition, `200,000 × 5 µs = 1.0 s` of CPU per second — one full core, ~1.6% of a 64-core node, and it scales with load rather than with configuration. The fix is the probability threshold: at `off_cpu_threshold = 0.01` the expected cost is `200,000 × 0.01 × 5 µs = 10 ms/s` ≈ 0.016%. The read-side caveat: sampled off-CPU is unbiased in expectation for aggregate blocked time, but rare-and-long blocking events are likely to be missed entirely, so raise the threshold on a suspect cohort rather than reasoning about rare events from a 1% sample.

- **Native frames are symbolized in the backend and interpreted frames in the agent. Why the split, and what identifies a binary across the boundary?** — Interpreted and JIT frames only exist as data structures inside the running process, so nothing but the agent can resolve them. Native symbols live in debuginfo that production binaries are stripped of; shipping debuginfo to every node would be gigabytes per node and would couple profiling adoption to the release pipeline, so the agent sends `(file_id, offset)` and the backend resolves centrally against a debuginfo store. The `file_id` is content-derived — `SHA256(file[:4096] ‖ file[-4096:] ‖ be64(length))` truncated to 16 bytes, and FNV128 of the GNU build ID for kernels — so it identifies a binary regardless of path, image name or tag, and one symbol upload serves every node running that exact build.

- **You need to cut profiling cost 10×. Compare reducing the sample rate to 1.9 Hz against profiling 10% of hosts at full rate.** — Both collect the same total samples and cost the same. The difference is where the loss lands. Frequency reduction degrades *every* per-node profile 10×, which destroys precisely the artefact you need once you have identified a suspect node. Host sampling keeps every collected per-node profile at full fidelity and loses coverage instead, which a fleet aggregate barely notices (a 10% sample of 4,000 homogeneous nodes estimates the aggregate to ~5%). Host sampling is unsafe when the fleet is heterogeneous in the dimension under investigation — a 12-node canary cohort sampled at 10% yields one node — so pin the investigated cohort to full rate.

- **A synchronous training job is slow. The off-CPU profile shows one cohort of nodes with far *less* `cudaStreamSynchronize` wait than the rest. Which nodes are the problem?** — The cohort with *less* wait. In a barrier-bound collective every rank blocks until the slowest finishes, so healthy ranks accumulate large synchronisation wait and the straggler accumulates little — it is busy. The intuitive read ("the node with the most wait is stuck") points at the victims. Cross-check with per-rank step time (lesson 09) and with the host profile of the low-wait cohort to find what it is spending its time on.

- **What is a profile store priced in, and how does that differ from the metrics and logs stores in this module?** — Query-time merge compute. Answering "the fleet flame graph for the last hour" merges millions of small profiles, so the dominant cost is CPU in the query path, not bytes at rest: in the §8 sizing, ~98% of the monthly bill is query instances and ~2% is storage. Metrics are priced in RAM per active series (lesson 01/03) and logs in bytes ingested and scanned (lesson 06). The practical consequences are different levers — pre-merge, cap query span (`max_query_length: 1d`), tier retention by aggregation level rather than by raw age.

## Connections & what's next

This lesson is the diagnosis layer that a correctly-firing burn-rate alert from [07 · SLOs and alerting](07-slos-and-alerting.md) hands off to — the alert says *what* is burning budget, the fleet profile store says *why*, and §12's worked example closes that loop with a burn rate computed from a profiling finding. It also reuses the cardinality and cost discipline from [01 · The signal model](01-signal-model.md) and [03 · Metrics at scale](03-metrics-at-scale.md): a profile store is another always-on signal with a label schema, a per-tenant limit, and a bill, not an exception to those rules. The mechanics it deliberately does not re-teach live in `modules/01b-linux-internals` lessons [08 (eBPF)](../../../modules/01b-linux-internals/lessons/08-ebpf.md) and [09 (perf, ftrace, USE)](../../../modules/01b-linux-internals/lessons/09-perf-ftrace-use.md).

Next: [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md) — the module's synthesis lesson, which folds host-side profiling in alongside device telemetry, collective-level tracing, and serving metrics to diagnose a fleet-wide training regression rather than a single node. After that, [10 · The telemetry lakehouse](10-telemetry-lakehouse.md) takes the same "second consumer" argument to its conclusion: some questions are worth a whole second data path.

## References & further reading

**Primary sources (read from upstream repositories; rendered doc sites are unreachable from this environment)**
- OpenTelemetry eBPF Profiler — `open-telemetry/opentelemetry-ebpf-profiler`, `README.md`. *Verified: whole-system multi-runtime profiler; `.eh_frame`-based native unwinding without frame pointers or host debug symbols; stated overhead envelope of 1% CPU / 250 MB memory as upper testing limits; minimum kernel 5.10 on current commits; shipped as an OTel Collector receiver.*
- OpenTelemetry eBPF Profiler — `doc/internals.md`. *Source of the file-ID construction `SHA256(head 4 KiB ‖ tail 4 KiB ‖ be64(len))[:16]`, the FNV128-of-build-ID rule for kernels, the backend-symbolization rationale, and the 64-bit-BPF → 128-bit-userland trace hashing scheme.*
- OpenTelemetry eBPF Profiler — `cli_flags.go`. *Verified defaults: `samples-per-second = 20`, `reporter-interval = 5s`, `reporter-jitter = 0.2`, `off-cpu-threshold = 0` (probability in [0,1]; 0 disables), `probabilistic-interval = 1m`, `probabilistic-threshold` max 100, `bpffs-root = /sys/fs/bpf/`, plus the `-pin-cpu-ids` warning about biased results.*
- Parca Agent — `parca-dev/parca-agent`, `README.md` (embedded `dist/help.txt`). *Verified: `--profiling-cpu-sampling-frequency=19`, `--profiling-duration=5s`, `--remote-store-batch-write-interval=10s`, the debuginfo flags (`--debuginfo-strip`, `--debuginfo-compress`, `--debuginfo-upload-queue-size=4096`, `--debuginfo-upload-max-parallel=25`, `--debuginfo-upload-timeout-duration=2m`), `--bpf-map-scale-factor=0` ⇒ 4 GB executable address space, `--profiling-enable-error-frames`, and `--instrument-cuda-launch`.*
- Grafana Alloy — `grafana/alloy`, `docs/sources/reference/components/pyroscope/pyroscope.ebpf.md`. *Verified: `sample_rate` default 19, `collect_interval` default 15s, `off_cpu_threshold` default 0, `demangle` default `none`, `kernel_frames` default true, the injected `__name__=process_cpu` / `service_name` / `__container_id__` labels, and the unwinder/symbolization metric names used in §7's alerting rules.*
- Grafana Pyroscope — `grafana/pyroscope`, `docs/sources/configure-server/reference-configuration-parameters/index.md`. *Verified limits: `max_global_series_per_tenant: 5000`, `max_label_names_per_series: 30`, `ingestion_rate_mb: 4`, `ingestion_burst_size_mb: 2`, `max_profile_size_bytes: 4194304`, `max_profile_stacktrace_samples: 16000`, `max_profile_stacktrace_depth: 1000` (truncates), `max_profile_symbol_value_length: 65535` (truncates), `max_query_lookback: 1w` (clamps), `max_query_length: 1d`, plus `ingestion_relabeling_rules` and `sample_type_relabeling_rules`.*
- pprof profile schema — `google/pprof`, `proto/profile.proto` and `proto/README.md`. *Source of the `string_table` / `function` / `location` / `mapping` / `sample` structure, `sample_type`, `period_type`, `period`, `time_nanos`, `duration_nanos`.*
- OpenTelemetry profiles schema — `open-telemetry/opentelemetry-proto`, `opentelemetry/proto/profiles/v1development/profiles.proto`. *Source of `ProfilesDictionary` (`mapping_table`, `location_table`, `function_table`, `link_table`, `string_table`, `attribute_table`, `stack_table`) and of `Sample{stack_index, attribute_indices, link_index, values, timestamps_unix_nano}` — the single-int32-per-stack representation §2 is built on.*
- AWS Price List API — `AmazonS3` and `AmazonEC2`, `us-east-1`, retrieved 2026-08-18. *Verified list prices used in §8 and §10: S3 Standard $0.023/GB-month (first 50 TB), gp3 $0.08/GB-month, `r7i.4xlarge` on-demand Linux $1.0584/hour.*

**Real-world engineering writing (cited from prior reading; these domains are unreachable from this environment and were not fetched)**
- Meta Engineering — "Strobelight: A profiling service built on open source technology": https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/ (repo: https://github.com/facebookincubator/strobelight).
- eBPF Foundation — Polar Signals case study, cross-zone network cost reduced ~50%: https://ebpf.foundation/case-study-polar-signals-uses-ebpf-to-monitor-internal-cross-zone-network-traffic-on-kubernetes-reducing-these-operating-costs-by-50/
- Brendan Gregg — "AI Flame Graphs": https://www.brendangregg.com/blog/2024-10-29/ai-flame-graphs.html
- Brendan Gregg — "Doom GPU Flame Graphs": https://www.brendangregg.com/blog/2025-05-01/doom-gpu-flame-graphs.html
- Brendan Gregg — "Off-CPU Flame Graphs": https://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html — the canonical write-up of the technique whose *mechanism* is taught in `modules/01b-linux-internals` lesson 09 and whose *fleet cost model* is derived in §6 here.
