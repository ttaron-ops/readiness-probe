---
lesson: "07.4"
title: "vLLM in production"
module: "07"
concept: "vLLM in production"
status: not-started
est_time: "8h"
prev: "03-pagedattention-and-vllm.md"
next: "05-batching-economics.md"
artifacts: []
sources: 14
---
# 07.4 · vLLM in production

> **Concept.** The production knobs — `gpu-memory-utilization`, `max-num-seqs`, `max-model-len`, `tensor-parallel-size`/`pipeline-parallel-size` — are not independent dials; they trade against a single fixed HBM budget, and when you oversubscribe it the scheduler *preempts* (recompute or swap). Tuning is finding the edge of that budget without falling off it.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

---

## Where this fits

07.3 gave you the mechanism that makes a large batch physically fit on one GPU: paged,
block-based KV allocation, and a scheduler that re-plans every forward pass. It ended with
the knobs named but not driven — `gpu-memory-utilization`, `max-num-seqs`,
`max-num-batched-tokens`, `block-size` — and with preemption sketched as "the allocator seen
from its failure side."

This lesson drives them. It is the operational half of 07.3: what each flag physically
changes inside the engine, what its real default is on the hardware you rent, what the
startup log tells you about whether your setting took effect, and what the system does when
traffic asks for more than you provisioned. By the end you should be able to write a
production launch command, read the twelve lines of startup log that validate it, and
diagnose a preemption storm from three metrics without guessing.

It sets up 07.5 directly. The batching-economics sweep measures a *config*, not a GPU: an
oversubscribed KV pool or an unnecessary `--tensor-parallel-size 2` produces a CPM curve
that is a measurement of your own misconfiguration. The non-preempting, correctly-sized
config you land on here is the baseline that sweep is run against.

**Version pin.** Everything below is checked against **vLLM v0.27.1** (the current release
tag) and cross-checked against `main` at commit `c1e4387` (2026-08-17), reading
`vllm/config/cache.py`, `vllm/config/scheduler.py`, `vllm/config/parallel.py`,
`vllm/config/compilation.py`, `vllm/engine/arg_utils.py`, `vllm/v1/worker/gpu_worker.py`,
`vllm/v1/worker/utils.py`, `vllm/v1/core/sched/scheduler.py`, and
`vllm/v1/metrics/loggers.py`. Metric names match the verified set in
[module 05 lesson 06](../../05-gpu-observability/lessons/06-inference-slos.md). Defaults in
this engine move between minor releases; the numbers here are pinned, and the procedure for
re-deriving them from your own build is in §12.

## Why this matters

Every flag in this lesson moves cost per million tokens, and most of them move it in a way
that produces no error message when you get it wrong.

Set `--gpu-memory-utilization` to 0.60 on a card you paid for and you have fenced off 25 GB
of HBM that could have held 190,000 tokens of KV. Nothing crashes. Throughput is simply
half what you bought, the KV-cache gauge sits at 0.4 while requests queue, and your cost per
token is 2× the number your competitor quotes for the same hardware. Set
`--max-model-len 131072` because that is what the model card says, when your p99 prompt is
6,000 tokens, and you have divided your admission headroom by 16 for a capability nobody
uses. Set `--tensor-parallel-size 2` for an 8B model that fits comfortably on one card and
you have doubled your GPU bill to buy an all-reduce on every layer.

The failure that actually pages you is the fourth one: oversubscribe the KV pool and the
scheduler starts evicting running requests. In vLLM's V1 engine an eviction is not a pause —
the victim's `num_computed_tokens` is reset to zero and its entire prefill is recomputed.
A preemption storm therefore looks like a latency incident, a throughput collapse and a
utilisation *increase* all at once, because the GPU is genuinely busy doing work it is about
to throw away again. Reading that signature correctly, in three metrics, is the difference
between a five-minute config change and an afternoon of adding GPUs that do not help.

This is also the lesson that maps most directly onto the job. "Design an LLM inference
platform" interviews converge on a whiteboard version of exactly this: given a model, a GPU
and a TTFT target, name the flags, justify each number, and say what breaks first.

## What's new here (calibration)

Referenced, not re-taught: the roofline and memory-bound decode (module 03); TTFT / ITL /
TPOT and the vLLM metric names (module 05); the KV-bytes-per-token formula and the
concurrency cap (07.1, 07.2); blocks, block tables, prefix hashing and iteration-level
scheduling (07.3).

Genuinely new here:

- **What `--gpu-memory-utilization` actually computes.** Not "fraction of free memory" —
  a fraction of **total** device memory, validated against free memory at init, with a
  documented interaction with CUDA-graph memory estimation that changed the effective
  meaning of the same number in v0.21.0.
- **The three-stage startup sequence** — weight load, memory profiling, graph capture —
  and which log line proves each stage did what you asked.
- **The real, device-dependent defaults** for `max_num_seqs` and `max_num_batched_tokens`,
  including why there is an explicit A100 exclusion in the source.
- **`--kv-cache-memory-bytes`**, the deterministic alternative to utilisation-based sizing,
  and why fleets with heterogeneous co-tenancy end up there.
- **Preemption as a code path**, not a concept: who gets chosen, what state is discarded,
  where the request goes, and the two admission-conservatism dials (`scheduler_reserve_full_isl`,
  `watermark`) that exist to prevent it.
- **A correction to the concept line above.** The blockquote at the top of this lesson says
  preemption is "recompute or swap." That was true of the V0 engine. **In V1 there is only
  recompute**; the CPU-swap path and the `--swap-space` flag were removed with V0. §7 gives
  the source evidence, and the blockquote is preserved as written so the correction is
  visible rather than silently patched.
- **CPU provisioning as a GPU-performance variable**, including the `2 + N` process floor.
- **A tuning procedure with a fixed order**, because these knobs interact and changing two
  at once produces a measurement you cannot attribute.

---

## Core concepts

### 1. What `vllm serve` actually does before it accepts a request

You cannot tune what you cannot see, and almost every flag in this lesson takes effect
during a specific, observable stage of startup. The V1 engine boots in a fixed order.

```
  vLLM V1 STARTUP — PROCESSES, MEMORY, AND WHERE EACH FLAG LANDS
  ══════════════════════════════════════════════════════════════════════════════

  PROCESS LAYOUT (single node, tensor-parallel-size = N)

   ┌──────────────────────────────────────────────────────────────────────┐
   │ P0  API SERVER  (FastAPI / uvicorn)          --api-server-count      │
   │     HTTP, chat templating, TOKENISATION, detokenisation, streaming   │
   │     ── CPU-bound. Never touches the GPU. ──                          │
   └──────────────────────────┬───────────────────────────────────────────┘
                              │  ZMQ IPC (request/​output queues)
   ┌──────────────────────────▼───────────────────────────────────────────┐
   │ P1  ENGINE CORE                              --max-num-seqs          │
   │     THE SCHEDULER: block pool, prefix-cache index, admission,        │
   │     preemption, the per-iteration token budget                       │
   │     ── busy loop. CPU starvation here shows up as GPU idle gaps. ──  │
   └──────────────────────────┬───────────────────────────────────────────┘
                              │  shared memory / collective RPC
   ┌──────────────────────────▼───────────────────────────────────────────┐
   │ W0 … W(N-1)  GPU WORKERS  (one per GPU)      --tensor-parallel-size  │
   │     weights · KV blocks · CUDA graphs · attention backend            │
   └──────────────────────────────────────────────────────────────────────┘

   Minimum physical CPU cores = 2 + N   (vLLM optimization docs)
   With data parallelism:  A + DP + N + (1 if DP > 1)   where A = api-server-count

  ──────────────────────────────────────────────────────────────────────────────
  HBM ON ONE WORKER, AFTER STARTUP  (H100-80GB, Llama-3.1-8B bf16, util 0.92)

   0                                                              ~79.1 GiB
   ├────────────────────────────────────────────────────────────────────────┤
   │ weights │ non-Torch │ act.  │ CUDA │        KV BLOCK POOL         │un- │
   │ 14.99   │   0.32    │ peak  │graph │        ~55 GiB               │req.│
   │  GiB    │   GiB     │ 1.2G  │ ~1G  │   = num_gpu_blocks × 16 tok  │6.3 │
   └─────────┴───────────┴───────┴──────┴──────────────────────────────┴────┘
    ◀───────────── requested = TOTAL × 0.92 = 72.8 GiB ─────────────────▶

    NOTE: the fraction multiplies TOTAL memory, not FREE memory.
          The unrequested tail is what a co-tenant process may use.
```

The temporal view is what you will actually grep:

```
  STARTUP TIMELINE — Llama-3.1-8B on 1×H100, cold page cache
  (representative wall-clock; yours varies with storage and CPU — 07.9 measures it)
  ══════════════════════════════════════════════════════════════════════════════

  t=0s     process start, CUDA context init, distributed env init
           │  ~2–5 s   (per worker; TP>1 adds NCCL rendezvous)
           ▼
  t≈5s     ┌─ STAGE 1 · WEIGHT LOAD ────────────────────────────────────┐
           │ read safetensors → host → cudaMemcpy → HBM                 │
           │ LOG: "Model loading took 14.99 GiB memory and 11.42 s"     │
           │ FLAGS: --load-format, --dtype, --quantization              │
           │ BOUND BY: storage bandwidth, then PCIe. Not the GPU.       │
           └────────────────────────────────────────────────────────────┘
  t≈17s    ┌─ STAGE 2 · MEMORY PROFILING ───────────────────────────────┐
           │ dummy forward at max_num_batched_tokens × max_num_seqs to  │
           │ measure PEAK activation; then estimate CUDA-graph memory   │
           │ LOG: "Memory profiling takes 6.42 seconds. Total non KV    │
           │       cache memory: 16.51GiB; torch peak memory increase:  │
           │       1.21GiB; weights memory: 14.99GiB."                  │
           │ LOG: "Available KV cache memory: 55.05 GiB"                │
           │ FLAGS: --gpu-memory-utilization, --kv-cache-memory-bytes,  │
           │        --max-num-batched-tokens (sets the dummy shape)     │
           └────────────────────────────────────────────────────────────┘
  t≈24s    ┌─ STAGE 3 · KV POOL ALLOCATION ─────────────────────────────┐
           │ carve the pool into blocks, build the free list            │
           │ LOG: "GPU KV cache size: 450,816 tokens"                   │
           │ LOG: "Maximum concurrency for 8,192 tokens per request:    │
           │       55.03x"                                              │
           │ FLAGS: --block-size, --kv-cache-dtype, --max-model-len     │
           └────────────────────────────────────────────────────────────┘
  t≈25s    ┌─ STAGE 4 · COMPILE + CUDA GRAPH CAPTURE ───────────────────┐
           │ torch.compile (cache hit → seconds; miss → 30–120 s)       │
           │ then capture one graph per batch size in the capture list  │
           │ LOG: "Capturing CUDA graphs (decode, FULL_AND_PIECEWISE)"  │
           │ LOG: "Graph capturing finished in 21 secs, took 0.94 GiB"  │
           │ FLAGS: -O0..-O3, --enforce-eager, --cuda-graph-sizes       │
           └────────────────────────────────────────────────────────────┘
  t≈47s    LOG: "init engine (profile, create kv cache, warmup model)
                 took 30.12 s (compilation: 8.44 s)"
           LOG: "Application startup complete."
           ▲
           └─ ONLY NOW does the readiness probe pass.
              Everything before this is 07.9's cold-start budget.
```

