---
lesson: "07.9"
title: "Model loading and storage"
module: "07"
concept: "Model loading and storage"
status: not-started
est_time: "6h"
prev: "08-autoscaling-inference.md"
next: "10-multi-model-lora.md"
artifacts: []
sources: 14
---
# 07.9 · Model loading and storage

> **Concept.** Cold-start latency is a storage-architecture problem, and storage architecture is what decides whether scale-to-zero saves money or just breaks your SLO.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.8 ended on an uncomfortable result. You measured a control-loop dead time, found it
dominated by a single term — `T_start`, the time from "pod scheduled" to "pod serving
tokens" — and discovered that at 170 seconds no choice of autoscaling metric, threshold or
rate limit protects a sub-second TTFT SLO through a step change in load. The lesson priced
scale-to-zero and, for a customer-facing endpoint, the answer was no.

This lesson opens `T_start` up. It decomposes into three physical transfers and one
compilation step, each with a bandwidth you can look up and an arithmetic you can do. Once
decomposed, the question stops being "how do I make cold starts faster" — which is
unanswerable — and becomes "which of these four stages is my bottleneck, what is its
bandwidth, and what architectural change moves it," which is answerable and usually has a
2–20× answer available.

07.10 then asks the complementary question: once a model *is* loaded, how many logical models
can you serve from that one resident copy? Both lessons are about extracting more served
traffic per GPU-dollar without buying GPUs — 09 attacks the **wake-up** cost, 10 attacks the
**steady-state multiplexing**.

**Version pin.** vLLM behaviour is read at **v0.27.1**, cross-checked against `main` @
`c1e4387` (2026-08-17): `vllm/config/load.py`, `vllm/config/compilation.py`,
`vllm/v1/worker/gpu_worker.py`, `vllm/v1/engine/core.py`, `vllm/benchmarks/startup.py`,
`docs/models/extensions/runai_model_streamer.md`, `docs/configuration/optimization.md`.
The safetensors format is read from `huggingface/safetensors` (`README.md` §Format and
`safetensors/src/tensor.rs`). Bandwidth figures are vendor specifications with the achieved
fraction stated separately — **measure yours**, because achieved bandwidth is a property of
your kernel, your filesystem and your driver, not of the spec sheet.

## Why this matters

The GPU meter runs at $2–$12/hour whether the card is decoding tokens or sitting empty
waiting for a `read()` to return. That single fact makes every second of cold start a
compound cost: you pay for the GPU during the wait, and you pay again in latency for whoever
is at the other end of the first request.

The tempting move at 70–90 % idle is to drop replicas to zero. The catch nobody costs out
until production is that the first request afterwards pays the full cold start: schedule a
pod, pull a multi-gigabyte image, move tens to hundreds of gigabytes of weights into VRAM,
compile, and capture graphs. Done naively — a 70B model pulled from Hugging Face over a
shared internet egress — that user waits **twenty minutes or more** for a first token. Done
with a node-local NVMe cache and a persisted compile cache, the same model wakes in **under a
minute**. The most aggressive production systems, using GPU-memory snapshotting, get a vLLM
server under fifteen seconds.

That is a 100× range, and it is entirely a function of *where the bytes live and how they
travel* — not of the GPU, not of the model, not of the autoscaler. It decides whether
scale-to-zero is a cost win or an outage, whether a node failure is a blip or a fifteen-minute
capacity hole, and how fast you can roll a new model version across a fleet.

There is a second, less obvious payoff. The same understanding tells you why your *deploys*
are slow, why a rolling update on a 40-replica fleet takes an hour, and why the third replica
on a node starts faster than the first. All three are page-cache and bandwidth questions, and
all three are invisible until you decompose the load path.

## What's new here (calibration)

Referenced, not re-taught: the startup timeline and its log lines (07.4 §1); CUDA graphs,
`torch.compile` and the `-O` levels (07.4 §9); FP8 and INT4 weight sizes (07.7); dead time
and the scale-to-zero economics (07.8).

Genuinely new:

1. **The load path as a bandwidth pipeline** — storage → page cache → host RAM → PCIe → HBM
   — with each hop's real number, so the bottleneck stage is a division rather than a guess.
2. **The safetensors format itself**: the 8-byte header length, the JSON header with
   `data_offsets`, the `__metadata__` key, the 100 MB header cap, and *why* that layout makes
   zero-copy `mmap` possible when a pickle checkpoint cannot be streamed at all.
3. **vLLM's `safetensors_load_strategy`** — `lazy` / `eager` / `prefetch` / `torchao`, the
   automatic NFS detection, and the 8-thread / 16 MiB prefetch defaults — which is a
   two-line change that frequently doubles load speed on network storage.
4. **Pinned versus pageable host memory**, and why the naive `mmap` → `cudaMemcpy` path
   achieves roughly half of PCIe's line rate.
5. **Streaming loaders as pipelining**, not magic: `runai_streamer`, `tensorizer`,
   `instanttensor`, and the sharded variants — with the `concurrency` and `memory_limit`
   knobs that actually control them.
6. **Why TP multiplies your load time** unless you shard the checkpoint, and the two vLLM
   load formats that fix it.
7. **`vllm bench startup`**, the in-tree harness that measures cold and warm startup with
   percentiles and separates compilation time from the rest.
8. **The cold-start budget as arithmetic** — from model bytes, storage bandwidth and PCIe
   bandwidth to a scale-up latency and therefore to a `behavior.scaleUp.periodSeconds`.

## Core concepts

### 1. The cold start, decomposed

Every second between "pod scheduled" and "first token" belongs to exactly one of five stages.
They are not interchangeable, they have different bottlenecks, and reaching for the wrong
lever is the standard failure.

```
  THE COLD-START PIPELINE — WHERE EVERY BYTE TRAVELS, AND HOW FAST
  ══════════════════════════════════════════════════════════════════════════════

  ┌── STAGE 0 · SCHEDULE ────────────────────────────────────────────────────┐
  │ kube-scheduler places pod; device plugin binds nvidia.com/gpu            │
  │ warm node: 2–10 s   ·   cold node (Karpenter/CA): 60–300 s               │
  │ BOUND BY: cloud API + kubelet join + device-plugin registration          │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
  ┌── STAGE 1 · IMAGE PULL ──────────────────────────────────────────────────┐
  │  registry ──── network ───▶ node disk ──── decompress ───▶ overlayfs     │
  │  vLLM+CUDA+PyTorch image: 5–12 GB compressed, 15–25 GB extracted         │
  │  cached on node: ~0 s                                                    │
  │  in-region registry @ 1 GB/s: ~10 s pull + 30–60 s DECOMPRESS            │
  │  cross-region / rate-limited: 2–6 min                                    │
  │  BOUND BY: network, then SINGLE-THREADED gzip decompression              │
  │  ⚠ decompression is often the larger half and no faster link fixes it    │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
  ┌── STAGE 2 · STORAGE READ ────────── the usual bottleneck ────────────────┐
  │                                                                          │
  │   object store / NFS / NVMe ──▶ page cache ──▶ process address space     │
  │                                                                          │
  │   HF Hub over shared egress   0.1– 0.3 GB/s  ├─ 16 GB model: 53–160 s    │
  │   S3/GCS, 1 stream            0.05–0.2 GB/s  │  140 GB model: 8–23 min   │
  │   S3/GCS, 16 concurrent       1  – 5  GB/s   │                           │
  │   NFS / EFS / Filestore       0.3 – 1  GB/s  │  16 GB: 16–53 s           │
  │   EBS gp3 (@16k IOPS/1000MB)  0.5 – 1  GB/s  │                           │
  │   EBS io2 Block Express       1  – 4  GB/s   │                           │
  │   local NVMe, PCIe Gen4 x4    3  – 7  GB/s   ├─ 16 GB: 2.3–5.3 s         │
  │   local NVMe, PCIe Gen5 x4    7  –14  GB/s   │  140 GB: 10–20 s          │
  │   page cache (2nd pod, warm) 10  –25  GB/s   ├─ 16 GB: 0.6–1.6 s         │
  │                                                                          │
  │  BOUND BY: the slowest link in that chain. NOT by the GPU.               │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
  ┌── STAGE 3 · HOST → DEVICE COPY ──────────────────────────────────────────┐
  │                                                                          │
  │   host RAM ──── PCIe ────▶ HBM                                           │
  │                                                                          │
  │   PCIe Gen4 x16   31.5 GB/s theoretical │ 22–26 GB/s achieved (PINNED)   │
  │                                          │ 8–13 GB/s (pageable/mmap)     │
  │   PCIe Gen5 x16   63.0 GB/s theoretical │ 45–55 GB/s achieved (PINNED)   │
  │                                          │ 15–25 GB/s (pageable/mmap)    │
  │   NVLink-C2C (GH200/GB200)  900 GB/s aggregate — host memory is          │
  │                             coherent; this stage nearly disappears       │
  │                                                                          │
  │   16 GB @ 24 GB/s pinned Gen4  = 0.67 s   ← almost never the bottleneck  │
  │   140 GB @ 24 GB/s             = 5.8 s      unless everything else is    │
  │                                             already fast                 │
  │  LOG: "Model loading took 14.99 GiB memory and 11.42 seconds"            │
  │       (covers stages 2 AND 3 together)                                   │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
  ┌── STAGE 4 · PROFILE, COMPILE, CAPTURE ───────────────────────────────────┐
  │  memory profiling dummy forward pass          5–10 s                     │
  │  torch.compile   cache HIT   2–8 s   ·  cache MISS  30–120 s             │
  │  CUDA graph capture (51 sizes by default)     15–40 s                    │
  │  BOUND BY: CPU (compilation) and GPU (capture). NO I/O.                  │
  │  LOG: "init engine (profile, create kv cache, warmup model) took 30.12 s │
  │        (compilation: 8.44 s)"                                            │
  │  LOG: "Graph capturing finished in 21 secs, took 0.94 GiB"               │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
                              first token
```

**The structural fact this diagram exists to convey:** stages 1, 2 and 3 are **I/O-bound**,
and stage 4 is **CPU- and GPU-bound with no I/O at all**. A faster GPU does nothing for the
first three. More replicas do nothing for any of them. The levers are entirely
architectural — move the bytes closer, read them with more parallelism, or avoid reading
them at all.

**And the stages have wildly different fixed costs.** Stage 4 is roughly constant for a given
(model, config, hardware) — it does not care whether the weights came from NVMe or from
Hugging Face. So as you optimise stages 1–3, stage 4 becomes the floor, and it is a floor of
20–60 seconds unless you attack the compile cache specifically.

### 2. How many bytes are you actually moving?

Weight bytes are `parameters × bytes_per_parameter`, and that is the only number stages 2 and
3 care about.

| Model | Params | BF16 (2 B) | FP8 (1 B) | INT4 group-128 (~0.53 B) |
|---|---|---|---|---|
| Llama-3.1-8B | 8.03e9 | 16.1 GB | 8.0 GB | 4.3 GB |
| Llama-3.3-70B | 70.6e9 | 141 GB | 70.6 GB | 37.4 GB |
| Qwen2.5-72B | 72.7e9 | 145 GB | 72.7 GB | 38.5 GB |
| Llama-3.1-405B | 405e9 | 810 GB | 405 GB | 215 GB |

The INT4 column is not `0.5 B` because a 4-bit group-quantised checkpoint also stores a scale
(and often a zero-point) per group. At group size 128 with FP16 scales that is
`4 + 16/128 = 4.125` bits per weight for symmetric schemes, and `4.25` with a zero-point —
hence "4-bit" checkpoints landing at ~0.53 bytes per parameter rather than 0.5. (07.7 §3
derives this.)

**This is why quantization pays twice.** 07.7 sold FP8 on throughput and VRAM. It also halves
stage 2 and stage 3, which for a 70B on a 1 GB/s network filesystem is 141 s → 70 s of load
time. On a fleet doing frequent scale events that saving is often worth more operationally
than the throughput one.

Two bytes you must not forget:

- **The tokenizer and config files** are small (megabytes) but involve extra round trips, and
  on a cold HF-Hub path each round trip carries connection setup and possible rate limiting.
- **`--ignore-patterns` defaults to `["original/**/*"]`** (`vllm/config/load.py`) precisely
  because Llama checkpoints ship a second copy of the weights in a different format under
  `original/`. Without that default you would download the model twice. If you are
  hand-rolling a weight sync job, replicate that exclusion or you will double your bytes.

### 3. Stage 2 in depth: the safetensors format, and why the layout is the performance

You cannot stream a format that was not designed to be streamed. The reason safetensors is
the default is not branding; it is that the layout permits `mmap` and random access, and
pickle's does not.

**The format**, from `huggingface/safetensors`:

```
  SAFETENSORS FILE LAYOUT
  ══════════════════════════════════════════════════════════════════════════════

  byte 0        8                            8+N                        EOF
  ├────────────┼────────────────────────────┼──────────────────────────────┤
  │  N: u64 LE │  JSON header, N bytes      │  raw tensor byte buffer      │
  │  (8 bytes) │  (must start with '{',     │  (little-endian, C/row-major,│
  │            │   may be space-padded)     │   fully indexed, no holes)   │
  └────────────┴────────────────────────────┴──────────────────────────────┘

  The JSON header:
  {
    "__metadata__": {"format": "pt"},            ← optional; STRING values only
    "model.layers.0.self_attn.q_proj.weight": {
        "dtype": "BF16",
        "shape": [4096, 4096],
        "data_offsets": [0, 33554432]            ← [begin, end) RELATIVE to the
    },                                             start of the byte buffer,
    "model.layers.0.self_attn.k_proj.weight": {    not to the start of the file
        "dtype": "BF16",
        "shape": [1024, 4096],
        "data_offsets": [33554432, 41943040]
    },
    ...
  }

  Constraints that matter operationally (safetensors/src/tensor.rs):
    • MAX_HEADER_SIZE = 100_000_000 (100 MB). A header larger than this is
      rejected — a real limit for checkpoints with very many small tensors.
    • Duplicate keys are disallowed.
    • The byte buffer must be ENTIRELY indexed with no holes. This is a
      deliberate anti-polyglot measure: you cannot hide a payload in the gaps.
    • Endianness is little-endian; layout is C (row-major).
```

**Why that layout is fast, mechanically:**

1. **The header tells you every tensor's exact byte range before you read any data.** So a
   loader can `mmap` the whole file and hand each tensor a view into it, or issue N parallel
   ranged reads, or fetch only the tensors this TP rank needs. All three are impossible
   without an index.
2. **Zero-copy on the CPU path.** `safe_open` + `get_tensor` maps the file and constructs a
   tensor pointing at the mapped pages — no allocation, no deserialisation, no copy. The
   safetensors documentation reports a **76.6×** CPU load speedup versus `torch.load` on
   GPT-2 (0.004 s versus 0.307 s) for exactly this reason.
3. **On the GPU path the win is smaller and worth understanding.** The same doc measures
   **2.1×** on a T4 (0.165 s versus 0.354 s), because the GPU path still has to move the
   bytes over PCIe — the library skips the *CPU* allocation and calls `cudaMemcpy` directly
   from the mapping, but it cannot skip the wire. **The 76× is a CPU-path number and quoting
   it for GPU loading is wrong**, which is why §5 treats the H2D copy as its own stage.

**Why pickle is both slow and unsafe, and why that is the same property.** A `.bin`
checkpoint is a Python pickle: an instruction stream for a stack machine that reconstructs
objects, including `GLOBAL`/`REDUCE` opcodes that import and call arbitrary callables. You
cannot know a tensor's byte range without executing the stream up to that point, so you
cannot `mmap` it, cannot read it in parallel, and cannot fetch a subset. **The
arbitrary-deserialisation flexibility that makes it dangerous is the same flexibility that
makes it unstreamable.** A 2023 security audit of safetensors by Trail of Bits, commissioned
by Hugging Face, EleutherAI and Stability AI, found no critical remote-code-execution flaw in
the format and its findings shipped upstream — so "fast" and "safe" happen to be the same
format choice. If you inherit a pipeline still producing `.bin`, converting is close to a
free win on both axes.

### 4. Stage 2, continued: vLLM's load strategies and why NFS needs a different one

`mmap` is the right default on local storage and the wrong default on a network filesystem,
and vLLM exposes the switch (`vllm/config/load.py`).

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --safetensors-load-strategy eager       # lazy | eager | prefetch | torchao
```

| Strategy | What it does | Use when |
|---|---|---|
| *(unset, default)* | memory-mapped (lazy) — **but** auto-enables prefetch when an **NFS filesystem is detected** and the checkpoint fits in 90 % of available RAM | most cases; the auto-detection covers the common trap |
| `lazy` | `mmap` only; **suppresses** the NFS auto-prefetch | local NVMe, or when you want to control it yourself |
| `eager` | read the whole file into CPU memory up front, then load | **network filesystems (NFS, Lustre, EFS)** — avoids inefficient random reads; costs host RAM |
| `prefetch` | read files into the OS page cache in the background before workers load them | network or high-latency storage, especially with TP where multiple workers read the same files |
| `torchao` | load then reconstruct into torchao tensor subclasses | torchao-quantised checkpoints saved as safetensors; needs `torchao >= 0.14.0` |

Prefetch tuning, with the defaults from the source:

```bash
  --safetensors-prefetch-num-threads 8          # DEFAULT_SAFETENSORS_PREFETCH_NUM_THREADS = 8
  --safetensors-prefetch-block-size  16777216   # DEFAULT ..._BLOCK_SIZE = 16 * 1024 * 1024
```

**The mechanism behind "NFS needs `eager`."** `mmap` turns tensor access into page faults.
On local NVMe a fault is a 4 KB–2 MB read served in microseconds with readahead doing most
of the work. On NFS the same fault is a network round trip, and the access pattern generated
by a loader walking tensors in header order is *not* sequential in file order once you factor
in per-rank slicing and dtype conversion. You get a storm of small, latency-bound,
poorly-coalesced reads, and a 1 GB/s filesystem delivers 100 MB/s. Reading the whole file
sequentially first — `eager` — converts that into one streaming read at line rate, at the
cost of holding the checkpoint in host RAM. **That is why a model that loads in 20 seconds
from a local disk can take five minutes from a network PVC, and why the fix is a flag rather
than a bigger PVC.**

### 5. Stage 3: PCIe, and why the naive path gets half the line rate

Once bytes are in host memory they still have to cross PCIe. This stage is rarely the
bottleneck, but it is worth understanding because the difference between doing it well and
doing it naively is 2×.

```
  PCIe Gen4 x16 : 16 GT/s x 16 lanes, 128b/130b encoding
                = 16e9 x 16 x (128/130) / 8 bytes = 31.5 GB/s each direction
  PCIe Gen5 x16 : 32 GT/s x 16 lanes             = 63.0 GB/s each direction