Two consequences worth carrying now:

**The memory profiling stage is why your flags interact.** vLLM does not compute activation
memory from a formula; it *runs* a dummy forward pass shaped by `max_num_batched_tokens`
and `max_num_seqs` and measures the peak. Raise `max_num_batched_tokens` and the measured
activation peak rises, the non-KV term grows, and the KV pool — the remainder — shrinks.
That is a real coupling in the code path, not an analogy.

**Readiness is not liveness.** A Kubernetes `readinessProbe` that hits `/health` before
stage 4 completes will mark the pod ready while the engine is still capturing graphs. Set
`initialDelaySeconds` from a measured startup, or probe `/v1/models`, which only responds
once the engine client is up.

### 2. `--gpu-memory-utilization`: what the number actually multiplies

Default: **0.92** (`CacheConfig.gpu_memory_utilization`, v0.27.1 and main). Older tutorials
say 0.90; that was the value through the 0.11.x line.

The single most common misconception is that this is a fraction of *free* memory. It is
not. From `vllm/v1/worker/utils.py`:

```python
def request_memory(init_snapshot: MemorySnapshot, cache_config: CacheConfig) -> int:
    requested_memory = math.ceil(
        init_snapshot.total_memory * cache_config.gpu_memory_utilization
    )
    if init_snapshot.free_memory < requested_memory:
        raise ValueError(
            f"Free memory on device ... on startup is less than desired "
            f"GPU memory utilization ({cache_config.gpu_memory_utilization}, ...)"
        )
```

Read that carefully, because three behaviours fall out of two lines:

1. **The budget is `total_memory × util`.** On an 80 GB H100 that reports 79.11 GiB total,
   `--gpu-memory-utilization 0.92` requests 72.78 GiB — regardless of what else is on the
   card.
2. **It is a request, not a reservation of someone else's memory.** If a co-tenant process
   already holds 10 GiB, free memory is 69 GiB, which is *less* than the 72.78 GiB
   requested, and vLLM **fails at startup with a clear error** rather than OOMing later.
   This is why the flag's docstring says it is "a per-instance limit": two vLLM instances
   sharing a card must each be given ≤ 0.5.
3. **The unrequested tail is not headroom for vLLM.** It is space vLLM promises *not* to
   use. Headroom for vLLM's own transient allocations comes out of the requested budget,
   as the difference between the profiled peak and the actual peak under production traffic.

Then the KV pool is the remainder:

```
  available_kv_cache_memory
      = requested_memory                       (total × util)
      − non_kv_cache_memory                    (weights + non-Torch + activation peak)
      − cudagraph_memory_estimate              (if graphs will be captured)
```

**The v0.21.0 change that invalidates older tuning advice.** CUDA-graph memory used to be
unaccounted during KV sizing; since v0.21.0 vLLM profiles it and subtracts it, and the
engine says so in a log line:

```
INFO ... CUDA graph memory profiling is enabled (default since v0.21.0).
         The current --gpu-memory-utilization=0.9200 is equivalent to
         --gpu-memory-utilization=0.9082 without CUDA graph memory profiling.
         To maintain the same effective KV cache size as before, increase
         --gpu-memory-utilization to 0.9318.
```

If you carried a `0.90` from a 2024 runbook and your KV pool is smaller than you remember,
this is why. The engine is telling you the exact number to move to. Do not fight it —
graph memory was always being consumed; it just used to be consumed out of the safety
margin, which is precisely how "worked fine for months, then OOMed under a traffic shift"
happens.

**How high can you go?** The honest answer is a procedure, not a number, because the right
value depends on how far your production activation peak exceeds the profiled peak, and
that depends on your traffic's prompt-length distribution.

| Value | KV pool on 80 GB, 8B bf16 | What it costs you |
|---|---|---|
| 0.60 | ~30 GiB | ~46 % of your concurrency, silently. The expensive mistake. |
| 0.85 | ~50 GiB | Conservative. Sensible on a shared or unpredictable node. |
| **0.92** | ~55 GiB | **Default.** Right for a dedicated card with profiled traffic. |
| 0.95 | ~57 GiB | ~4 % more KV. Requires characterised traffic and a real OOM test. |
| 0.98 | ~59 GiB | Startup OOM during graph capture, or a load OOM on a prompt burst. |

Note the shape: going 0.92 → 0.95 buys about 4 % more KV pool, because weights and
activations are a **fixed subtraction** that the fraction does not touch. The upside is
small and bounded; the downside is a crash. **This is the weakest lever in the lesson with
the worst failure mode, and reaching for it to fix preemption is attacking the supply side
of a demand-side problem.**

**Failure signatures.**

- *Too high, at startup:* `torch.OutOfMemoryError: CUDA out of memory` during "Capturing
  CUDA graphs", or a vLLM error naming the requested vs available figures. Deterministic and
  loud — you find it on the first boot.
- *Too high, under load:* the process survives startup and dies hours later when a burst of
  long prefills pushes the real activation peak past the profiled one. Restart loop, no
  config change, "it was fine yesterday."
- *Too low:* nothing fails. `vllm:kv_cache_usage_perc` sits at 0.3–0.5 at saturation,
  `vllm:num_requests_waiting` climbs, `vllm:num_requests_running` plateaus well below
  `max_num_seqs`. You are paying full price for a fraction of the card.

### 3. `--kv-cache-memory-bytes`: the deterministic alternative

Profiling is adaptive, which is a virtue in development and a liability in a fleet: the
same manifest on two nodes with different co-tenants produces two different KV pool sizes,
two different concurrency caps, and two different cost-per-token curves. Load balancing
across them is then quietly unfair.

`--kv-cache-memory-bytes` sets the pool size directly, in bytes, and **ignores
`gpu-memory-utilization` entirely** (`vllm/config/cache.py`; the worker logs this
explicitly). vLLM prints the value that reproduces the current allocation at startup, so
the workflow is: boot once with utilisation-based sizing, read the suggested number, pin it
in the manifest.

```bash
# Boot 1 — let it profile, then read the suggestion out of the log.
vllm serve meta-llama/Llama-3.1-8B-Instruct --gpu-memory-utilization 0.92
# ... "Available KV cache memory: 55.05 GiB"

# Boot 2..N — pin it. Same pool on every node, every restart.
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --kv-cache-memory-bytes 59110000000
```

Three properties, from the config docstring and the worker source:

- It **skips the memory-profiling measurement** on subsequent boots, which is a real
  cold-start saving (the profiling stage is single-digit seconds, and the CUDA-graph
  estimation pass on top of it is not free). This is one of the three startup accelerators
  vLLM's optimization guide names.
- It is **only valid on the same GPU with the same initial free memory.** Change the SKU,
  add a co-tenant, or upgrade the driver, and the pinned value may no longer fit — the
  failure is an allocation error at boot, which is at least loud.
- A conservative value **caps concurrency and therefore throughput**; an optimistic one
  fails at allocation time. There is no forgiving middle, which is exactly why you derive
  it from a profiled boot rather than picking it.

Use utilisation-based sizing while you are tuning. Pin bytes when you ship, if determinism
across a fleet matters more than adaptivity.

### 4. `--max-model-len`: a reservation ceiling, not an allocation

This is the flag most often set wrong, and the mistake is always the same direction: to the
model's architectural maximum, "to be safe."

What it changes:

- It is the **maximum prompt + generated tokens** for any single request. Longer requests
  are rejected at admission with a 400, not truncated.
- It sets the **worst-case per-request block count** the scheduler plans against:
  `ceil(max_model_len / block_size)`. That is the denominator of the "Maximum concurrency"
  line printed at startup.
- With `scheduler_reserve_full_isl = True` (the default), the scheduler checks that a
  request's **full input length** fits in the pool before admitting it.
- It does **not** allocate anything. Under paged allocation a sequence holds blocks
  proportional to its *current* length. Raising `max-model-len` from 8k to 128k consumes
  zero additional bytes and divides your printed concurrency by 16.

The clean experiment — the one that makes this stick — is in 07.2's worked example: boot the
same server at 8192 and at 32768 and observe that `GPU KV cache size: N tokens` is
**identical** in both, while `Maximum concurrency` divides by exactly 4. The pool did not
change; only how you chose to divide it.

**How to set it.** From your own traffic, not the model card:

```promql
# p99 of prompt + generated tokens, over a representative week.
histogram_quantile(0.99, sum by (le) (rate(vllm:request_prompt_tokens_bucket[7d])))
  +
histogram_quantile(0.99, sum by (le) (rate(vllm:request_generation_tokens_bucket[7d])))
```

Round up to a comfortable margin (say 1.3×) and set that. If you have no history, start
from the product contract — "we accept documents up to 20 pages" is about 16k tokens — not
from `config.json`.

**`--max-model-len -1`** is a real value: it tells the engine to auto-fit the context length
to available memory after profiling (`vllm/config/model.py`). Useful for "fit the biggest
context this card allows" experiments; unsuitable for production, because your admission
contract then silently changes when a co-tenant appears.

Two edge behaviours worth knowing. Requests longer than `max-model-len` are rejected, so a
too-tight setting shows up as a rise in HTTP 400s, not as latency. And if you *disable*
chunked prefill, `max_num_batched_tokens` must be ≥ `max_model_len` or the server refuses
to start — a documented startup failure that catches people who turn chunking off to
"simplify" a benchmark.