```

Achieved is lower, and how much lower depends on one thing: **whether the host buffer is
page-locked.**

- **Pageable memory** (ordinary `malloc`, or an `mmap`ed file) cannot be DMA'd from directly,
  because the OS may move or evict those pages mid-transfer. The driver therefore stages
  through an internal pinned bounce buffer: one CPU-side `memcpy` into pinned memory, then
  DMA. That is an extra full pass over the data through the CPU's memory subsystem, and it
  serialises with the transfer. Typical achieved: **8–13 GB/s on Gen4, 15–25 GB/s on Gen5.**
- **Pinned (page-locked) memory** — `cudaHostAlloc` / `torch.empty(..., pin_memory=True)` —
  can be DMA'd directly. Typical achieved: **22–26 GB/s on Gen4, 45–55 GB/s on Gen5**, i.e.
  70–87 % of theoretical.

The default `mmap` → `cudaMemcpy` path is by construction the pageable one. This is precisely
what a streaming loader fixes: it reads into its own pinned buffers and DMAs from them.

**On Grace Hopper (GH200) and Grace Blackwell this stage largely disappears.** NVLink-C2C
provides 900 GB/s of coherent CPU↔GPU bandwidth, so the host-to-device copy stops being a
distinct cost and the bottleneck moves entirely to stage 2. If you are sizing a cold-start
budget on such a platform, delete stage 3 from the arithmetic rather than scaling it.

**Sanity check the whole stage.** For Llama-3.1-8B at bf16, 16.1 GB:

```
  pinned Gen5   : 16.1 / 50   = 0.32 s
  pinned Gen4   : 16.1 / 24   = 0.67 s
  pageable Gen4 : 16.1 / 10   = 1.61 s
```

Against a stage-2 read of 2–160 seconds depending on tier, **stage 3 is noise unless stage 2
is already fast.** Optimise in order.

### 6. Streaming loaders: pipelining, not magic

Naive loading is **serial**: read all the bytes to disk/RAM, *then* copy them to the GPU. A
streaming loader **overlaps** the two, so the wall clock approaches `max(read, copy)` instead
of `read + copy`, and it uses concurrency to raise the read term itself.

```
  SERIAL vs PIPELINED LOADING — 70B FP8 (70 GB), 2 GB/s storage, 24 GB/s PCIe
  ══════════════════════════════════════════════════════════════════════════════

  ── SERIAL (default HF loader path) ────────────────────────────────────────
   read  ████████████████████████████████████  35.0 s
   H2D                                       ███  2.9 s
                                                 └─ total 37.9 s

  ── PIPELINED, 1 reader thread ─────────────────────────────────────────────
   read  ████████████████████████████████████  35.0 s
   H2D    ███████████████████████████████████   overlapped
                                              └─ total ≈ 35.3 s   (read-bound)

  ── PIPELINED, 16 concurrent readers (object storage) ──────────────────────
   read  ██████  aggregate 8 GB/s ⇒ 8.8 s
   H2D    ████████  2.9 s, overlapped
                  └─ total ≈ 9.2 s

  ┌────────────────────────────────────────────────────────────────────────┐
  │ CONCURRENCY IS THE LEVER ON OBJECT STORAGE, OVERLAP IS THE LEVER       │
  │ EVERYWHERE. A single S3 stream delivers 50–200 MB/s regardless of      │
  │ your NIC; 16 streams deliver 1–5 GB/s on the same NIC. The default     │
  │ loader opens one.                                                      │
  └────────────────────────────────────────────────────────────────────────┘
```

vLLM's `--load-format` options (`vllm/config/load.py`), with what each is actually for:

| `--load-format` | Mechanism | Use when |
|---|---|---|
| `auto` *(default)* | safetensors, falling back to `.bin` | the default; fine on local NVMe |
| `safetensors` | force safetensors | you want the fallback to be an error |
| `runai_streamer` | Run:ai Model Streamer: concurrent reads into CPU buffers, streamed to GPU. Reads `s3://`, `gs://`, `az://` and local paths | **object storage, or any high-latency tier** |
| `runai_streamer_sharded` | same, from pre-sharded `model-rank-{rank}-part-{part}.safetensors` | TP > 1 (see §7) |
| `tensorizer` | CoreWeave Tensorizer: serialised model streamed from S3/HTTP straight into GPU memory | object storage; a pre-serialisation step is required |
| `instanttensor` | distributed loading with pipelined prefetch and direct I/O, CUDA only | large models on fast local storage |
| `sharded_state` | pre-sharded per-rank checkpoint, plain reader | TP > 1 with local storage |
| `modelexpress` | ModelExpress backend | that platform |
| `bitsandbytes` | load with bnb quantisation applied | bnb checkpoints |
| `dummy` | random weights, no I/O at all | **profiling** — isolates stage 4 from stages 2–3 |

`dummy` deserves a callout: `--load-format dummy` initialises random weights and skips the
read entirely, which makes it the cleanest way to measure stage 4 in isolation. Run once with
`dummy` and once normally, and the difference is your stages 2+3.

Run:ai Model Streamer's tunables, passed through `--model-loader-extra-config`:

```bash
pip install "vllm[runai]==0.27.1"

# From object storage, with explicit concurrency.
vllm serve s3://models/llama-3.1-8b-instruct \
  --load-format runai_streamer \
  --model-loader-extra-config '{"concurrency": 16, "memory_limit": 5368709120}'

# Distributed streaming: significantly better from object storage or a
# high-throughput network share (CUDA/ROCm only).
vllm serve /models/llama-3.1-8b-instruct \
  --load-format runai_streamer \
  --model-loader-extra-config '{"distributed": true}'
```

- **`concurrency`** — the number of OS threads reading tensors into the CPU buffer; for S3 it
  is the number of client connections opened. This is the knob that converts one 100 MB/s
  stream into sixteen.
- **`memory_limit`** — a cap in bytes on the CPU staging buffer. Set it on memory-constrained
  nodes; without it a large model can push the node into reclaim, which is slower than the
  read you were optimising.
- **`distributed`** — pipelined distributed streaming, CUDA/ROCm only.

For S3-compatible stores that are not AWS:

```bash
RUNAI_STREAMER_S3_USE_VIRTUAL_ADDRESSING=0 \
AWS_EC2_METADATA_DISABLED=true \
AWS_ENDPOINT_URL=https://storage.googleapis.com \
vllm serve s3://core-llm/Llama-3-8b --load-format runai_streamer
```

### 7. The TP multiplier, and the sharded-checkpoint fix

A trap that scales exactly wrong. vLLM's documentation states it plainly: **with tensor
parallelism enabled, each worker process reads the whole model and slices it**, so disk
reading time grows proportionally to the TP size.

```
  70B FP8 (70 GB) on 4 GPUs, TP=4, from a 2 GB/s filesystem
  ══════════════════════════════════════════════════════════════════════════════

  UNSHARDED CHECKPOINT
    each of 4 workers reads the FULL 70 GB
    aggregate bytes read                       = 280 GB
    if the filesystem is per-node 2 GB/s       = 140 s     ← 4x the single-GPU time
    if the page cache absorbs 3 of the 4 reads = ~40–60 s  ← if RAM > 70 GB

  SHARDED CHECKPOINT (--load-format sharded_state or runai_streamer_sharded)
    each worker reads ONLY its own 17.5 GB shard
    aggregate bytes read                       = 70 GB
    reads run in PARALLEL across 4 file handles
    ⇒ ≈ 70 / 2 GB/s                            = 35 s, INDEPENDENT of TP size
```

The docs' own summary: "the model loading time should remain constant regardless of the size
of tensor parallelism." Producing the sharded checkpoint is a one-off conversion
(`examples/features/sharded_state/save_sharded_state_offline.py`), and the expected filename
pattern is `model-rank-{rank}-part-{part}.safetensors`, overridable via
`--model-loader-extra-config '{"pattern": "..."}'`.

**The page-cache caveat is important and under-appreciated.** If node RAM comfortably exceeds
the checkpoint size, the first worker's read populates the page cache and the other three are
served from memory at 10–25 GB/s — so the TP penalty appears to be small. Then you deploy a
70B on a node with 200 GB of RAM, the checkpoint no longer fits alongside everything else, and
the penalty appears in full. **Test cold, with `echo 3 > /proc/sys/vm/drop_caches` between
runs, or you will measure your page cache and ship a surprise.**

### 8. Stage 4: compilation and graph capture, and how to skip them

Stage 4 has no I/O, which means none of the storage work above touches it. Once stages 1–3
are fast, this is your floor, and it is 20–60 seconds unless you attack it directly.

Its three components (07.4 §9 covers the flags; here is what to do about the time):

- **Memory profiling** — a dummy forward pass to measure the activation peak, plus the
  CUDA-graph memory estimate. **5–10 s.** Skippable with `--kv-cache-memory-bytes`, which
  bypasses the measurement entirely (07.4 §3).
- **`torch.compile`** — **2–8 s on a cache hit, 30–120 s on a miss.** vLLM persists artifacts
  under `VLLM_CACHE_ROOT` (default `~/.cache/vllm`).
- **CUDA graph capture** — one graph per size in `cudagraph_capture_sizes`, which by default
  is `[1,2,4] + range(8,256,8) + range(256, max+1, 16)` with `max_cudagraph_capture_size`
  capped at 512 — **51 graphs, 15–40 s.**

Three concrete levers:

```bash
# 1. PERSIST AND SHIP THE COMPILE CACHE.
#    The directory can be copied between machines or baked into the image.
ENV VLLM_CACHE_ROOT=/opt/vllm-cache
# ...and in the Dockerfile, after installing vLLM, run the model once so the
# cache is populated at build time rather than at every pod start.

# 2. FAIL LOUDLY ON A CACHE MISS instead of silently recompiling.
ENV VLLM_FORCE_AOT_LOAD=1
#    Without this, ANY change to the model, config, relevant VLLM_* env vars,
#    torch build or GPU model invalidates the cache and you silently pay
#    30–120 s per pod — with no error and no log line saying why.

# 3. TRIM THE CAPTURE LIST to the batch sizes you actually run.
vllm serve ... --compilation-config '{"cudagraph_capture_sizes": [1,2,4,8,16,32,64,96]}'
#    8 graphs instead of 51. Saves capture time AND capture memory. Sizes not
#    in the list fall back to the eager path, so include your operating batch.
```

And the two blunt instruments:

- **`-O0`** — no optimizations, fastest startup, lowest steady-state performance. Fine for a
  dev loop, never for production.
- **`--enforce-eager`** — skips compilation and capture entirely. Useful as a *measurement*:
  the difference between a normal boot and an `--enforce-eager` boot is exactly your stage-4
  compile+capture cost.

**What invalidates the compile cache** is the list to put in your runbook, because every entry
is a silent 30–120 second regression: the model (including a revision bump), the engine
config, relevant `VLLM_*` environment variables, the torch build, and the GPU model. That last
one bites hardest on a heterogeneous fleet — an image whose cache was baked on an H100 misses
on every A100 node, and nothing tells you.

### 9. Image pull is a separate byte stream — treat it separately

The single most common architectural error in this area is conflating stage 1 and stage 2.
They are different bytes, from different sources, fixed by different means.

**Do not bake weights into the container image.** It feels fast — everything is in one
artefact — and it is a trap:

- The image becomes 20–160 GB, so **stage 1 now carries the weight-download cost**, and
  layer decompression is largely single-threaded, so you cannot buy your way out with
  bandwidth.
- You **re-pull it on every new node**, whereas a weights volume is mounted.
- You **cannot share one copy across models or versions**; two adapters of the same base are
  two full images.
- **Swapping a checkpoint means rebuilding and re-rolling the image.**
- Registry storage and egress costs multiply by your replica-fleet churn.

Keep the image lean — code, CUDA, PyTorch, vLLM — and let a volume or a streaming loader own
the weights. The one defensible exception is a genuinely small model (a few GB) on a fast
in-region registry, where the operational simplicity of one artefact wins.

**Fixing stage 1 properly:**

| Lever | Removes | Notes |
|---|---|---|
| Registry pull-through cache / mirror **in-region** | most of the network time | also removes rate limiting as a failure mode |
| Pre-pull DaemonSet (a `pause` container referencing the image) | the whole pull on warm nodes | costs node disk; must be updated with the image tag |
| `imagePullPolicy: IfNotPresent` with an immutable digest | redundant pulls | never use `:latest` with `Always` on a GPU fleet |
| Fewer, larger layers; order by change frequency | decompression time | the CUDA/PyTorch layer should never be rebuilt |
| Lazy-pulling snapshotters (eStargz, SOCI-style) | starts the container before the whole image lands | real benefit, real operational complexity; evaluate before adopting |

### 10. The Kubernetes patterns that actually move the number

Four patterns, in increasing order of both effectiveness and operational cost.

**(a) A shared read-only PVC.** Simplest. One `ReadWriteMany` volume holds the weights; every
pod mounts it. Removes the download entirely and gets you network-filesystem bandwidth
(0.3–1 GB/s). **Pair it with `--safetensors-load-strategy eager`** (§4) or you will get NFS
random-read behaviour and a fraction of that.

```yaml
      volumes:
        - name: models
          persistentVolumeClaim:
            claimName: model-weights-rwx
            readOnly: true
      containers:
        - name: vllm
          args:
            - --model=/models/Llama-3.1-8B-Instruct
            - --safetensors-load-strategy=eager      # ← the load-bearing line
          volumeMounts:
            - { name: models, mountPath: /models, readOnly: true }
```