### 5. `--max-num-seqs` and `--max-num-batched-tokens`: the two iteration budgets

These are the scheduler's own limits, distinct from the KV pool's physical limit. Every
engine step is bounded by both.

- **`--max-num-seqs`** — the maximum number of sequences in one forward pass.
- **`--max-num-batched-tokens`** — the maximum number of tokens processed in one forward
  pass, summed over all sequences in it. Under the V1 scheduler's unified "tokens owed"
  model (07.3 §7), one decode step of a resident sequence costs 1 token against this
  budget; a prefill chunk costs however many prompt tokens it covers.

The dataclass defaults in `vllm/config/scheduler.py` are `DEFAULT_MAX_NUM_SEQS = 128` and
`DEFAULT_MAX_NUM_BATCHED_TOKENS = 2048`, but the source comments say plainly that these are
"mainly for convenience when testing" — the values you actually get come from
`EngineArgs.get_batch_defaults(world_size)`, which switches on device memory, device name
and usage context:

| Device / context | `max_num_batched_tokens` | `max_num_seqs` |
|---|---|---|
| ≥ 70 GiB and **not** A100 (H100/H200/MI300X…), OpenAI API server | **8192** | **1024** |
| ≥ 70 GiB and not A100, offline `LLM` class | 16384 | 1024 |
| Everything else (incl. **A100**, L40S, A10, <70 GiB), API server | 2048 | 256 |
| Everything else, offline `LLM` class | 8192 | 256 |
| TPU v6e / v5e / v5p (API server) | 1024 / 512 / 256 | — |

**The A100 exclusion is a real, load-bearing string comparison in the source**, with the
comment: *"Setting large `max_num_batched_tokens` for A100 reduces throughput, see PR
#17885."* Two cards with the same 80 GB therefore boot with a 4× different token budget,
because the A100's memory-system behaviour under large fused prefill batches differs from
Hopper's. If you benchmark an A100 with H100 flags copied from a blog post, you are
measuring the wrong configuration and the regression is not your model.

**What each budget costs you when it binds.**

`max_num_seqs` binding is easy to see and easy to fix: `vllm:num_requests_running` sits
*exactly* on the configured value while `vllm:kv_cache_usage_perc` is low. Memory is not the
constraint; the flag is. Raise it. The only real cost of a high `max_num_seqs` is that it
shapes the memory-profiling dummy run (a bigger profiled activation peak, so a slightly
smaller KV pool) and lengthens the CUDA-graph capture list.

`max_num_batched_tokens` is the interesting one, because **it is one dial with two ends**:

```
  THE max_num_batched_tokens TRADE — ONE ITERATION'S BUDGET
  ══════════════════════════════════════════════════════════════════════════════
  Scenario: 40 resident decode sequences + one arriving 8,000-token prompt.
  The scheduler serves RUNNING requests first, then spends what is left on prefill.

  ── budget = 2048 ────────────────────────────────────────────────────────────
   iter n   [40 decode tokens][2008 prefill tokens ......................]
   iter n+1 [40 decode tokens][2008 prefill tokens ......................]
   iter n+2 [40 decode tokens][2008 prefill tokens ......................]
   iter n+3 [40 decode tokens][1976 prefill][ spare ]        prompt done
            └── each iteration ≈ 12 ms ──┘
   ITL for the 40 streaming users : ~12 ms   ← smooth
   TTFT for the arriving prompt   : ~48 ms of prefill + queue

  ── budget = 16384 ───────────────────────────────────────────────────────────
   iter n   [40 decode tokens][8000 prefill tokens .........................]
            └────────────── one iteration ≈ 38 ms ─────────────────────────┘
   iter n+1 [40 decode tokens]
   ITL for the 40 streaming users : ~38 ms on iter n  ← a visible 3× STALL
                                     ~5 ms after
   TTFT for the arriving prompt   : ~38 ms            ← better

  ┌────────────────────────────────────────────────────────────────────────┐
  │ SMALLER budget  → better ITL / TPOT, worse TTFT.                       │
  │ LARGER  budget  → better TTFT, worse ITL / TPOT.                       │
  │ It is ONE dial. "Tune for throughput" without naming which end you     │
  │ are optimising is how a throughput win ships as a smoothness           │
  │ regression that nobody attributes to this flag.                        │
  └────────────────────────────────────────────────────────────────────────┘
```

vLLM's own optimization guide states the two ends directly: smaller values (e.g. 2048)
achieve better ITL because fewer prefill tokens crowd out decode steps; higher values
achieve better TTFT; and for pure throughput it recommends > 8192, "especially for smaller
models on large GPUs."

**Observe which budget is binding** with `vllm:iteration_tokens_total`, a histogram of
tokens per engine step with buckets `[1, 8, 16, …, 16384]`. A pile-up in the bucket at your
configured budget means iterations are saturating it and prefill is being deferred; a
distribution clustered at small values means the budget is not the constraint and raising it
will change nothing.

### 6. `--block-size` and `--enable-prefix-caching`

**`--block-size`** — default **16** (`CacheConfig.DEFAULT_BLOCK_SIZE`). It is the number of
tokens of KV, for every layer, held in one physical block. It sets:

- **rounding waste**, bounded at `block_size − 1` tokens per sequence, forever;
- **prefix-cache granularity**, because only *full* blocks are hashed and cached, so a
  shared prefix of length P yields at most `floor(P / block_size) × block_size` tokens of
  hit;
- **block-table length** per sequence, `ceil(len / block_size)`, which the attention
  kernel's gather walks.

Leave it at 16. It moves your concurrency cap by fractions of a percent while
`--max-model-len` and `--kv-cache-dtype` move it by 2–8× each. The one real reason to raise
it is very long contexts where block-table length itself becomes a cost, and you should
re-measure prefix-cache hit rate when you do, because you have just coarsened it. Current
vLLM also exposes `prefix_match_unit` (`hash_block_size`), which decouples *matching*
granularity from physical block size for hybrid models — you can hash every 32 tokens
inside a 1024-token physical block, as long as every group's `block_size` is divisible by
it.