**(b) Node-local NVMe cache, populated by a DaemonSet.** The biggest single win available for
most fleets: it turns stage 2 from a network read into a local read, 0.3–1 GB/s → 3–14 GB/s,
and once the page cache is warm the *second* pod on that node loads at 10–25 GB/s.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: model-warmer, namespace: inference }
spec:
  selector: { matchLabels: { app: model-warmer } }
  template:
    metadata: { labels: { app: model-warmer } }
    spec:
      nodeSelector: { node.kubernetes.io/instance-type: p5.48xlarge }
      containers:
        - name: sync
          image: registry.example.com/model-sync:v3     # aws-cli / gsutil / rclone
          command: ["/bin/sh","-c"]
          args:
            - |
              set -eu
              DEST=/cache/Llama-3.1-8B-Instruct
              mkdir -p "$DEST"
              # --exact-timestamps so a re-uploaded checkpoint is actually re-synced
              aws s3 sync s3://models/Llama-3.1-8B-Instruct "$DEST" \
                  --exact-timestamps --only-show-errors
              # Warm the page cache so the FIRST pod is fast too, not just the second.
              cat "$DEST"/*.safetensors > /dev/null
              sleep infinity
          volumeMounts:
            - { name: nvme, mountPath: /cache }
          resources:
            requests: { cpu: "2", memory: 8Gi }
      volumes:
        - name: nvme
          hostPath: { path: /mnt/nvme/models, type: DirectoryOrCreate }
```

The serving pods then mount the same `hostPath` read-only. Two details that matter: the
`cat > /dev/null` is what warms the page cache (without it, the first pod still pays a disk
read), and `--exact-timestamps` is what makes a re-uploaded checkpoint actually re-sync
rather than being skipped on a size match.

**(c) An `initContainer` that fetches into an `emptyDir`.** Use when you cannot run a
DaemonSet or the node has no local disk. It makes the fetch explicit and observable — the pod
stays in `Init:0/1` while it runs, so `kubectl get pods` shows you the stage — but it pays the
fetch **per pod**, so it does not amortise across replicas on a node.

**(d) A warm floor.** `minReplicaCount ≥ 1` (07.8). Removes the cold start entirely for the
first replica's worth of capacity, at the cost of one continuously-running GPU. Every other
lever here is an attempt to make this unnecessary; sometimes it is simply the correct answer.

### 11. The tier beyond: memory snapshotting

Everything above still moves the bytes into VRAM at least once per cold start. Serverless GPU
platforms are pushing past that floor by snapshotting the *post-initialisation GPU state* —
weights already resident in HBM, CUDA context, compiled kernels and captured graphs — and
restoring the snapshot instead of re-running load and warmup. Modal has published cutting a
vLLM server's cold start from roughly two minutes to about ten seconds this way, with a
follow-up on a Mistral model showing a similar ~10× cut (≈118 s to ≈12 s median).

Structurally this is different from everything in §6–§10: it does not optimise stage 3, it
**skips stages 2, 3 and most of 4** and replaces them with one memory-restore operation. Its
limits are equally instructive — the snapshot is tied to a specific GPU model, driver and
process layout, and it is a platform-engineering project rather than a config flag. Know it
exists, know why it is a step beyond NVMe caching, and know that vLLM's **sleep mode** (07.8
§8) is the in-process cousin you *can* turn on today: level 1 keeps the process, CUDA context
and compiled graphs alive while offloading weights to CPU RAM, so waking is a PCIe copy
rather than a storage read plus compile plus capture.

### 12. Measuring it: `vllm bench startup`

vLLM ships a harness for exactly this, and it distinguishes cold from warm.

```bash
vllm bench startup \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --num-iters-cold 3 \
  --num-iters-warmup 1 \
  --num-iters-warm 3 \
  --output-json startup.json
```

From `vllm/benchmarks/startup.py`: **cold** iterations run with *temporary cache directories*
so nothing is reused; **warm** iterations use the cached compilation and model info. For each
phase it reports `total_startup_time` and `compilation_time` (plus `encoder_compilation_time`
for multimodal models), with percentiles at 10/25/50/75/90/99.

**The cold-versus-warm delta is your compile-and-model-info cache value**, quantified without
argument. And the harness composes with the sweep tooling:

```bash
vllm bench sweep startup \
  --startup-cmd 'vllm bench startup --model meta-llama/Llama-3.1-8B-Instruct' \
  --serve-params serve_hparams.json \
  --output-dir results --experiment-name startup
```

so you can compare startup across `tensor_parallel_size`, `load_format`, `gpu_memory_utilization`
and so on with one invocation.

For the stages the harness does not separate, use log timestamps and two ablations:

```bash
# Stage 2+3 only (weights): compare a normal boot against dummy weights.
vllm serve <model> --load-format dummy    # random weights, no I/O
# ⇒ Δ(normal, dummy) = stages 2 + 3

# Stage 4 compile+capture only: compare a normal boot against eager.
vllm serve <model> --enforce-eager
# ⇒ Δ(normal, eager) = compile + graph capture
```

## Perspectives

**The storage-engineer view.** Every number in §1 is a bandwidth divided into a byte count,
and the discipline that makes this tractable is refusing to reason about "cold start" as one
thing. Decompose, measure each stage, find the division that dominates, and fix *that*. The
same method applies to database restore times, CI image builds and CDN fills; what makes it
feel novel here is only that the last hop is PCIe instead of a socket. If you have optimised
a data pipeline you already own the technique.

**The platform-economics view.** Cold start is what decides whether 07.8's scale-to-zero
verdict can flip. At 170 s, an interactive endpoint must keep a warm floor and you pay for
idle GPUs. At 20 s it becomes arguable. Under 10 s — snapshot territory — it becomes the
default for most endpoints, and your fleet's utilisation moves from 35 % toward 70 %, which
is a larger cost lever than anything in 07.5 or 07.7. **Storage architecture is therefore an
inference-cost lever that shows up nowhere in an inference benchmark**, which is exactly why
it gets under-invested.

**The reliability view.** Cold start is also your blast radius during an incident. If a node
fails and its replicas take fifteen minutes to come back, you have a fifteen-minute capacity
hole, and any surge protection you had is gone for that window. Teams that measure cold start
only for autoscaling purposes miss that the same number governs recovery. A useful reframing
for a design review: *"how long are we down for, per lost replica?"*

**The security view.** The safetensors format is the rare case where the fast choice and the
safe choice are the same choice, and for the same underlying reason — pickle's arbitrary
deserialisation is what makes it both dangerous and unstreamable. There is a second-order
point worth carrying: a model checkpoint is executable-adjacent supply chain. Pin revisions,
verify digests, and prefer a format whose parser cannot call `os.system`. That argument is
much easier to make when the safe format is also 76× faster to load on the CPU path.

**The skeptic's view.** Every mitigation here has a cost that the headline number omits. A
node-local NVMe cache needs a sync job, cache-invalidation logic and disk capacity planning,
and a stale cache serves the wrong weights silently. A pre-pull DaemonSet must be updated in
lockstep with your image tag or it warms the wrong thing. Baking the compile cache into an
image ties that image to a GPU model. Snapshotting is a platform to build, not a flag. The
honest sequencing is: measure first, take the biggest division, and stop when the cold start
is short enough for the decision it gates — not when you have run out of techniques.

## Real-world use cases

- **The safetensors format specification and its measured speedups
  (`huggingface/safetensors`).** The format is 8 bytes of header length, a JSON header
  mapping tensor names to `{dtype, shape, data_offsets}`, and a fully-indexed raw byte
  buffer, with `MAX_HEADER_SIZE = 100_000_000` and no holes permitted in the buffer. The
  project's own benchmark reports **76.6×** faster loading than `torch.load` on CPU for GPT-2
  (0.004 s vs 0.307 s) and **2.1×** on a T4 GPU (0.165 s vs 0.354 s). **What it shows:** two
  things at once. The CPU number is what an index plus `mmap` buys — no deserialisation, no
  copy. The GPU number is much smaller *because the bytes still have to cross PCIe*, which is
  the cleanest possible demonstration that stages 2 and 3 are different problems. Quoting the
  76× as a GPU loading figure is a common and detectable error.

- **vLLM's automatic NFS detection in `safetensors_load_strategy`.** The default strategy is
  memory-mapped lazy loading, *except* that "when an NFS filesystem is detected and the total
  checkpoint size fits within 90 % of available RAM, prefetching is enabled automatically."
  **What it shows:** the engine's maintainers hit the `mmap`-on-NFS pathology often enough to
  build detection for it. It also shows the shape of the fix — convert a fault-driven random
  read pattern into one sequential streaming read — and gives you the manual override
  (`eager`) for the filesystems the detection does not recognise, which includes most
  CSI-mounted network volumes that do not present as NFS.

- **vLLM's TP loading note and the sharded-state loaders.** The conserving-memory
  documentation states that with tensor parallelism "each process will read the whole model
  and split it into chunks, which makes the disk reading time even longer (proportional to the
  size of tensor parallelism)," and points at a sharded-checkpoint conversion after which
  "the model loading time should remain constant regardless of the size of tensor
  parallelism." **What it shows:** a scaling behaviour that is exactly backwards from
  intuition — adding GPUs makes loading *slower*, not faster — and a fix that is a one-off
  conversion plus a `--load-format` change. It is also a reminder that the page cache can hide
  the problem until node RAM stops covering the checkpoint, at which point it appears at full
  size in production.

- **Run:ai Model Streamer's `concurrency` parameter, and object-storage physics.** The vLLM
  integration exposes `concurrency` as "the level of concurrency and number of OS threads
  reading tensors from the file to the CPU buffer — for reading from S3, the number of client
  instances the host is opening to the S3 server," plus `memory_limit` for the CPU buffer and
  `distributed` for pipelined distributed streaming. **What it shows:** object-storage
  throughput is per-connection, not per-host. A single stream delivers 50–200 MB/s no matter
  how large your NIC is; sixteen streams on the same NIC deliver 1–5 GB/s. The default loader
  opens one. This is the highest-leverage, lowest-effort change available to anyone serving
  weights from S3/GCS/Azure, and it requires no new infrastructure.

- **[Modal's GPU memory snapshotting](https://modal.com/blog/gpu-mem-snapshots).** Modal reports
  cutting a Qwen2.5-0.5B vLLM server's cold start from ~45 s to ~5 s by snapshotting
  post-initialisation GPU state and restoring it, with a
  [follow-up post on a Mistral model](https://modal.com/blog/mistral-3) showing the Ministral-3
  (3B) median falling ≈118 s → ≈12 s. **What it shows:** the ceiling on this problem is not
  "read the bytes faster" — it is "do not read them at all." It also shows what that costs: the
  snapshot is bound to a GPU model, driver and process layout, so it is a platform capability
  rather than a flag, which is why it belongs in the "know it exists" tier rather than in this
  lesson's practice. *(modal.com is blocked by this environment's egress proxy; these figures
  are Modal's own published claims, corroborated via web search rather than a direct fetch for
  this QA pass.)*

## Worked example

**Decompose the cold start for Llama-3.1-8B, compute a budget from first principles, then
measure and close the gap.** One GPU, roughly 45 minutes.

### Step 1 — the budget, before touching a machine

```
  MODEL:    Llama-3.1-8B-Instruct, bf16
  BYTES:    8.03e9 params x 2 B                              = 16.1 GB
  IMAGE:    vllm/vllm-openai:v0.27.1                        ≈ 9 GB compressed,
                                                             ≈ 22 GB extracted

  ── SCENARIO A: naive (HF Hub, no cache, cold node) ────────────────────────
    stage 0  node provision (Karpenter, p5)                    150 s
    stage 1  image pull @ 400 MB/s + decompress                 23 + 45 = 68 s
    stage 2  weights from HF Hub @ 200 MB/s   16.1e9/2.0e8  =   81 s
    stage 3  H2D, pageable Gen5 @ 20 GB/s     16.1/20       =    0.8 s
    stage 4  profile 7 s + compile MISS 70 s + capture 25 s =  102 s
    ──────────────────────────────────────────────────────────────────
    TOTAL                                                    ≈ 402 s  (6.7 min)

  ── SCENARIO B: warm node, cached image, node-local NVMe, warm compile ─────
    stage 0  pod schedule on existing node                       4 s
    stage 1  image cached                                        0 s
    stage 2  local NVMe Gen4 @ 5 GB/s         16.1/5        =    3.2 s
    stage 3  H2D, pinned Gen5 @ 50 GB/s       16.1/50       =    0.3 s
    stage 4  profile 7 s + compile HIT 5 s + capture 25 s   =   37 s
    ──────────────────────────────────────────────────────────────────
    TOTAL                                                    ≈  45 s

  ── SCENARIO C: B, plus a trimmed capture list and pinned KV size ──────────
    stage 4  profile SKIPPED (--kv-cache-memory-bytes)
             + compile HIT 5 s + capture 8 sizes ≈ 6 s      =   11 s
    ──────────────────────────────────────────────────────────────────
    TOTAL                                                    ≈  18 s

  ┌────────────────────────────────────────────────────────────────────────┐
  │ A → B is 9x, and NOT ONE BYTE of it came from a faster GPU.            │
  │ B → C is another 2.5x, and it is entirely stage 4 — which no amount    │
  │ of storage work would ever have touched.                              │
  │                                                                        │
  │ THE COMPOSITION MATTERS: in Scenario A, storage is 20 % of the total   │
  │ and node provisioning plus compilation is 63 %. A team that "fixed     │
  │ cold start" by buying faster storage would have moved 81 s of 402.     │
  └────────────────────────────────────────────────────────────────────────┘
```

**Write the budget down before measuring.** The prediction is what makes the measurement
informative.

### Step 2 — measure the whole thing, cold and warm

```bash
pip install "vllm==0.27.1" && vllm --version

vllm bench startup \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --num-iters-cold 3 --num-iters-warmup 1 --num-iters-warm 3 \
  --output-json startup.json
```

Representative output for a warm node with weights already on local NVMe:

```
=============== Cold Startup ===============
Startup time (s):            avg 71.42   p50 71.08   p90 73.91   p99 74.60
Compilation time (s):        avg 68.30   p50 68.11   p90 70.22   p99 70.88
=============== Warm Startup ===============
Startup time (s):            avg 42.15   p50 41.88   p90 43.02   p99 43.40
Compilation time (s):        avg  5.61   p50  5.44   p90  6.02   p99  6.19
```

**Read the delta, not the totals.** Cold minus warm is 29.3 s, and compilation alone accounts
for 62.7 s of it — meaning the compile cache is worth about a minute per pod, every pod. On a
40-replica rolling update that is 42 minutes of GPU time you are paying for compilation you
already did. **That single number is the business case for baking `VLLM_CACHE_ROOT` into the
image.**

### Step 3 — attribute the rest with two ablations

```bash
# (i) Stage 4 in isolation: dummy weights ⇒ no storage read, no H2D.
vllm serve meta-llama/Llama-3.1-8B-Instruct --load-format dummy \
  --port 8000 2>&1 | tee dummy.log

# (ii) Stages 2+3 in isolation: normal weights, no compile or capture.
vllm serve meta-llama/Llama-3.1-8B-Instruct --enforce-eager \
  --port 8000 2>&1 | tee eager.log

# (iii) The baseline.
vllm serve meta-llama/Llama-3.1-8B-Instruct --port 8000 2>&1 | tee normal.log

for f in dummy eager normal; do
  echo "== $f"
  grep -E 'Model loading took|Memory profiling takes|Available KV cache|GPU KV cache size|Graph capturing finished|init engine' "$f.log"
done
```

Representative extract from `normal.log`:

```
INFO ... Model loading took 14.99 GiB memory and 3.61 seconds
INFO ... Memory profiling takes 6.42 seconds. Total non KV cache memory: 16.51GiB; ...
INFO ... Available KV cache memory: 56.27 GiB
INFO ... GPU KV cache size: 460,800 tokens
INFO ... Graph capturing finished in 24 secs, took 0.94 GiB
INFO ... init engine (profile, create kv cache, warmup model) took 36.88 s (compilation: 5.44 s)
```

Assemble:

| Stage | Measurement | Value | Predicted (Scenario B) |
|---|---|---|---|
| 2 + 3 · weights | `Model loading took … 3.61 seconds` | 3.6 s | 3.5 s ✓ |
| 4a · profiling | `Memory profiling takes 6.42 seconds` | 6.4 s | 7 s ✓ |
| 4b · compile (warm) | `(compilation: 5.44 s)` | 5.4 s | 5 s ✓ |
| 4c · graph capture | `Graph capturing finished in 24 secs` | 24.0 s | 25 s ✓ |
| 4 · total | `init engine … took 36.88 s` | 36.9 s | 37 s ✓ |
| **Pod total, warm** | wall clock to `/health` 200 | **~44 s** | **45 s ✓** |

**Stage 2+3 is 8 % of a warm start. Stage 4 is 84 %.** For this configuration on this
storage tier, *storage is already solved* and every further storage optimisation is wasted
effort. That is the conclusion the decomposition exists to produce, and it is the opposite of
the folk answer.

### Step 4 — attack the actual bottleneck

Graph capture is 24 s of the 37 s. The default list is 51 sizes; your operating point (07.5)
is 96, and your traffic rarely runs below 8:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --kv-cache-memory-bytes 60420000000 \
  --compilation-config '{"cudagraph_capture_sizes": [1,2,4,8,16,32,64,96,128]}' \
  --port 8000 2>&1 | tee trimmed.log
```

```
INFO ... Model loading took 14.99 GiB memory and 3.58 seconds
INFO ... Initial free memory 79.11 GiB, reserved 56.27 GiB memory for KV Cache
         as specified by kv_cache_memory_bytes config and skipped memory profiling.
INFO ... Graph capturing finished in 6 secs, took 0.31 GiB
INFO ... init engine (profile, create kv cache, warmup model) took 13.02 s (compilation: 5.41 s)
```

| | Baseline | Trimmed | Change |
|---|---|---|---|
| Memory profiling | 6.4 s | 0 s (skipped) | −6.4 s |
| Graph capture | 24.0 s | 6.0 s | −18.0 s |
| Graph capture memory | 0.94 GiB | 0.31 GiB | −0.63 GiB (more KV) |
| `init engine` total | 36.9 s | 13.0 s | **−65 %** |
| **Pod total** | ~44 s | **~20 s** | **−55 %** |

Two things came free: the smaller capture set also returned 0.63 GiB to the KV pool, and
pinning `--kv-cache-memory-bytes` made the pool size identical across every node (07.4 §3).
Then verify the cost side: re-run 07.5's throughput sweep and confirm that the trimmed
capture list has not regressed decode throughput at your operating batch. **Sizes outside the
list fall back to the eager path**, so if your traffic drifts to a batch you dropped, you pay
for it in ITL — check `vllm:num_requests_running`'s distribution before trimming, not after.

### Step 5 — translate into the autoscaling budget

Feed the result back into 07.8:

```
  T_start (measured, warm node + cached image + local NVMe + warm compile
           + trimmed capture)                                =  20 s

  T_d = scrape 15 + staleness 15 + poll 15 + schedule 5
        + node 0 (warm) + image 0 (cached) + T_start 20
        + probe 10                                           =  80 s
                                                    (was 170 s in 07.8)

  ⇒ behavior.scaleUp.policies[].periodSeconds: 90    (was 180)
  ⇒ the loop can now correct itself more than twice as often,
    so overshoot per correction halves for the same 'value'.

  SCALE-TO-ZERO RE-CHECK (07.8's endpoint):
    8 wakes/day x 20 s x 40 affected requests
      = 320 requests/day, 116,800/yr → unchanged in COUNT,
        but each now waits 20 s instead of 170 s.
    Still ~5.8 % of a 2 M-request/yr endpoint's traffic breaching a
    500 ms SLO ⇒ STILL off the table for interactive traffic.
    ⇒ The verdict did not flip. What DID improve: recovery from a node
      failure went from 170 s to 20 s per replica, and a 40-replica
      rolling update went from ~50 min to ~14 min.
```

**Report the honest result.** An 8.5× cold-start improvement did not make scale-to-zero viable
for an interactive endpoint, because 20 s is still 40× a 500 ms SLO. It substantially improved
deploy velocity and incident recovery, and it halved the autoscaler's dead time. Those are the
wins; claiming the scale-to-zero one would be overselling.

## Practice

Rented GPU, roughly 60 minutes. Feeds **component 3** of the
[cost-per-token deliverable](../practice/cost-per-token/README.md) — the cold-start
breakdown, cached versus uncached.

### 1. Budget before measuring

Write the five-stage budget for your model, your storage tier and your image, using §1's
bandwidth table. Produce two scenarios: worst case (cold node, uncached image, network
weights, cold compile cache) and best case (warm node, cached image, local weights, warm
cache).

**Acceptance:** two five-row budgets with the arithmetic shown, and a prediction of which
stage dominates each.

### 2. Measure cold versus warm with the in-tree harness

```bash
vllm bench startup --model <your-model> \
  --num-iters-cold 3 --num-iters-warmup 1 --num-iters-warm 3 \
  --output-json startup.json
```

**Acceptance:** the cold and warm `Startup time` and `Compilation time` averages and p90s,
plus the cold-minus-warm delta expressed as "GPU-seconds per pod that a persisted compile
cache would save," and the same figure multiplied by your fleet size for a rolling update.

### 3. Attribute the stages with the two ablations

Boot three times — normal, `--load-format dummy`, `--enforce-eager` — and extract
`Model loading took`, `Memory profiling takes`, `Graph capturing finished` and `init engine`
from each.

**Acceptance:** a stage-attribution table with each stage's seconds and its **percentage of
the total**, plus one sentence naming the dominant stage. If it is not the stage you predicted
in #1, say what your budget got wrong.

### 4. Move stage 2 by two tiers, cold

Load the same model from two different storage tiers — at minimum "network/remote" versus
"node-local" — dropping caches between runs:

```bash
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches   # ← or you measure the page cache
```

If you have object storage available, add a third run with
`--load-format runai_streamer --model-loader-extra-config '{"concurrency": 16}'`.

**Acceptance:** a table of `Model loading took` seconds per tier, the implied achieved
bandwidth (`bytes ÷ seconds`), and that bandwidth compared against §1's expected range for
that tier. Explain any figure more than 2× off.

### 5. Quantify the page cache

Run the same local-NVMe load twice without dropping caches. Record both.

**Acceptance:** the two load times and the ratio, with a one-line statement of what this
implies for the *second* replica scheduled onto a node and for your DaemonSet's
`cat > /dev/null` warm-up step.

### 6. Attack whichever stage dominates

If stage 4 dominates: persist `VLLM_CACHE_ROOT`, set `VLLM_FORCE_AOT_LOAD=1`, trim
`cudagraph_capture_sizes` to your operating batch, and pin `--kv-cache-memory-bytes`. If
stage 2 dominates: move to a node-local cache or a streaming loader. If stage 1 dominates:
pre-pull the image.

**Acceptance:** before/after totals, the specific change made, and — for a trimmed capture
list — a confirmation from a short 07.5-style throughput run that decode throughput at your
operating batch has **not** regressed.

### 7. Feed it back into the autoscaler

Recompute `T_d` from 07.8 with your new `T_start`, and state the `scaleUp.periodSeconds` it
implies. Re-price the scale-to-zero decision for one endpoint.

**Acceptance:** the new `T_d` arithmetic, the new rate-limit value, and an explicit statement
of whether the scale-to-zero verdict flipped — **including "no" as a valid, well-supported
answer**, with what `T_start` would have to reach for it to change.

**Overall acceptance:** a cold-start breakdown by stage, cached versus uncached, from your own
GPU run; the achieved bandwidth per storage tier with the arithmetic; one bottleneck attacked
with a before/after; and the revised autoscaling dead time — committed to the deliverable.
This is the measured number the [checkpoint](../checkpoint.md) asks you to defend.

## Common pitfalls

- **Reaching for a bigger GPU to fix a slow cold start.** *Mechanism:* stages 1–3 are
  I/O-bound and stage 4 is bounded by CPU compilation and GPU graph capture, neither of which
  scales with tensor throughput. A faster card loads weights no faster if the bottleneck is a
  200 MB/s network read. The fix is always architectural.

- **Measuring cold start without dropping the page cache.** *Mechanism:* after one load the
  checkpoint sits in RAM, and the second read runs at 10–25 GB/s instead of your storage
  tier's rate. You measure a number you will never see in production, then discover the truth
  during an incident. `sync && echo 3 > /proc/sys/vm/drop_caches` between runs.

- **Baking weights into the container image.** *Mechanism:* a 20–160 GB image moves the
  weight-download cost into stage 1, where layer decompression is largely single-threaded and
  cannot be bought out with bandwidth. It also re-pulls per node, cannot share a copy across
  models, and requires an image rebuild to change a checkpoint.

- **Using `mmap` (the default) on a network filesystem.** *Mechanism:* every tensor access
  becomes a page fault, which on NFS is a network round trip; the loader's access pattern is
  not sequential in file order, so you get many small latency-bound reads and a fraction of the
  filesystem's throughput. Use `--safetensors-load-strategy eager` (or `prefetch`). vLLM
  auto-detects NFS specifically, but many CSI-mounted network volumes do not present as NFS.

- **Running TP > 1 with an unsharded checkpoint on a RAM-constrained node.** *Mechanism:* each
  worker reads the whole checkpoint, so aggregate bytes scale with TP. The page cache hides
  this while node RAM exceeds the checkpoint size, then stops hiding it. Convert to a sharded
  checkpoint and use `--load-format sharded_state` or `runai_streamer_sharded`.

- **Serving from object storage with the default loader.** *Mechanism:* object-storage
  throughput is per-connection. One stream is 50–200 MB/s regardless of your NIC; sixteen are
  1–5 GB/s on the same NIC. Use `--load-format runai_streamer` with an explicit `concurrency`.

- **Assuming a persisted compile cache is being used.** *Mechanism:* any change to the model
  (including a revision bump), engine config, relevant `VLLM_*` variables, torch build or **GPU
  model** invalidates it, and vLLM silently recompiles — no error, no obvious log line. On a
  heterogeneous fleet an image whose cache was baked on H100s misses on every A100 node. Set
  `VLLM_FORCE_AOT_LOAD=1` so a miss fails loudly.

- **Quoting safetensors' 76× speedup for GPU loading.** *Mechanism:* the 76.6× is the CPU
  path, where `mmap` eliminates allocation and deserialisation entirely. The published GPU
  figure is 2.1×, because the bytes still cross PCIe. Different stages, different numbers.

- **Setting `readinessProbe.initialDelaySeconds` from a warm boot.** *Mechanism:* the engine
  serves `/health` only after stage 4 completes. Without a `startupProbe` with a generous
  `failureThreshold`, the liveness probe restarts a pod that is still loading — forever — and
  the symptom presents as a crashloop rather than a probe misconfiguration.

- **Trimming `cudagraph_capture_sizes` without checking the traffic distribution.**
  *Mechanism:* batch sizes outside the list fall back to the eager path, losing kernel-launch
  amortisation. Check the distribution of `vllm:num_requests_running` first, and re-run a
  throughput measurement after trimming.

## Self-check

**(a) Decompose a cold start into its stages and say what bounds each.**

**Answer:** Five stages. **(0) Schedule + GPU bind** — 2–10 s on a warm node, 60–300 s if a
node must be provisioned; bounded by the cloud API, kubelet join and device-plugin
registration. **(1) Image pull** — 0 s cached, 1–6 min otherwise; bounded by network *and*
largely single-threaded layer decompression, which is often the larger half. **(2) Storage
read** — bounded by the storage tier: 0.1–0.3 GB/s from HF Hub, 0.3–1 GB/s from a network
filesystem, 3–14 GB/s from local NVMe, 10–25 GB/s from a warm page cache. **(3) Host-to-device
copy** — bounded by PCIe: 31.5 GB/s theoretical on Gen4 x16 and 63 GB/s on Gen5, achieved
22–26 and 45–55 GB/s respectively **with pinned memory**, roughly half that from pageable/mmap
buffers; near-free on NVLink-C2C platforms. **(4) Profile, compile, capture** — 5–10 s
profiling, 2–8 s compile on a cache hit versus 30–120 s on a miss, 15–40 s of CUDA-graph
capture for the default 51 sizes; bounded by CPU and GPU with **no I/O at all**. The
operational point: stages 1–3 are I/O-bound and stage 4 is not, so no storage work touches
stage 4, and once storage is fast stage 4 becomes a 20–60 s floor.

**(b) Why is safetensors faster to load than a PyTorch `.bin`, and why is the CPU speedup much
larger than the GPU speedup?**

**Answer:** safetensors is 8 bytes of little-endian header length, then a JSON header mapping
each tensor name to `{dtype, shape, data_offsets:[begin,end)}`, then a fully-indexed raw byte
buffer. Because the header gives every tensor's exact byte range *before* any data is read, a
loader can `mmap` the file and hand out zero-copy views, issue parallel ranged reads, or fetch
only the tensors one TP rank needs. A pickle `.bin` is an instruction stream for a stack
machine: you cannot know a tensor's offset without executing the stream up to that point, so
you cannot mmap, parallelise or subset it — and the same arbitrary-deserialisation flexibility
is what makes pickle a remote-code-execution risk. Fast and safe are the same property.
The CPU speedup is **76.6×** (0.004 s vs 0.307 s on GPT-2) because `mmap` removes the
allocation and the deserialisation entirely. The GPU speedup is only **2.1×** (0.165 s vs
0.354 s on a T4) because the bytes still have to cross PCIe — the library skips the CPU
allocation and calls `cudaMemcpy` from the mapping, but it cannot skip the wire. That gap is
the clearest demonstration that stage 2 and stage 3 are separate problems.

**(c) Compute a cold-start budget for a 70B FP8 model loading from a 1 GB/s network
filesystem over PCIe Gen5, with a cold compile cache, and say which stage to attack.**

**Answer:** Weights are `70.6e9 × 1 B = 70.6 GB`. Stage 2: `70.6 / 1.0 = 70.6 s` — and if the
loader is using `mmap` on NFS, realistically 3–5× worse, so 200–350 s. Stage 3, pinned Gen5 at
~50 GB/s: `70.6 / 50 = 1.4 s`; pageable, ~20 GB/s: 3.5 s. Stage 4 with a cold compile cache:
~7 s profiling + 70–120 s compile + 25–40 s capture ≈ 100–170 s. Stage 1, uncached image:
60–360 s. Total, naive: roughly **330–880 s (5.5–15 min)**. **Attack in order of size, not of
familiarity:** first the `mmap`-on-NFS pathology (`--safetensors-load-strategy eager`), which
recovers 130–280 s for a one-line change; then the compile cache (persist `VLLM_CACHE_ROOT`,
set `VLLM_FORCE_AOT_LOAD=1`), worth 65–115 s per pod; then the image (pre-pull DaemonSet or
in-region mirror); then move weights to node-local NVMe, which takes stage 2 from 70 s to
~10 s. **Stage 3 is 1.4 s and is never worth attacking here.** If TP > 1, also shard the
checkpoint, because otherwise every worker reads all 70.6 GB and stage 2 multiplies by TP.

**(d) Your cold start is 44 s and you want it under 20 s. What do you need to know before
choosing a lever?**

**Answer:** The per-stage attribution, because the answer differs completely by stage. Get it
from three boots: normal, `--load-format dummy` (random weights, no storage read or H2D — the
difference is stages 2+3), and `--enforce-eager` (no compile or capture — the difference is
stage 4's compile+capture). Then read the log lines directly: `Model loading took … N seconds`
is stages 2+3, `Memory profiling takes N seconds` is 4a, `(compilation: N s)` inside the
`init engine` line is 4b, and `Graph capturing finished in N secs` is 4c. In the worked
example the split was 3.6 s of weights against 36.9 s of `init engine` — **stage 4 was 84 %
of a warm start**, so every storage optimisation would have been wasted effort. The levers that
did work were pinning `--kv-cache-memory-bytes` (skips the 6.4 s profiling pass entirely) and
trimming `cudagraph_capture_sizes` from 51 sizes to 9 (24 s → 6 s), together taking the pod
from ~44 s to ~20 s and returning 0.63 GiB to the KV pool as a bonus. Verify afterwards that
decode throughput at your operating batch has not regressed, since sizes outside the capture
list fall back to eager execution.

**(e) Why does adding GPUs make model loading slower, and what fixes it?**

**Answer:** With tensor parallelism, **each worker process reads the whole checkpoint and
slices out its own shard** — vLLM's documentation states this directly. So a 70 GB checkpoint
at TP=4 reads 280 GB aggregate, and on a per-node-bandwidth-limited filesystem that is 4× the
single-GPU load time. The page cache masks this when node RAM comfortably exceeds the
checkpoint (workers 2–4 are served from memory at 10–25 GB/s), which is why the problem is
often invisible in testing and appears in production when the model grows or the node is
busier. The fix is a pre-sharded checkpoint — convert once with
`examples/features/sharded_state/save_sharded_state_offline.py`, producing
`model-rank-{rank}-part-{part}.safetensors`, then serve with `--load-format sharded_state`
(local) or `--load-format runai_streamer_sharded` (object storage). Each worker then reads
only its own shard, the reads run in parallel across file handles, and load time becomes
**independent of TP size**. Measure it cold, with caches dropped, or you will measure the page
cache and conclude there was never a problem.

**(f) Does making cold start 8× faster make scale-to-zero viable for an interactive endpoint?**

**Answer:** Usually not, and being able to say so is the point. Going from 170 s to 20 s is a
large operational win — the autoscaler's dead time roughly halves (so 07.8's
`scaleUp.periodSeconds` drops from 180 to ~90), recovery from a lost replica goes from ~3 min
to ~20 s, and a 40-replica rolling update goes from ~50 min to ~14 min. But the scale-to-zero
verdict is set by the SLO, not by the improvement ratio: 20 s is still **40×** a 500 ms TTFT
SLO, so the same ~117k requests/year caught behind wake-ups still breach it, still consuming
several times the error budget. For the verdict to flip, `T_start` has to approach the SLO
itself — which is snapshot territory (Modal reports ~10 s for a vLLM server, still 20× a
500 ms budget) or vLLM **sleep mode**, where the process, CUDA context and compiled graphs stay
alive and waking is a PCIe copy of the weights. Sleep mode's caveat is that the **GPU remains
allocated**, so it converts idle HBM into a shareable resource rather than stopping the meter —
it pays only when something else can use the card. The honest reporting is: name the wins you
got, and do not claim the one you did not.

## Connections & what's next

This lesson replaced 07.8's placeholder cold start with a measured, five-stage budget, and in
doing so gave you the second input the autoscaler needed — `T_start`, which sets `T_d`, which
sets the anti-oscillation rate limit. It also showed why 07.7's quantization pays twice: half
the weight bytes is half of stages 2 and 3. And it exposed a general fact worth carrying
beyond this module: **for a service whose data plane is a GPU, the control-plane costs — image
pull, weight load, compilation, graph capture — routinely dominate the operations that gate
availability**, and they are invisible in every throughput benchmark you will ever run.

**Next: [07.10 — Multi-model serving with LoRA](10-multi-model-lora.md)** attacks the same
economics from the other side. This lesson made one model's wake-up cheap; the next makes one
loaded base model serve dozens of logical models at once, so the wake-up you just optimised is
amortised across many tenants instead of one — the last multiplier on the cost-per-token
deliverable.

## References & further reading

**Primary sources (vLLM v0.27.1, cross-checked against `main` @ `c1e4387`, 2026-08-17; safetensors format spec and reference implementation)**

1. **`vllm/config/load.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/load.py — the authoritative `load_format` list with each option's description (`auto`, `safetensors`, `pt`, `npcache`, `dummy`, `tensorizer`, `runai_streamer`, `runai_streamer_sharded`, `instanttensor`, `sharded_state`, `bitsandbytes`, `mistral`, `modelexpress`); `safetensors_load_strategy` (`lazy` / `eager` / `prefetch` / `torchao`) **including the automatic NFS-detection behaviour and the 90 %-of-RAM condition**; `DEFAULT_SAFETENSORS_PREFETCH_NUM_THREADS = 8` and `DEFAULT_SAFETENSORS_PREFETCH_BLOCK_SIZE = 16 MiB`; and `ignore_patterns` defaulting to `["original/**/*"]`.
2. **`docs/models/extensions/runai_model_streamer.md`** — https://github.com/vllm-project/vllm/blob/main/docs/models/extensions/runai_model_streamer.md — installation (`pip install vllm[runai]`), the `s3://` / `gs://` / `az://` URI support, the S3-compatible environment variables, and the three tunables quoted in §6: `concurrency` ("the number of client instances the host is opening to the S3 server"), `memory_limit`, and `distributed`. Also the sharded loader's `model-rank-{rank}-part-{part}.safetensors` pattern and its `pattern` override.
3. **`docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — the "Faster Startup" section naming the three mechanisms used in §8: compile-cache reuse under `VLLM_CACHE_ROOT` (default `~/.cache/vllm`) with `VLLM_FORCE_AOT_LOAD=1` to fail loudly on a miss and the full list of what invalidates it; `--kv-cache-memory` to skip the profiling measurement, with its explicit warning that a conservative value caps concurrency; and `--enforce-eager` as the fastest-startup option. Also the `-O0`–`-O3` optimization levels.
4. **`docs/configuration/conserving_memory.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/conserving_memory.md — the TP loading note quoted in §7: each process reads the whole model, disk reading time is proportional to TP size, and a sharded checkpoint makes load time constant regardless of TP. Also `cudagraph_capture_sizes` trimming as a memory lever.
5. **`vllm/config/compilation.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/compilation.py — the default capture-size generation `[1,2,4] + range(8,256,8) + range(256, max+1, 16)` (51 sizes) with `max_cudagraph_capture_size` capped at 512 (1024 on data-centre Blackwell), and `cudagraph_num_of_warmups`.
6. **`vllm/benchmarks/startup.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/benchmarks/startup.py — the cold/warm harness used in §12: cold iterations run with **temporary cache directories** so nothing is reused, warm iterations use the cached compilation and model info; metrics `total_startup_time`, `compilation_time`, `encoder_compilation_time`; percentiles at 10/25/50/75/90/99; flags `--num-iters-cold`, `--num-iters-warmup`, `--num-iters-warm`, `--output-json`.
7. **`vllm/v1/engine/core.py` and `vllm/v1/worker/gpu_model_runner.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core.py — the exact log strings this lesson greps: `"init engine (profile, create kv cache, warmup model) took %.2f s (compilation: %.2f s)"`, `"Model loading took %s GiB memory and %.6f seconds"`, `"Graph capturing finished in %.0f secs, took %.2f GiB"`.
8. **`vllm/v1/worker/gpu_worker.py` and `vllm/utils/mem_utils.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_worker.py — the `Memory profiling takes N seconds` line and its breakdown, the `kv_cache_memory_bytes` path that skips profiling with its "does not respect gpu_memory_utilization" warning, and `sleep()` / `wake_up()` for §11's in-process alternative.
9. **`docs/benchmarking/sweeps.md`** — https://github.com/vllm-project/vllm/blob/main/docs/benchmarking/sweeps.md — `vllm bench sweep startup`, for comparing startup across engine configurations (`--startup-cmd`, `--serve-params`, `--startup-params`, `--strict-params`).

10. **`huggingface/safetensors` — README §Format** — https://github.com/huggingface/safetensors — the byte layout quoted in §3: 8-byte little-endian `N`, an N-byte JSON header (must begin with `{`, may be space-padded), then the byte buffer; the `{"dtype","shape","data_offsets":[BEGIN,END)}` per-tensor record with offsets **relative to the buffer, not the file**; the `__metadata__` string-to-string map; and the constraints — no duplicate keys, the buffer must be **entirely indexed with no holes** (an explicit anti-polyglot measure), little-endian, C/row-major.
11. **`safetensors/src/tensor.rs`** — https://github.com/huggingface/safetensors/blob/main/safetensors/src/tensor.rs — `MAX_HEADER_SIZE = 100_000_000` and the three sites that enforce it. Relevant if you generate checkpoints with very many small tensors.
12. **`docs/source/speed.mdx`** — https://github.com/huggingface/safetensors/blob/main/docs/source/speed.mdx — the benchmark numbers used in §3 and pitfall 8: GPT-2 CPU load 0.004 s (safetensors) versus 0.307 s (`torch.load`) — **76.6×** — on an Intel Xeon @ 2.00GHz; and GPU load on a Tesla T4, 0.165 s versus 0.354 s — **2.1×** — with the doc's own explanation that the GPU path works by memory-mapping the file, creating an empty tensor and calling `cudaMemcpy` directly.