**`--enable-prefix-caching`** — `enable_prefix_caching` defaults to **`True`**. Any runbook
that tells you to turn it on is pre-V1. What you may need to do instead is turn it *off*
(`--no-enable-prefix-caching`) — for a benchmark that must not benefit from cross-request
reuse, or for a workload with no shared prefixes at all where the hashing is pure overhead.
That overhead is small (V1's design made it cheap enough to default on) but it is not zero.

Related knobs worth knowing exist:

| Flag / field | Default | What it does |
|---|---|---|
| `--prefix-caching-hash-algo` | `sha256` | `sha256_cbor` for cross-language reproducibility; `xxhash` is faster but not cryptographic — the docstring warns about collisions in multi-tenant settings. |
| `POST /reset_prefix_cache` | — | Clears the cache between benchmark runs. `vllm bench sweep serve` calls the `/reset_*_cache` endpoints between runs for exactly this reason. |
| `vllm:prefix_cache_queries_total` / `vllm:prefix_cache_hits_total` | — | Counted in **tokens**, not requests. Hit rate = hits ÷ queries, computed in PromQL. |

A note that matters for benchmarking honesty: with prefix caching on by default, a sweep
that reuses prompts measures your cache, not your engine. Reset between runs or accept that
your CPM curve is optimistic.

### 7. Preemption: the mechanism, and the correction to this lesson's concept line

The concept blockquote at the top says the scheduler preempts by "recompute or swap." That
described the V0 engine. **The V1 engine has only recompute.** vLLM's own metrics design
notes list `vllm:num_requests_swapped` and `vllm:cpu_cache_usage_perc` as legacy metrics for
a mode "no longer relevant in v1," and `--swap-space` was removed with V0. The optimization
guide states it plainly: *"In vLLM V1, the default preemption mode is RECOMPUTE rather than
SWAP, as recomputation has lower overhead in the V1 architecture."* If a tutorial has you
tuning `--swap-space` or choosing between "preemption modes," it predates the engine you are
running. The blockquote is preserved as originally written so this correction stays visible.

Here is the actual code path, from `vllm/v1/core/sched/scheduler.py` (v0.27.1). When a
running request needs blocks and `allocate_slots` returns `None`:

```python
while True:
    new_blocks = self.kv_cache_manager.allocate_slots(
        request, num_new_tokens, num_lookahead_tokens=self.num_lookahead_tokens)
    if new_blocks is not None:
        break                                   # allocated; proceed
    if self.policy == SchedulingPolicy.PRIORITY:
        preempted_req = max(self.running,
                            key=lambda r: (r.priority, r.arrival_time))
        self.running.remove(preempted_req)
    else:                                       # FCFS — the default
        preempted_req = self.running.pop()      # the LAST admitted request
    self._preempt_request(preempted_req, scheduled_timestamp, ...)
    if preempted_req == request:
        break                                   # nothing left to evict
```

and `_preempt_request` does:

```python
self._free_request_blocks(request)      # ALL its blocks return to the pool
self.encoder_cache_manager.free(request)
request.status = RequestStatus.PREEMPTED
request.num_computed_tokens = 0         # ← every computed token is discarded
request.num_preemptions += 1
self.waiting.prepend_request(request)   # goes to the FRONT of the waiting queue
```

Four facts to carry out of those twelve lines:

1. **Victim selection is LIFO under the default FCFS policy.** `self.running.pop()` takes
   the most recently admitted request. That is the standard anti-convoy choice: it protects
   the progress of the oldest requests, so a preemption storm does not turn into a situation
   where nobody finishes. Under `--scheduling-policy priority` the victim is instead the
   lowest-priority, latest-arriving request.
2. **Preemption is recomputation, not suspension.** `num_computed_tokens = 0` means the
   victim re-runs its entire prefill when resumed. A preempted request with a 16k prompt
   pays a second 16k prefill. This is why preemption hits *latency* far harder than the
   eviction itself costs.
3. **The victim goes to the front of the queue**, not the back — so it is not starved, and
   the same request being preempted repeatedly is possible and is the pathological case.
   `request.num_preemptions` counts it per request.
4. **Prefix caching materially softens the recompute.** The resumed request's fresh prefill
   goes through the normal `get_computed_blocks` lookup, and the blocks it just freed are
   still hashed in the pool unless they have been evicted. With APC on by default, a
   preempted request often recovers most of its prefill from cache. This is an
   under-appreciated second-order benefit of default-on prefix caching, and it is why
   preemption in current vLLM is less catastrophic than 2023 write-ups suggest.

**Two dials exist specifically to prevent preemption rather than to survive it**
(`vllm/config/scheduler.py`):

- **`scheduler_reserve_full_isl`, default `True`.** The scheduler verifies that a request's
  **full input sequence length** fits in the KV cache before admitting it, rather than
  admitting on the first prefill chunk and hoping. The docstring names the goal: "Prevents
  over-admission and KV cache thrashing with chunked prefill." Operationally this means
  **you will see requests waiting with the pool at 0.85 and that is correct behaviour**, not
  a bug. The cost asymmetry justifies it: waiting costs a request its queue time, while
  admit-then-preempt costs a full prefill recomputation *plus* the queue time.
- **`watermark`, default `0.0`** (disabled). A fraction of total blocks to keep free when
  admitting waiting or preempted requests. Set it to 0.02–0.05 when you see repeated
  preemption of the same requests: it trades a little admission aggressiveness for stability,
  which is the right trade when the alternative is thrashing.

**The causal picture — why a preemption storm is self-sustaining:**

```
  PREEMPTION CASCADE — why the storm feeds itself
  ══════════════════════════════════════════════════════════════════════════════

  t0   arrival rate rises; resident sequences keep growing (1 block / 16 tokens)
       kv_cache_usage_perc:  0.88 ─▶ 0.97 ─▶ 1.00
                                                │
  t1   a running request needs one more block; pool has none
       │                                        ▼
       └──▶ PREEMPT the last-admitted running request
              • its blocks freed          (pool pressure relieved, briefly)
              • num_computed_tokens = 0   (its prefill work is now WASTED)
              • pushed to FRONT of waiting queue
                                                │
  t2   next iteration: the preempted request is at the front, so it is
       re-admitted early and RE-RUNS ITS WHOLE PREFILL
       │                                        ▼
       └──▶ prefill tokens consume max_num_batched_tokens budget
              • decode steps for everyone else get fewer tokens per iteration
              • ITL rises for ALL streaming users
              • the re-prefill re-allocates the SAME blocks it just freed
                                                │
  t3   pool is full again, one iteration later ──┘  ← LOOP

  WHAT THE THREE SIGNALS DO, TOGETHER:
    kv_cache_usage_perc      ▔▔▔▔▔╲__╱▔▔▔▔  pinned ~1.0, sawtoothing
    num_requests_running     ▔▔▔╲▁▁╱▔▔╲▁▁╱  oscillating, BELOW max_num_seqs
    num_requests_waiting     ▁▁▁▁▁▁▁▁▁▁▁▁▔  monotonically climbing
    num_preemptions_total    ▁▁▁▁▁▁▁▁▁▁▁▁▔  rate > 0 and rising
    GPU utilisation (DCGM)   ▔▔▔▔▔▔▔▔▔▔▔▔▔  ~100 % THROUGHOUT — the util lie:
                                            the GPU is busy doing work it is
                                            about to throw away.

  THE FIX IS ON THE DEMAND SIDE:
    lower --max-num-seqs  ·  lower --max-model-len  ·  --kv-cache-dtype fp8
    ·  set a watermark  ·  add replicas
  NOT --gpu-memory-utilization: +0.03 buys ~4 % pool against a demand
  overshoot that is usually much larger, and spends your OOM margin.
```

**Burst versus trend.** A single preemption event means almost nothing — several long
requests happened to need a block in the same step. Read the *slope* of
`rate(vllm:num_preemptions_total[5m])` against a flat or gently-varying
`vllm:num_requests_running`:

- **Isolated steps under a traffic burst** — the graceful-degradation valve working. Do not
  tune against one event.
- **A sustained non-zero rate at steady-state load** — you are oversubscribed. Your
  `max-num-seqs` × typical-context product promises more resident KV than the pool holds at
  your actual traffic mix. This is a capacity decision, not a transient.

### 8. `--tensor-parallel-size` and the multi-GPU decision

**What TP physically does.** It shards every layer's weight matrices across N GPUs —
column-parallel for the QKV and MLP up-projections, row-parallel for the output and down
projections — and shards the attention heads. Each GPU computes a partial result and the
group performs an **all-reduce per transformer block** (in practice two per layer: one after
attention, one after the MLP) to reconstitute the activation. `tensor_parallel_size`
defaults to **1**.

What that buys and costs, quantitatively:

```
  weights per GPU     ≈ total_weights / N          (linear saving)
  KV per GPU          ≈ total_KV / N               (KV heads shard with TP,
                                                    when n_kv is divisible by N)
  ⇒ KV POOL PER GPU grows SUPERLINEARLY, because weights are a fixed subtraction:
       pool_1 = 0.92·M − W − A
       pool_N = 0.92·M − W/N − A            per GPU, N of them
    Example, 70B bf16 (140 GB) on H100-80GB, util 0.92, A ≈ 3 GB:
       TP=1 : 72.8 − 140 − 3  →  DOES NOT FIT AT ALL
       TP=2 : 72.8 −  70 − 3  =  −0.2 GB   →  still does not fit
       TP=4 : 72.8 −  35 − 3  =  34.8 GB per GPU × 4 = 139 GB of KV
       TP=8 : 72.8 − 17.5− 3  =  52.3 GB per GPU × 8 = 418 GB of KV
    ⇒ For a 70B at bf16 on 80 GB cards, TP=4 is the MINIMUM that fits,
      and the jump 4→8 nearly triples aggregate KV, not doubles it.

  latency cost        + 2 all-reduces per layer per forward pass
                      NVLink (H100 node): ~450 GB/s per direction, sub-100 µs
                        per collective for decode-sized activations
                      PCIe-only (no NVLink): the collectives dominate; TP
                        efficiency collapses. CHECK YOUR TOPOLOGY.
  scaling             sublinear. 2× GPUs does NOT give 2× throughput.
```

**Choosing between TP, PP and replicas.**

| | Tensor parallel (TP) | Pipeline parallel (PP) | Independent replicas |
|---|---|---|---|
| Splits | every layer, across GPUs | contiguous layer ranges | nothing — full copy each |
| Communication | all-reduce **every layer** | point-to-point at stage boundaries | none |
| Needs | fast intra-node fabric (NVLink) | tolerates slower / cross-node links | anything |
| Helps single-request latency | **yes** | no (adds pipeline bubbles) | no |
| Throughput scaling | sublinear | sublinear, bubble-limited | ~linear |
| Failure domain | whole group dies together | whole pipeline dies together | independent |
| Use when | model does not fit on one GPU, **or** you are latency-bound | you must cross a node boundary TP cannot | model fits and you are throughput-bound |

The decision rule, in order:

1. **Does the model + a usable KV pool fit on one GPU?** If yes and you are
   throughput-bound, **use replicas.** They avoid the collective tax entirely and scale
   roughly linearly. TP=2 on a model that fits on one card is a pure loss.
2. **If it does not fit**, choose the **smallest TP that leaves a workable KV pool** —
   compute the residual as above rather than picking a power of two. Keep TP within an
   NVLink domain; `tensor_parallel_size` should not exceed GPUs per NVLink node.
3. **If it still does not fit within one node**, add PP across nodes and keep TP within
   them: TP=8 within node × PP=2 across nodes for a very large model.
4. **If you are latency-bound on a single request** (a low-concurrency, tight-TTFT product),
   TP is the only one of the three that helps, because it puts more compute and more
   bandwidth behind one token.

Related flags you will meet: `--pipeline-parallel-size` (default 1),
`--data-parallel-size` (replicate the whole engine and load-balance internally),
`--enable-expert-parallel` (shard MoE experts instead of tensor-sharding them, at the same
degree as TP), and `--distributed-executor-backend` (`mp` for single node, `ray` for
multi-node). One operational note from the docs that bites: with TP > 1, **each worker reads
the whole checkpoint and slices it**, so weight-load time scales with TP unless you convert
to a sharded checkpoint (`--load-format sharded_state` or `runai_streamer_sharded`) — see
07.9.

### 9. CUDA graphs, compilation, and `--enforce-eager`

At decode-sized batches the per-kernel launch overhead is a real fraction of iteration time,
so vLLM captures CUDA graphs: a recorded sequence of kernel launches replayed as one
submission. The capture list is generated automatically
(`vllm/config/compilation.py`):

```
  cudagraph_capture_sizes = [1, 2, 4] + list(range(8, 256, 8))
                                      + list(range(256, max_cudagraph_capture_size + 1, 16))
  max_cudagraph_capture_size default: 512   (1024 on data-centre Blackwell)
  ⇒ 3 + 31 + 17 = 51 captured graphs by default
```

Each captured graph costs memory (the `Graph capturing finished in N secs, took X GiB`
line) and startup time. Three levers:

- **`-O0` … `-O3`** — optimization levels trading startup time for steady-state performance.
  `-O0` no optimizations, fastest boot; `-O1` simple compilation + fusions + PIECEWISE
  cudagraphs; **`-O2` is the default** (more compile ranges, more fusions,
  FULL_AND_PIECEWISE cudagraphs); `-O3` currently equals `-O2`.
- **`--cuda-graph-sizes` / `compilation_config.cudagraph_capture_sizes`** — cut the list
  (e.g. `[1, 2, 4, 8, 16]`) to reclaim capture memory and startup seconds when you know your
  batch distribution is concentrated.
- **`--enforce-eager`** — skips compilation and capture entirely. Fastest possible startup,
  worst steady-state decode. **It changes nothing about the scheduler**, contrary to a
  persistent piece of folklore; it only disables graph capture. Use it to isolate how much
  of a cold start is compile+capture (07.9), and in dev loops. Do not ship it.

The compile cache is the other half. vLLM persists `torch.compile` artifacts under
`VLLM_CACHE_ROOT` (default `~/.cache/vllm`); the directory can be copied between machines or
baked into a container image, and `VLLM_FORCE_AOT_LOAD=1` makes a cache miss fail loudly
instead of silently recompiling. Any change to the model, config, relevant `VLLM_*`
variables, torch build, or GPU model invalidates it. This is a 30–120 second swing on every
cold start, which is why 07.9 treats it as a first-class cold-start component rather than a
detail.

### 10. CPU provisioning, and why it shows up as GPU idle

An easy-to-miss production failure: **underprovisioned CPU degrades GPU throughput**, and the
symptom looks like a GPU problem.

vLLM V1's process architecture requires, at minimum, `2 + N` processes for N GPUs: one API
server (HTTP, tokenisation, detokenisation, streaming), one engine core (the scheduler, in a
busy loop), and one worker per GPU. The optimization guide is explicit that these must be
**physical cores**, not vCPUs — with hyperthreading, 1 vCPU is half a physical core, so the
floor doubles in vCPU terms. With data parallelism the formula becomes
`A + DP + N + (1 if DP > 1 else 0)`, where `A` is `--api-server-count` (defaults to DP).

The engine core is the sensitive one: it runs a busy loop, and if it is descheduled the GPU
sits idle between iterations. The failure signature is *low* GPU utilisation with *high*
queue depth and no memory pressure — which reads exactly like a scheduling bug and is
actually a `resources.requests.cpu` that someone set to `4` on an 8-GPU node.

**`--api-server-count N`** runs N API server processes in front of the engine, for when
tokenisation and input processing (all CPU work, all in P0) become the bottleneck. Two
caveats from the docs: it disables multi-modal IPC caching (which needs a 1:1 API-to-engine
mapping), and it is **incompatible with `VLLM_ALLOW_RUNTIME_LORA_UPDATING`** — the CLI
rejects the combination outright (07.10 §Pitfalls).

### 11. A production launch, annotated

Everything above, as one command you could ship, for an 8B model on one H100 with a
500 ms TTFT-p99 target:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --served-model-name chat-8b \
  --port 8000 \
  \
  `# ── memory ──────────────────────────────────────────────────` \
  --gpu-memory-utilization 0.92 \
  `#   fraction of TOTAL device memory. 0.92 is the v0.27 default.` \
  `#   Pin --kv-cache-memory-bytes instead once profiled, for fleet` \
  `#   determinism across nodes with different co-tenancy.` \
  --max-model-len 8192 \
  `#   set from measured p99(prompt+generation), NOT the model card.` \
  `#   Divides printed concurrency; allocates nothing.` \
  --kv-cache-dtype fp8 \
  `#   halves KV bytes/token ⇒ ~2x the token capacity of the same` \
  `#   pool. Hopper+. Validate accuracy (07.7) before shipping.` \
  \
  `# ── scheduler ───────────────────────────────────────────────` \
  --max-num-seqs 256 \
  `#   default would be 1024 on this card. 256 caps resident demand` \
  `#   so the pool is not oversubscribed at 8k context.` \
  --max-num-batched-tokens 4096 \
  `#   default 8192 here. Halved to protect ITL: fewer prefill` \
  `#   tokens per iteration crowding out decode steps.` \
  --block-size 16 \
  `#   the default. Do not change it first.` \
  \
  `# ── engine ──────────────────────────────────────────────────` \
  --tensor-parallel-size 1 \
  `#   8B fits with room. TP>1 here would be a pure all-reduce tax.` \
  --api-server-count 2 \
  `#   tokenisation is CPU-bound and lives in P0. Needs cores.` \
  --disable-log-requests
```

`--enable-prefix-caching` is absent because it is **already on**. `--swap-space` is absent
because it **no longer exists**.

The same thing as a Kubernetes Deployment, with the fields that actually matter for this
lesson annotated:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-chat-8b
spec:
  replicas: 2
  selector:
    matchLabels: { app: vllm-chat-8b }
  template:
    metadata:
      labels: { app: vllm-chat-8b }
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:v0.27.1        # PIN IT. Defaults move.
          args:
            - --model=meta-llama/Llama-3.1-8B-Instruct
            - --served-model-name=chat-8b
            - --gpu-memory-utilization=0.92
            - --max-model-len=8192
            - --kv-cache-dtype=fp8
            - --max-num-seqs=256
            - --max-num-batched-tokens=4096
            - --api-server-count=2
          ports:
            - { name: http, containerPort: 8000 }
          resources:
            limits:
              nvidia.com/gpu: 1
              cpu: "8"          # floor is 2 + N = 3 PHYSICAL cores; give slack.
              memory: 48Gi      # host RAM for weights staging + tokeniser pools.
            requests:
              nvidia.com/gpu: 1
              cpu: "8"
              memory: 48Gi
          env:
            - { name: VLLM_CACHE_ROOT, value: /cache/vllm }   # persist compile cache
            - { name: HF_HOME,         value: /cache/hf }     # persist weights
          volumeMounts:
            - { name: model-cache, mountPath: /cache }
            - { name: shm,         mountPath: /dev/shm }
          readinessProbe:
            httpGet: { path: /health, port: http }
            initialDelaySeconds: 90     # MEASURE THIS (07.9). Stage 4 is slow.
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet: { path: /health, port: http }
            periodSeconds: 10
            failureThreshold: 60        # 10 minutes for a cold weight pull.
          livenessProbe:
            httpGet: { path: /health, port: http }
            initialDelaySeconds: 600    # never restart a still-loading pod.
            periodSeconds: 20
      volumes:
        - name: model-cache
          persistentVolumeClaim: { claimName: vllm-model-cache }
        - name: shm
          emptyDir: { medium: Memory, sizeLimit: 16Gi }   # IPC between P0/P1/W*
      terminationGracePeriodSeconds: 120   # let in-flight generations drain.
```

Three lines there are load-bearing and routinely wrong:

- **`startupProbe` with a large `failureThreshold`.** Without it, a cold weight pull trips
  the liveness probe and Kubernetes restarts the pod mid-load, forever.
- **`/dev/shm`.** The V1 processes communicate through shared memory; the container default
  of 64 MB is not enough and produces obscure IPC failures under load.
- **`terminationGracePeriodSeconds`.** A 30-second default kills in-flight generations. Set
  it above your p99 end-to-end latency.

### 12. The tuning procedure, in order

These knobs interact, so change one at a time and re-measure. This order goes from the
constraints you cannot negotiate to the ones you can.

```
  1. FIX max-model-len from measured traffic.
       p99(prompt) + p99(generation), ×1.3 margin, rounded up.
       Everything downstream is denominated in this number.

  2. FIX the parallelism that makes the model fit.
       Compute the residual: 0.92·M − W/N − A. Smallest N with a workable
       pool. If N=1 works and you are throughput-bound, use replicas.

  3. BOOT and read the log. Verify:
       "Model loading took ..."          weights are what you expect
       "Available KV cache memory: ..."  the pool is what you predicted
       "GPU KV cache size: N tokens"     capacity
       "Maximum concurrency: Kx"         = N / max-model-len, sanity check
       If any is surprising, stop. Do not tune past a wrong number.

  4. SATURATE with your real length distribution and find the binding cap.
       vllm bench serve --max-concurrency <high>, sample /metrics:
         kv_cache_usage_perc ≈ 1.0, running << max_num_seqs  → MEMORY binds
         kv_cache_usage_perc low, running == max_num_seqs    → FLAG binds
         iteration_tokens_total piled at budget              → TOKEN BUDGET binds

  5. SET max-num-seqs to just below the measured memory-bound cap.
       Goal: preemption rate ≈ 0 at your p95 offered load. This is the
       flag that actually prevents preemption, not utilisation.

  6. TUNE max-num-batched-tokens against your SLO, LAST.
       ITL over budget → lower it.   TTFT over budget → raise it.
       Re-check preemption after each change (bigger prefill chunks
       allocate blocks faster).

  7. ONLY NOW consider --gpu-memory-utilization above the default, and
     only with a real OOM test at 1.5× your peak prompt length.
```

---

## Perspectives

**The allocator's-eye view.** Every flag in §2–§7 is either sizing the pool or bounding
demand against it. `gpu-memory-utilization` and `kv-cache-memory-bytes` size supply;
`max-model-len`, `max-num-seqs` and `max-num-batched-tokens` bound demand; preemption is what
happens when demand wins. Holding that supply/demand frame is what stops the single most
common tuning error, which is reaching for a supply knob (utilisation) to fix a demand
problem (too many resident sequences). Supply is bounded by physics and is nearly exhausted
at the default; demand is bounded by flags you own.

**The SRE / on-call view.** At 3am, the signal is not "the GPU is slow." It is
`rate(vllm:num_preemptions_total[5m]) > 0` together with `vllm:kv_cache_usage_perc` pinned
near 1.0 and `vllm:num_requests_running` oscillating *below* `max_num_seqs`. That
conjunction is diagnostic; any one of the three alone is not. The first lever is
`--max-num-seqs`, the second is `--max-model-len`, and the one that looks tempting and is
wrong is `--gpu-memory-utilization`. Note also that DCGM GPU utilisation reads ~100 %
throughout a preemption storm, which is the util lie from module 05 in its most expensive
form: the GPU really is busy, doing work it is about to discard.

**The fleet-operator view.** Profiling-based KV sizing is adaptive, which means two nodes
with different co-tenancy get different concurrency caps from the same manifest — and your
load balancer, which assumes replicas are interchangeable, will quietly overload the smaller
one. `--kv-cache-memory-bytes` exists for exactly this: it trades adaptivity for
determinism. The same instinct — pin the thing that would otherwise vary per node — applies
to the container tag, the compile cache, and the driver version. Reproducibility is a
serving property, not just a build property.

**The hardware-topology view.** The TP-versus-replica crossover is not a fixed rule; it moves
with the interconnect. On an NVLink-connected H100 node the per-layer all-reduce is a
sub-100 µs cost that TP's memory saving easily repays. On a node where the GPUs talk over
PCIe, the same collective can dominate the step and TP efficiency collapses. Before applying
any TP guidance on unfamiliar hardware, run `nvidia-smi topo -m` and check whether the pairs
you plan to group say `NV#` or `PIX`/`SYS`. The rules in §8 assume the former.

**The economics view.** The chain from flags to dollars is short. `max-num-seqs` and the KV
pool set the resident batch; resident batch sets decode throughput while below the ridge
point; decode throughput is the denominator of cost per million tokens. A default-tuned 8B
endpoint left at `--gpu-memory-utilization 0.60` and `--max-model-len 131072` can be running
at a fifth of its achievable throughput on identical hardware — a 5× cost-per-token penalty
delivered entirely by two numbers in a YAML file, with no error, no alert, and no
degradation anyone would notice except in the bill. That is why this lesson precedes the
cost curve rather than following it.

## Real-world use cases

- **vLLM PR #17885 and the A100 batch-token default.** The source of `get_batch_defaults`
  carries the comment *"Setting large `max_num_batched_tokens` for A100 reduces throughput,
  see PR #17885 for more details,"* and the code performs a literal device-name check for
  `"a100"` to route A100s to the 2048/256 defaults instead of the 8192/1024 ones used for
  other ≥70 GiB cards. **What it shows:** a "tuned config" is hardware-specific in ways that
  are not derivable from memory capacity. Two 80 GB cards from the same vendor get 4×
  different budgets. Copying a config across GPU generations without re-measuring is how you
  ship a regression you cannot explain.

- **The v0.21.0 CUDA-graph memory accounting change.** Before v0.21.0, CUDA-graph memory was
  not subtracted when sizing the KV pool; it came silently out of the safety margin. Since
  v0.21.0 it is profiled and subtracted, and the engine prints the exact equivalence — *"the
  current --gpu-memory-utilization=0.92 is equivalent to 0.9082 without CUDA graph memory
  profiling; to maintain the same effective KV cache size, increase to 0.9318."* **What it
  shows:** the meaning of a config value can change between minor releases without the value
  changing. It also shows what good engine ergonomics look like — rather than silently
  shrinking your pool, the engine tells you the number to move to. Pin your version and read
  your startup log after every upgrade.

- **The V0 → V1 removal of swap preemption.** V0 could copy a preempted sequence's KV to CPU
  memory over PCIe and copy it back. V1 removed the path, removed `--swap-space`, and marked
  `vllm:num_requests_swapped` and `vllm:cpu_cache_usage_perc` as legacy metrics "no longer
  relevant in v1." **What it shows:** the ecosystem's documentation half-life is short and
  asymmetric — removed features linger in blog posts far longer than they linger in code.
  The operational tell is any content that presents "recompute vs swap" as a live choice; it
  is describing an engine you are not running. This lesson's own concept blockquote is an
  instance, preserved deliberately so the correction is visible.

- **vLLM's CPU-provisioning guidance for GPU deployments.** The optimization docs state a
  `2 + N` **physical**-core floor and warn that the engine-core busy loop "is particularly
  sensitive to CPU starvation," with the observable symptom being GPU utilisation lower than
  expected. **What it shows:** the bottleneck in a modern inference engine is frequently the
  control plane's CPU, not the GPU — a genuinely counter-intuitive property of a system whose
  entire purpose is matrix multiplication, and one that turns "our GPUs are only 60 % busy"
  from a scheduling mystery into a `resources.requests.cpu` line.

## Worked example

**Take an untuned 8B deployment into a preemption storm on purpose, diagnose it from
metrics alone, and fix it with the cheapest correct lever.** One H100-80GB, roughly 40
minutes of GPU time. Numbers below are representative of this configuration; your exact
figures will differ with driver, dtype and version, and the point is the *shape*, not the
digits.

### Step 1 — the deliberately naive config

The config a reasonable person writes on day one: model card context, defaults everywhere.

```bash
pip install "vllm==0.27.1" && vllm --version

vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 131072 \
  --port 8000 2>&1 | tee naive.log
```

Startup log, read line by line:

```
INFO ... Model loading took 14.99 GiB memory and 11.42 seconds
INFO ... Memory profiling takes 6.42 seconds. Total non KV cache memory: 17.73GiB;
         torch peak memory increase: 1.87GiB; weights memory: 14.99GiB.
INFO ... Available KV cache memory: 55.05 GiB
INFO ... GPU KV cache size: 450,816 tokens
INFO ... Maximum concurrency for 131,072 tokens per request: 3.44x
INFO ... Graph capturing finished in 23 secs, took 0.96 GiB
INFO ... init engine (profile, create kv cache, warmup model) took 34.71 s
```

- `17.73 GiB` non-KV is higher than the tuned run below, because the default
  `max_num_batched_tokens` of 8192 on this card shapes a larger profiling dummy batch.
- **`Maximum concurrency: 3.44x`** is the alarm. The engine is planning for a world where
  every request uses 131,072 tokens. `--max-num-seqs` is defaulting to 1024. The scheduler
  is being told it may admit 1024 sequences into a pool that, in the worst case, holds 3.
- `450,816 tokens` is the real capacity, and it does not depend on `max-model-len` at all.

### Step 2 — drive it and watch it fail

```bash
vllm bench serve \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 \
  --dataset-name random --random-input-len 4096 --random-output-len 512 \
  --request-rate inf --max-concurrency 400 --num-prompts 2000 \
  --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
  --save-result --result-filename naive.json
```

Sampling `/metrics` every two seconds during the run:

```bash
while true; do
  curl -s localhost:8000/metrics | grep -E \
    '^vllm:(kv_cache_usage_perc|num_requests_running|num_requests_waiting|num_preemptions_total)' \
    | tr '\n' ' '; echo; sleep 2
done
```

A representative saturated sample:

```
vllm:kv_cache_usage_perc      1.00
vllm:num_requests_running      88
vllm:num_requests_waiting     312
vllm:num_preemptions_total    417        # was 190 sixty seconds earlier
```

And the benchmark summary:

```
============ Serving Benchmark Result ============
Successful requests:                     2000
Maximum request concurrency:             400
Benchmark duration (s):                  312.84
Output token throughput (tok/s):         2104.55
Total token throughput (tok/s):          18893.10
---------------Time to First Token----------------
Median TTFT (ms):                        4185.22
P99 TTFT (ms):                           19402.77
-----Time per Output Token (excl. 1st token)------
Median TPOT (ms):                        61.80
P99 TPOT (ms):                           388.41
```

**Diagnose before touching anything.** Run the three-signal test:

1. `kv_cache_usage_perc` = 1.00 → blocks exhausted. ✓
2. `num_requests_running` = 88, well below `max_num_seqs` = 1024 → **memory binds, not the
   flag.** ✓
3. `num_preemptions_total` rising at ≈ 3.8/s → admitted work is being discarded. ✓

All three agree: genuinely KV-bound, with active thrashing. The P99 TPOT of 388 ms against a
median of 62 ms is the preemption signature — a subset of requests stalled and re-prefilled
while the rest streamed normally. Note also that DCGM GPU utilisation reads ~99 % throughout.

Predicted versus measured concurrency, as a sanity check: mean live context is about
`4096 + 512/2 = 4,352` tokens, so `450,816 / 4,352 ≈ 103`. Measured 88 — about 15 % low,
explained by block rounding, the conservative full-input-length admission check, and the
blocks tied up by preempted requests mid-recompute.

### Step 3 — fix it with the demand-side lever

The pool is 450,816 tokens and mean live context is ~4,352, so the sustainable resident
count is roughly 100. Leave headroom for growth and set `--max-num-seqs 64`. Right-size
`--max-model-len` to the workload's actual 4,608-token maximum, rounded to 8192. Halve
`--max-num-batched-tokens` to protect ITL. Change nothing else — in particular, **do not**
touch `--gpu-memory-utilization`.

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 \
  --max-num-seqs 64 \
  --max-num-batched-tokens 4096 \
  --port 8000 2>&1 | tee tuned.log
```

```
INFO ... Memory profiling takes 5.91 seconds. Total non KV cache memory: 16.51GiB;
         torch peak memory increase: 1.20GiB; weights memory: 14.99GiB.
INFO ... Available KV cache memory: 56.27 GiB
INFO ... GPU KV cache size: 460,800 tokens
INFO ... Maximum concurrency for 8,192 tokens per request: 56.25x
```

Two things moved and one did not. The non-KV term fell from 17.73 to 16.51 GiB — the smaller
`max_num_batched_tokens` shaped a smaller profiling dummy batch, so the pool grew by
~1.2 GiB **for free**. `Maximum concurrency` went from 3.44x to 56.25x, entirely because the
denominator changed. And `gpu-memory-utilization` was never touched.

### Step 4 — re-run the identical benchmark

```
============ Serving Benchmark Result ============
Successful requests:                     2000
Maximum request concurrency:             400
Benchmark duration (s):                  241.09
Output token throughput (tok/s):         2731.63
Total token throughput (tok/s):          24519.44
---------------Time to First Token----------------
Median TTFT (ms):                        2874.51
P99 TTFT (ms):                           6612.08
-----Time per Output Token (excl. 1st token)------
Median TPOT (ms):                        22.44
P99 TPOT (ms):                           38.90
```

With `/metrics` at saturation:

```
vllm:kv_cache_usage_perc      0.61
vllm:num_requests_running      64        # ← exactly max_num_seqs. The FLAG now binds.
vllm:num_requests_waiting     336
vllm:num_preemptions_total      0        # ← flat for the whole run
```

Read the result honestly:

| | Naive | Tuned | Change |
|---|---|---|---|
| Output tok/s | 2,104 | 2,731 | **+30 %** |
| P99 TTFT | 19,403 ms | 6,612 ms | **−66 %** |
| P99 TPOT | 388 ms | 38.9 ms | **−90 %** |
| Preemptions | 417 and climbing | 0 | eliminated |
| CPM @ $2.89/hr, util 1.0 | $0.381 | $0.294 | **−23 %** |

`CPM = 2.89 / (tok/s × 3600) × 1e6`, so `2.89 / (2104 × 3600) × 1e6 = $0.381` and
`2.89 / (2731 × 3600) × 1e6 = $0.294`. **A 23 % cost reduction and a 10× tail-latency
improvement from three flags, on identical hardware, with no quantisation and no extra
GPUs.**

### Step 5 — find the actual optimum, and stop

`num_requests_running` is now pinned exactly at 64 with the pool at 0.61 — the flag binds
and there is headroom. So walk `--max-num-seqs` up and watch for the return of preemption:

| `--max-num-seqs` | Output tok/s | P99 TTFT | P99 TPOT | KV usage | Preemptions |
|---|---|---|---|---|---|
| 64 | 2,731 | 6,612 ms | 38.9 ms | 0.61 | 0 |
| 96 | 3,088 | 6,940 ms | 51.2 ms | 0.88 | 0 |
| 112 | 3,144 | 7,455 ms | 78.6 ms | 0.97 | 6 |
| 128 | 3,061 | 9,120 ms | 165.3 ms | 1.00 | 143 |

**Throughput peaks at 112 and then falls.** That is the signature to internalise: past the
point where preemption starts, more admitted sequences produce *less* completed work,
because a growing share of the GPU's time goes into re-prefilling requests it already
prefilled. The operating point is 96 — the largest value with zero preemptions and headroom
for a traffic burst — not 112, which is inside the cliff.

The `--max-num-seqs 96` config is what carries forward into 07.5's CPM sweep. Sweeping
offered load against an oversubscribed config would have produced a curve whose knee was an
artefact of preemption rather than a property of the hardware.

## Practice

Rented GPU, roughly 60–75 minutes. Pin the version (`vllm==0.27.1` or later) and record it —
every default in this lesson is version-sensitive.

### 1. Read your own startup profile

Boot the naive config from the worked example and extract the numbers:

```bash
pip install "vllm==0.27.1" && vllm --version
vllm serve meta-llama/Llama-3.1-8B-Instruct --max-model-len 131072 \
  --port 8000 2>&1 | tee naive.log

grep -E 'Model loading took|Memory profiling|Available KV cache|GPU KV cache size|Maximum concurrency|Graph capturing|init engine' naive.log
```

**Acceptance:** the seven-line extract, plus a one-sentence statement of which term of the
budget (`weights` / `non-KV` / `pool`) differed most from your prediction and why.

### 2. Prove `--gpu-memory-utilization` multiplies total, not free

Boot at 0.85 and at 0.92 on an otherwise idle card, and record `Available KV cache memory`
for each. Then compute the implied `total_memory` from the two data points and compare it to
`nvidia-smi --query-gpu=memory.total --format=csv`.

**Acceptance:** two pool sizes, the derived total, and the identity
`pool ≈ total × util − non_kv` shown to hold within a few hundred MiB.

### 3. Induce and diagnose a preemption storm

Run the naive config under `vllm bench serve --max-concurrency 400` with `/metrics` sampled
every 2 s. Capture, at saturation: `kv_cache_usage_perc`, `num_requests_running`,
`num_requests_waiting`, and the `num_preemptions_total` *rate*.

**Acceptance:** the four values at one instant, plus an explicit statement of which of the
three ceilings (KV pool / `max_num_seqs` / `max_num_batched_tokens`) was binding, with the
evidence for the claim.

### 4. Fix it and quantify the fix

Apply the tuned config. Re-run the **identical** benchmark command. Produce the
naive-vs-tuned comparison table: output tok/s, P99 TTFT, P99 TPOT, preemption count, and CPM
at your GPU's hourly rate.

**Acceptance:** the table, plus the CPM arithmetic written out with units so it can be
re-run with different inputs.

### 5. Walk `--max-num-seqs` to the cliff

Sweep `--max-num-seqs` over at least four values spanning the preemption onset. Record
output throughput, P99 TPOT, peak `kv_cache_usage_perc` and total preemptions for each.

**Acceptance:** the four-row table with the **throughput maximum and the preemption onset
identified as different points**, and a one-line justification for the operating point you
would ship — which should be the largest preemption-free value, not the throughput peak.

### 6. Establish the `max_num_batched_tokens` trade in your own numbers

At your chosen `--max-num-seqs`, run the sweep at `--max-num-batched-tokens` of 2048, 4096
and 8192, holding everything else fixed.

**Acceptance:** a three-row table of P99 TTFT and P99 ITL showing them move in **opposite**
directions, plus the value you would pick given a 500 ms TTFT-p99 target.

**Overall acceptance:** a tuned, preemption-free launch command with every non-default flag
justified by one of your own measurements, committed to the
[cost-per-token deliverable](../practice/cost-per-token/README.md) as `configs/`. This
config is the baseline for 07.5's CPM sweep — the sweep measures a config, so the config has
to be defensible first.

## Common pitfalls

- **Raising `--gpu-memory-utilization` to fix preemption.** The instinctive move and the
  wrong one. *Mechanism:* going 0.92 → 0.95 on an 80 GB card adds ~2.4 GiB to a ~55 GiB
  pool — about 4 % more concurrency — because weights and activations are a fixed
  subtraction the fraction does not touch. Meanwhile the demand overshoot causing the
  preemption is usually tens of percent, and you have just spent the margin that absorbs
  allocator spikes and prefill bursts. Fix demand (`--max-num-seqs`, `--max-model-len`,
  `--kv-cache-dtype fp8`), not supply.

- **Setting `--max-model-len` to the model's architectural maximum.** *Mechanism:* the
  printed concurrency is `num_gpu_blocks ÷ ceil(max_model_len / block_size)`; the numerator
  does not move. A 128k setting for 8k traffic divides your admission headroom by 16 for a
  capability nobody uses, and with `scheduler_reserve_full_isl` the scheduler is *also*
  being conservative against that inflated worst case. Set it from measured p99, revisit
  when traffic shape drifts.

- **Assuming `--gpu-memory-utilization` is a fraction of *free* memory.** *Mechanism:*
  `requested = ceil(total_memory × util)`, validated against free memory at init. On a card
  with a co-tenant holding 10 GiB, `0.92` fails at startup rather than adapting. For two
  vLLM instances on one card, each needs ≤ 0.5 — not "whatever is left."

- **Copying an H100 config onto an A100.** *Mechanism:* `get_batch_defaults` performs an
  explicit device-name check and routes A100s to `max_num_batched_tokens = 2048` /
  `max_num_seqs = 256`, because large batched-token values measurably reduce A100
  throughput. Same 80 GB, 4× different budget, deliberately. Re-measure per SKU.

- **Tuning `--swap-space`, or reasoning about "recompute vs swap preemption modes."**
  *Mechanism:* the CPU-swap preemption path does not exist in V1; the flag was removed and
  the associated metrics are marked legacy. Any content presenting this as a live choice
  predates your engine. There is one mode: recompute.

- **Using `--enforce-eager` in production to "reduce memory."** *Mechanism:* it disables
  CUDA-graph capture, which does free the ~1 GiB the graphs occupy — while costing you the
  kernel-launch amortisation on every decode step. That is trading a 2 % memory win for a
  double-digit decode-throughput loss. If you need the memory, trim
  `cudagraph_capture_sizes` instead.

- **A `readinessProbe` with a default `initialDelaySeconds`.** *Mechanism:* the engine
  serves `/health` only after stage 4 (graph capture) completes, which is tens of seconds
  even warm and minutes cold. Without a `startupProbe` with a generous `failureThreshold`,
  the liveness probe restarts the pod mid-load, forever, and the symptom looks like a
  crashloop rather than a probe misconfiguration.

- **Underprovisioning CPU and diagnosing it as a GPU problem.** *Mechanism:* the engine core
  runs a busy loop; when it is descheduled the GPU idles between iterations. The floor is
  `2 + N` **physical** cores (double that in vCPUs with hyperthreading). Symptom: low GPU
  utilisation, high queue depth, no memory pressure — which reads as a scheduler bug and is
  a `resources.requests.cpu` value.

- **Benchmarking with prefix caching on and not resetting between runs.** *Mechanism:*
  `enable_prefix_caching` defaults to `True`, and a sweep that replays the same prompts
  measures cache hits, not engine throughput. `POST /reset_prefix_cache` between runs, or
  state clearly that the number includes cache benefit.

## Self-check

**(a) What does `--gpu-memory-utilization 0.92` actually compute, and what happens if
another process is already holding 10 GiB on the card?**

**Answer:** It computes `requested_memory = ceil(total_device_memory × 0.92)` — a fraction of
**total**, not free, memory (`vllm/v1/worker/utils.py: request_memory`). On a card reporting
79.11 GiB total that is 72.78 GiB. The engine then checks `free_memory >= requested_memory`
and, if a co-tenant holds 10 GiB (leaving ~69 GiB free), **raises a `ValueError` at startup**
naming both figures, rather than adapting or OOMing later. The KV pool is then
`requested_memory − non_kv_cache_memory − cudagraph_memory_estimate`, where the non-KV term
is measured by an actual dummy forward pass shaped by `max_num_batched_tokens` and
`max_num_seqs`. Consequences: two vLLM instances sharing a card must each be given ≤ 0.5;
the unrequested tail is space vLLM promises not to use, not headroom for vLLM's own spikes;
and since v0.21.0 CUDA-graph memory is profiled and subtracted, so the same numeric value
yields a slightly smaller pool than it did before that release — the engine prints the exact
equivalent value in a log line.

**(b) You double `--max-model-len`. What happens to memory consumption, to the printed
maximum concurrency, and to a single request's cost?**

**Answer:** Memory consumption does not change **at all** — the pool is
`total×util − weights − working memory`, none of which depends on `max_model_len`, so
`num_gpu_blocks` and `GPU KV cache size: N tokens` are byte-identical before and after.
Printed maximum concurrency **halves**, because it is `num_gpu_blocks ÷ ceil(max_model_len /
block_size)` and you doubled the denominator. A single request's cost is unchanged: under
paged allocation a sequence holds blocks proportional to its *current* length, so raising
the ceiling makes no individual request more expensive. What you actually spent is
**admission headroom** — the worst case the scheduler plans against, and (with
`scheduler_reserve_full_isl = True`) the size it must verify fits before admitting. That is
why setting it to a model's architectural maximum is expensive despite allocating nothing.

**(c) `vllm:num_requests_waiting` is climbing. Describe the test that distinguishes the three
possible causes and name the fix for each.**

**Answer:** Evaluate three signals as a **conjunction**, not a disjunction.
(1) `vllm:kv_cache_usage_perc` ≈ 1.0 **and** (2) `vllm:num_requests_running` flat at a
ceiling *below* `max_num_seqs` **and** (3) `rate(vllm:num_preemptions_total[5m]) > 0` ⇒
**KV-pool-bound**; fix by reducing demand (`--max-num-seqs`, `--max-model-len`,
`--kv-cache-dtype fp8`) or adding capacity. If `kv_cache_usage_perc` is low while
`num_requests_running` sits *exactly* on the configured `max_num_seqs` ⇒ **flag-bound**;
raise `--max-num-seqs`, and more GPU memory buys nothing. If neither holds but
`vllm:iteration_tokens_total` piles up in the bucket at your configured budget, with TTFT
rising and ITL flat ⇒ **token-budget-bound**; raise `--max-num-batched-tokens` (accepting
worse ITL) or shed prefill load. Also check `vllm:num_requests_waiting_by_reason`:
`capacity` is genuine scheduling pressure, `deferred` is a transient constraint (LoRA
budget, KV transfer, blocked status) with a different fix. And rule out the client: a
connection-pool cap produces a growing server-side queue with a low pool and no memory
explanation.

**(d) Exactly what happens to a request when it is preempted in vLLM V1, and why does that
make preemption look like a latency incident rather than a throughput one?**

**Answer:** Under the default FCFS policy the victim is `self.running.pop()` — the **most
recently admitted** running request (LIFO, an anti-convoy choice that protects the oldest
requests' progress). `_preempt_request` then frees **all** of its KV blocks, sets
`status = PREEMPTED`, sets **`num_computed_tokens = 0`**, increments `num_preemptions`, and
**prepends** it to the waiting queue. Because the computed-token counter is zeroed, resuming
means re-running the entire prefill: a preempted 16k-context request pays a second 16k
prefill. That is why the symptom is latency — the victim's TTFT-like and ITL intervals
stretch by its queue time plus a full re-prefill, while other requests' ITL degrades because
the re-prefill consumes the shared `max_num_batched_tokens` budget. Aggregate throughput
falls at the same time, because a growing share of GPU time produces work that was already
produced. GPU utilisation stays near 100 % throughout, which is the util lie at its most
expensive. Prefix caching softens it materially: the re-prefill goes through the normal
cached-block lookup and the victim's own just-freed blocks are usually still hashed. There
is **no swap path in V1** — recompute is the only mode.

**(e) An 8B model fits comfortably on one H100. A colleague proposes `--tensor-parallel-size 2`
to "double throughput." What do you say?**

**Answer:** TP does not double throughput; it splits one model's work across two GPUs and
adds two all-reduce collectives per transformer layer per forward pass. For a model that
already fits, throughput scaling from TP is sublinear and the marginal cost is a second GPU,
so cost per token goes **up**. The correct answer for a throughput-bound workload on a
model that fits is **two independent replicas** behind a load balancer: no collective tax,
roughly linear throughput scaling, and independent failure domains. TP is the right tool in
exactly two cases — (i) the model plus a usable KV pool does not fit on one GPU, where you
choose the smallest N whose residual `0.92·M − W/N − A` leaves a workable pool; and (ii) you
are **latency-bound on a single request**, where TP is the only option of the three that
lowers one request's TTFT and ITL, because replicas do nothing for a single request. One
caveat in both cases: check `nvidia-smi topo -m` first — TP's economics assume an NVLink
domain, and over PCIe the collectives can dominate the step.

**(f) Why do `--max-num-batched-tokens` and `--max-num-seqs` change the size of your KV pool
even though neither one allocates KV?**

**Answer:** Because the non-KV term is **measured, not computed**. During startup stage 2 the
worker runs `profile_run()` — a dummy forward pass shaped by `max_num_batched_tokens` and
`max_num_seqs` — and records the peak activation memory, then estimates CUDA-graph memory.
The KV pool is the remainder: `requested_memory − non_kv_cache_memory −
cudagraph_memory_estimate`. Raising the token budget makes the profiling batch bigger, so the
measured activation peak is bigger, so the remainder is smaller. In the worked example,
halving `max_num_batched_tokens` from 8192 to 4096 dropped the non-KV term from 17.73 to
16.51 GiB and grew the pool by ~1.2 GiB for free. `max_num_seqs` also lengthens the
CUDA-graph capture list (the default list runs to `max_cudagraph_capture_size`, capped at
512), costing both capture memory and startup time. This is the concrete mechanism behind
"the knobs are not independent."

## Connections & what's next

This lesson turned 07.3's mechanism into a config you can defend and a failure mode you can
diagnose. The tuned, preemption-free launch command is now a prerequisite artefact: 07.5
sweeps offered load against **this** config to build the CPM curve, and a curve measured
against an oversubscribed engine has a knee that is an artefact of preemption rather than a
property of the hardware. The three-signal runbook from 07.2, extended here with
`vllm:iteration_tokens_total` and the preemption cascade, is the same diagnostic you will
reuse in 07.8 when choosing an autoscaling signal — because `kv_cache_usage_perc` and
`num_requests_waiting` are what actually track inference saturation, and GPU utilisation, as
§7's cascade shows, reads ~100 % even while the engine destroys its own work. `--kv-cache-dtype
fp8`, used here as a demand-side lever, gets its accuracy treatment in 07.7, and the startup
timeline in §1 is decomposed into a cold-start budget with real storage bandwidths in 07.9.

**Next: [07.5 — Batching economics](05-batching-economics.md)** takes this config, sweeps
concurrency across it, and converts measured output tokens per second into dollars per
million tokens — locating the operating point where cost is minimised subject to your TTFT
SLO, which is the flagship artefact of the module.

## References & further reading

**Primary sources — vLLM engine (read at tag `v0.27.1`, cross-checked against `main` @ `c1e4387`, 2026-08-17)**

1. **`vllm/config/cache.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/cache.py — `DEFAULT_BLOCK_SIZE = 16`; `gpu_memory_utilization` default **0.92** with the "per-instance limit" docstring; `enable_prefix_caching = True`; `prefix_caching_hash_algo` (`sha256` default, `xxhash` with its collision warning); `kv_cache_memory_bytes` and its "ignores gpu_memory_utilization" semantics; the `CacheDType` enum; `prefix_match_unit`. **Correction to earlier versions of this lesson:** the utilisation default is 0.92, not 0.90 — 0.90 was the 0.11.x value.
2. **`vllm/v1/worker/utils.py` — `request_memory()`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/utils.py — the two lines that settle §2: `requested_memory = ceil(total_memory × gpu_memory_utilization)` and the `free_memory < requested_memory` startup check. This is the authority for "fraction of total, not free."
3. **`vllm/v1/worker/gpu_worker.py` — `determine_available_memory()`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_worker.py — the profiling path, the CUDA-graph memory estimate, the `available_kv_cache_memory = requested − non_kv − cudagraph_estimate` subtraction, and the v0.21.0 equivalence log line quoted in §2. Also `sleep()`/`wake_up()`, used in 07.8/07.9.
4. **`vllm/v1/core/sched/scheduler.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py — the `allocate_slots` retry-with-preemption loop, FCFS `self.running.pop()` versus PRIORITY `max(running, key=(priority, arrival_time))`, and `_preempt_request` with `num_computed_tokens = 0` and `waiting.prepend_request`. §7's code excerpts are from here.
5. **`vllm/config/scheduler.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/scheduler.py — `DEFAULT_MAX_NUM_BATCHED_TOKENS = 2048`, `DEFAULT_MAX_NUM_SEQS = 128` (both documented as test-convenience values), `enable_chunked_prefill = True`, `policy = "fcfs"`, `scheduler_reserve_full_isl = True`, `watermark = 0.0`, `async_scheduling`, `stream_interval`.
6. **`vllm/engine/arg_utils.py` — `EngineArgs.get_batch_defaults()`** — https://github.com/vllm-project/vllm/blob/main/vllm/engine/arg_utils.py — the real device- and usage-context-dependent defaults table in §5, including the literal `"a100" not in device_name` check and its source comment referencing PR #17885. Also the `--kv-cache-memory-bytes` CLI wiring.
7. **`vllm/config/parallel.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/parallel.py — `tensor_parallel_size`, `pipeline_parallel_size`, `data_parallel_size` (all default 1), `enable_expert_parallel`, `distributed_executor_backend`.
8. **`vllm/config/compilation.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/compilation.py — `cudagraph_capture_sizes` generation `[1,2,4] + range(8,256,8) + range(256, max+1, 16)`, `max_cudagraph_capture_size` capped at 512 (1024 on data-centre Blackwell), `cudagraph_num_of_warmups`, `cudagraph_specialize_lora`.
9. **`vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/metrics/loggers.py — the exact metric names used throughout: `vllm:kv_cache_usage_perc`, `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:num_requests_waiting_by_reason` (`capacity` / `deferred`), `vllm:num_preemptions_total`, `vllm:iteration_tokens_total`, `vllm:prefix_cache_queries_total` / `vllm:prefix_cache_hits_total`. Matches [module 05 lesson 06](../../05-gpu-observability/lessons/06-inference-slos.md), which verified this set. **Correction:** `vllm:gpu_cache_usage_perc` was renamed to `vllm:kv_cache_usage_perc`; panels on the old name silently return no data.

**Primary sources — vLLM documentation (in-tree, read from the cloned repo)**

10. **`docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — the `-O0`–`-O3` optimization levels; the three startup accelerators (compile-cache reuse under `VLLM_CACHE_ROOT` with `VLLM_FORCE_AOT_LOAD=1`, `--kv-cache-memory` to skip profiling, `--enforce-eager`); the preemption section confirming RECOMPUTE is the only V1 mode; the `max_num_batched_tokens` ITL-vs-TTFT trade with the concrete 2048 / >8192 guidance; parallelism strategy selection; NUMA binding; **and the CPU-provisioning section with the `2 + N` physical-core floor** and the busy-loop warning. *(The rendered docs at docs.vllm.ai are behind this environment's egress proxy; this was read from `docs/` in the cloned repository at the pinned commit, which is the same source the site is built from.)*
11. **`docs/configuration/conserving_memory.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/conserving_memory.md — the ordered memory-reduction levers (TP, quantisation, `max_model_len`/`max_num_seqs`, trimming `cudagraph_capture_sizes`, `enforce_eager`), and the note that with TP > 1 each worker reads the whole checkpoint unless you use a sharded format.
12. **`docs/benchmarking/sweeps.md`** — https://github.com/vllm-project/vllm/blob/main/docs/benchmarking/sweeps.md — `vllm bench sweep serve` with `--serve-params` / `--bench-params`, whose worked example is literally a `max_num_seqs` × `max_num_batched_tokens` grid; and the note that the harness calls the `/reset_*_cache` endpoints between runs, which is why §6's benchmarking caveat matters. Used directly in 07.5.

**Deeper dives**

13. **07.2 — KV cache as a concurrency problem** — [02-kv-cache-concurrency.md](02-kv-cache-concurrency.md) — the arithmetic this lesson's flags operate on: `kv_bytes_per_token`, `num_gpu_blocks`, the three-ceiling diagnostic, and the levers table. If §2's pool arithmetic felt fast, that lesson derives it.
14. **Module 05 lesson 06 — Inference SLOs** — [../../05-gpu-observability/lessons/06-inference-slos.md](../../05-gpu-observability/lessons/06-inference-slos.md) — the verified V1 metric names and the renamed/removed set, the TTFT decomposition into queue-wait plus prefill, and the documented fact that preemption distorts the very intervals you are quantiling (decode-phase preemption stretches inter-token and decode intervals; prefill-phase preemption stretches TTFT). That is the measurement-side companion to §7's mechanism.