**Real-world engineering blogs**

13. **Modal — "GPU Memory Snapshots: Supercharging sub-second startup"** — <https://modal.com/blog/gpu-mem-snapshots> — reports cutting a Qwen2.5-0.5B vLLM server's cold start from ~45 s to ~5 s by snapshotting post-initialisation GPU state; a companion post, **"Modal + Mistral 3: 10x faster cold starts with GPU snapshotting"** (<https://modal.com/blog/mistral-3>), reports the Ministral-3 (3B) median figure quoted in this lesson's Real-world use cases (~118 s → ~12 s). **What it shows:** the ceiling on cold start is not "read the bytes faster," it is "do not read them at all" — the source for the claim this lesson's use-case section previously carried without a citation. *(modal.com is blocked by this environment's egress proxy; both figures are Modal's own published claims, corroborated via web search rather than a direct fetch for this QA pass.)*

**Deeper dives**

14. **07.4 §1 and §9 — vLLM in production** — [04-vllm-in-production.md](04-vllm-in-production.md) — the startup timeline this lesson decomposes, the memory-profiling mechanism behind stage 4a, `--kv-cache-memory-bytes` for fleet-deterministic sizing, and the CUDA-graph and compile-cache flags. Read together, 07.4 §1 is the *what happens* and this lesson is the *how long and why*.
