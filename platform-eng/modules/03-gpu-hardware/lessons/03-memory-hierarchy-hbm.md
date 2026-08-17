---
lesson: "03.3"
title: "Memory hierarchy & HBM: what fits and how fast"
module: "03"
concept: "Memory hierarchy & HBM"
status: not-started
est_time: "6h"
prev: "02-compute-vs-memory-bound-roofline.md"
next: "04-decode-bandwidth-batching.md"
artifacts: []
sources: 13
---

# 03.3 · Memory hierarchy & HBM: what fits and how fast

> **Concept.** HBM capacity decides what *fits*; HBM bandwidth decides how *fast* you decode. Two independent buying levers, one spec sheet.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 03.2 gave you the roofline: every workload sits under the compute roof or the memory roof, decided by arithmetic intensity against a ridge point you can compute (~295 FLOP/byte dense BF16 on H100). It treated the memory roof as a single number — 3.35 TB/s — and moved on. That number is a summary of a five-level hierarchy, and the summary hides everything you need for capacity planning.

This lesson opens it up. Where does that bandwidth physically come from, and why is it the number it is? What sits above HBM in the hierarchy, how much faster is each rung, and why can none of it hold a model? What competes for HBM capacity, in bytes, for a specific model at a specific batch size and context length? And what happens when you run out — because unlike a CPU, there is no graceful paging path.

By the end you can answer the two questions that dominate every GPU capacity conversation — *"can we serve model X on SKU Y?"* and *"how many tokens/sec will we get?"* — on a napkin, before renting anything. Lesson 03.4 then takes the bandwidth ceiling built here and turns it into a full batching and serving-architecture model.

## Why this matters

Get the capacity question wrong and you discover it in production as an out-of-memory crash mid-deployment, or as a serving process that silently caps its batch size at 4 and burns 90% of a $3/hr card doing nothing. Get the bandwidth question wrong and you either over-provision GPUs you did not need or miss an SLO. Both are line items someone reviews.

It is also the concrete reason GPU SKUs sharing a compute die cost meaningfully different amounts. An H100 (80 GB HBM3, 3.35 TB/s) and an H200 (141 GB HBM3e, 4.8 TB/s) run **identical FLOPS** — same GH100 die, same 132 SMs, same 528 tensor cores. If GPUs were priced on compute alone they would cost the same. They do not, because for inference — where most GPU-fleet dollars go — the two numbers this lesson teaches you to compute are the *entire* value proposition of the more expensive card.

And there is a hard asymmetry that makes the capacity question unforgiving. A CPU that runs out of RAM pages to disk and gets slow. A GPU that runs out of HBM has no equivalent: the nearest tier is host DRAM across PCIe Gen5 x16 at roughly **64 GB/s**, against HBM's 3,350 GB/s — **52× slower**. Spilling weights to host memory does not degrade performance, it destroys it. HBM capacity is a wall, not a gradient.

## What's new here (calibration)

Per the module README's calibration you are not becoming a kernel developer, and you already know how memory hierarchies work in general-purpose computing. This lesson does not re-teach cache theory, register allocation, or how a compiler schedules loads. What it adds:

- **The full GPU hierarchy with real capacity, bandwidth and latency at every rung**, several of them *derived* from the architecture rather than quoted, so you can reconstruct them for a SKU you have never seen.
- **What HBM physically is** — stacks, through-silicon vias, the interposer, channels and pseudo-channels, pin rates — enough that "why can't they just make it bigger/faster" has a mechanical answer, and enough that you can check a vendor's bandwidth claim against JEDEC's pin-rate limits.
- **The two independent buying levers named precisely** — capacity (what fits) and bandwidth (how fast) — which move separately across SKUs, and whose conflation is the most common GPU-purchasing mistake.
- **A KV-cache formula derived from the attention mechanism**, not asserted, with the GQA/MQA substitution explained by what it physically changes, and carried through to the batch size at which HBM is exhausted.
- **The compounding compression levers** and how far real production systems push them.

## Core concepts

### 1. The hierarchy, rung by rung

Every rung is roughly an order of magnitude larger and an order of magnitude slower than the one above. Numbers below are for **H100 SXM5** unless noted; latencies are measured cycle counts from independent microbenchmarking (Luo et al., arXiv:2402.13499, reporting A100-class figures — the tier *ratios* hold across Ampere/Hopper: L2 ≈ 6.5× L1, global ≈ 1.9× L2).

```
   THE GPU MEMORY HIERARCHY — H100 SXM5, one card
   ══════════════════════════════════════════════════════════════════════════

   TIER            SCOPE        CAPACITY          BANDWIDTH        LATENCY
   ────            ─────        ────────          ─────────        ───────
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ REGISTERS      per-thread   256 KB / SM       ~400 TB/s        ~0 cycles │
 │                             33.8 MB total     (derived, §2)    (pipelined)│
 │  64K × 32-bit per SM; max 255 registers/thread. Compiler-allocated.       │
 └───────────────────────────┬──────────────────────────────────────────────┘
                             │  spill / explicit load
 ┌───────────────────────────▼──────────────────────────────────────────────┐
 │ SHARED MEM     per-block    up to 228 KB/SM   ~33 TB/s         ~29 cycles│
 │  / L1 CACHE    per-SM       (of 256 KB        (derived, §2)              │
 │                             unified L1+shared)                 L1 ~38 cy │
 │  Software-managed scratchpad. 32 banks × 4 B/clk. This is the "SRAM"     │
 │  FlashAttention keeps the S×S score matrix in.                           │
 └───────────────────────────┬──────────────────────────────────────────────┘
                             │
 ┌───────────────────────────▼──────────────────────────────────────────────┐
 │ L2 CACHE       whole GPU    50 MB             several TB/s     ~262 cycles│
 │  Shared by all 132 SMs; 2 partitions on Hopper (4 on Blackwell).         │
 │  Big enough for one 4096² BF16 tile. Nowhere near a model.               │
 └───────────────────────────┬──────────────────────────────────────────────┘
                             │
 ┌───────────────────────────▼──────────────────────────────────────────────┐
 │ ★ HBM3         whole GPU    80 GB             3.35 TB/s        ~466 cycles│
 │   (global memory)           (141 on H200)     (4.8 on H200)    ≈ 235 ns  │
 │  THE ONLY TIER YOU BUDGET IN. Weights + KV cache + activations live here.│
 └───────────────────────────┬──────────────────────────────────────────────┘
                             │
        ┌────────────────────┴────────────────────┐
        │                                         │
 ┌──────▼─────────────────────┐   ┌───────────────▼──────────────────────────┐
 │ PEER GPU over NVLink 4     │   │ HOST DRAM over PCIe Gen5 x16             │
 │ 900 GB/s aggregate         │   │ ~64 GB/s per direction                   │
 │ = 3.7× slower than local   │   │ = 52× SLOWER THAN LOCAL HBM              │
 │ (tensor parallelism lives  │   │ (offload = emergency, never a plan)      │
 │  here — see 02b)           │   │                                          │
 └────────────────────────────┘   └──────────────────────────────────────────┘

   SIZE RATIO, top to bottom:  256 KB → 228 KB → 50 MB → 80 GB → TB of host
   SPEED RATIO, top to bottom: 400 TB/s → 33 → ~7 → 3.35 → 0.064 TB/s
```

Two things to internalise:

1. **Each rung down is roughly 10× bigger and 10× slower.** Kernel writing is the art of keeping hot data on the top rungs. *Platform* engineering only needs the consequence: for LLM inference, the working set — tens of GB of weights plus a growing KV cache — always lives in HBM. There is no arrangement of a 50 MB L2 that holds a 70B model. HBM is where the budget conversation happens.
2. **Only HBM is measured in "GB you can spend."** Registers, shared memory and L2 are fixed, tiny, and not allocatable for model state. When someone says "the model doesn't fit," they mean HBM, full stop.

### 2. Deriving the on-chip bandwidths (so you can do it for any SKU)

The top two rungs' bandwidths are rarely on a datasheet, but they follow from the architecture, and being able to derive them is what lets you sanity-check a claim.

**Shared memory.** Each SM's shared memory has **32 banks, each 4 bytes wide, serving one access per clock** (the classic CUDA banking model — this is why bank conflicts matter). So per SM per clock: `32 × 4 B = 128 B`. Across the chip:

```
A100:  128 B/clk × 108 SMs × 1.41 GHz = 19.5 TB/s
H100:  128 B/clk × 132 SMs × 1.98 GHz = 33.4 TB/s
```

The A100 figure is worth pausing on: the FlashAttention paper (Dao et al., 2022) states the A100 has "192 KB of on-chip SRAM per each of 108 SMs with bandwidth estimated around 19 TB/s." **19.5 vs "around 19" — the derivation and the published estimate agree.** That is a good sign your model of the hardware is right, and it means you can produce the equivalent number for a part nobody has published one for.

**Registers.** To sustain 128 FP32 FMA lanes per SM you must supply three 4-byte operands and retire one per lane per clock. `128 lanes × 3 reads × 4 B = 1,536 B/clk/SM` of read bandwidth alone:

```
H100:  1,536 B/clk × 132 SMs × 1.98 GHz ≈ 401 TB/s   (read side, derived)
```

Roughly **120× HBM bandwidth**, which is precisely why the whole game is data reuse: every byte you can serve from registers instead of HBM is served 120× faster.

**L2.** Harder to derive cleanly and not published; independent microbenchmarks put Hopper's L2 in the several-TB/s range, i.e. a low single-digit multiple of HBM. Treat it as "meaningfully faster than HBM, not remotely as fast as shared memory," and do not build a budget on it.

**The compulsory conclusion:** the FLOP:byte ratios of §03.2 are ratios against *HBM*. Every optimisation that matters — tiling, fusion, FlashAttention, operator scheduling — is an attempt to move traffic from the 3.35 TB/s rung to the 33 TB/s or 400 TB/s rungs.

### 3. What HBM physically is, and why capacity and bandwidth are separate knobs

**HBM (High Bandwidth Memory)** is DRAM dies stacked vertically, wired through the stack with **through-silicon vias (TSVs)**, and mounted on the *same package* as the GPU die, connected by a silicon **interposer** carrying an extremely wide parallel bus. Contrast with a CPU's DIMMs: those sit centimetres away on the motherboard, connected by a 64-bit channel per DIMM at high clock rates.

```
   HBM PACKAGE CROSS-SECTION (schematic, one of several stacks)
   ═════════════════════════════════════════════════════════════════════

              ┌──────────┐  ┌──────────┐  ┌──────────┐   ← 8–12 DRAM dies,
              │ DRAM die │  │ DRAM die │  │ DRAM die │     stacked, joined
              ├──────────┤  ├──────────┤  ├──────────┤     by TSVs (vertical
              │ DRAM die │  │ DRAM die │  │ DRAM die │     copper vias
              ├──────────┤  ├──────────┤  ├──────────┤     through the silicon)
              │ DRAM die │  │ DRAM die │  │ DRAM die │
              ├──────────┤  ├──────────┤  ├──────────┤   CAPACITY KNOB:
              │ base die │  │ base die │  │ base die │   dies per stack ×
              └────┬─────┘  └────┬─────┘  └────┬─────┘   density per die ×
                   │             │             │         number of stacks
   ════════════════╪═════════════╪═════════════╪══════════════════════════
     SILICON       │  1024 bits  │  1024 bits  │  1024 bits   ← BANDWIDTH KNOB:
     INTERPOSER    │  per stack  │  per stack  │              bus width ×
   ════════════════╪═════════════╪═════════════╪═══════════   pin data rate
                   │             │             │
              ┌────┴─────────────┴─────────────┴────┐
              │        GPU DIE (GH100 / GB100)       │
              │   memory controllers, L2, 132 SMs    │
              └──────────────────────────────────────┘
                          PACKAGE SUBSTRATE

   PER-STACK LOGICAL ORGANISATION (HBM3, JEDEC JESD238)
   ────────────────────────────────────────────────────
     16 independent channels × 64 bits           = 1024-bit interface
     each channel splits into 2 pseudo-channels  = 32 pseudo-channels
     up to 6.4 Gb/s per pin                      → 6.4 × 1024 / 8
                                                 = 819 GB/s per stack (JEDEC max)
     HBM3E (JESD238B) raises the pin rate to up to 9.6 Gb/s
                                                 → up to ~1.23 TB/s per stack
```

Now read real products off that structure. **Bandwidth per GPU = stacks × 1024 bits × pin rate ÷ 8**, and every shipping part checks out:

| GPU | HBM gen | Stacks (enabled) | Capacity | Bus width | Implied pin rate | Bandwidth |
|---|---|---|---|---|---|---|
| A100 SXM 80GB | HBM2e | 5 × 16 GB | 80 GB | 5,120 bit | **3.19 Gb/s** (HBM2e max ≈3.2) | 2.039 TB/s |
| H100 SXM5 | HBM3 | 5 × 16 GB | 80 GB | 5,120 bit | **5.23 Gb/s** (HBM3 max 6.4) | 3.35 TB/s |
| H200 SXM | HBM3e | 6 × 24 GB | 144 GB physical, **141 GB usable** | 6,144 bit | **6.25 Gb/s** | 4.8 TB/s |
| B200 SXM | HBM3e | 8 | 192 GB physical, **180 GB usable** | 8,192 bit | **7.52 Gb/s** | 7.7 TB/s |

Check one yourself: H200 is `6,144 bits × 6.25e9 /s ÷ 8 bits/byte = 4.8e12 B/s`. ✓ Check H100: `5,120 × 5.23e9 ÷ 8 = 3.35e12`. ✓

**This table is the mechanism behind every "why does the newer card cost more" question.** Look at what changed H100 → H200: the GH100 compute die is *byte-identical*. NVIDIA added one physical stack (5 → 6), moved to a denser and faster HBM generation (16 GB @ 5.23 Gb/s → 24 GB @ 6.25 Gb/s per stack), and shipped it as a different SKU. Capacity went up 76%, bandwidth 43%, compute 0%.

Three consequences worth carrying:

- **Capacity and bandwidth are genuinely independent knobs** a vendor turns by changing the memory package, not the compute die. You cannot infer one from the other. Check both.
- **Both knobs have physical limits**: dies per stack (thermal and TSV yield), stacks per package (interposer area around the die's perimeter), and pin rate (JEDEC signalling). This is why HBM capacity has grown far more slowly than model sizes — it is a packaging problem, not a process-node problem.
- **The "usable vs physical" gap is real and you must budget against usable.** H200 ships 144 GB of DRAM and exposes 141 GB; B200 ships 192 GB and exposes 180 GB in HGX/DGX configurations. Some materials quote the physical number. Plan against the exposed one.

### 4. The generational memory snapshot

| GPU | HBM gen | Capacity | Bandwidth | Dense BF16 | FP8 tensor HW? |
|---|---|---|---|---|---|
| A100 SXM 80GB | HBM2e | 80 GB | 2.039 TB/s | 312 TFLOP/s | **No** — FP8 debuts in Hopper; A100 has tensor-core-accelerated INT8 (624 TOP/s dense) but not FP8 |
| H100 SXM5 | HBM3 | 80 GB | 3.35 TB/s | 989 TFLOP/s | Yes — 4th-gen tensor cores + Transformer Engine |
| H200 SXM | HBM3e | **141 GB** | **4.8 TB/s** | 989 TFLOP/s — identical to H100 | Yes |
| B200 SXM | HBM3e | 180 GB | 7.7 TB/s | 2,250 TFLOP/s | Yes, plus FP4 (9,000 TFLOP/s dense) |

The A100 row matters for a capacity reason, not a precision one: on Ampere, "quantise to FP8" is not a hardware-accelerated path at all. You would fall back to INT8, which *is* tensor-core accelerated on Ampere, for a comparable capacity and throughput win. **Which quantised formats run at full speed is gated by which silicon generation you rent**, not by a software flag. Lesson 03.5 goes deep on precision; the takeaway here is narrow and purely about capacity planning.

### 5. Capacity: what fits

HBM capacity is a hard wall. Your resident footprint for LLM inference is:

```
HBM used  ≈  weights  +  KV cache  +  activations/workspace  +  framework overhead
```

**Weights.** Fixed per model and precision: `bytes = params × bytes_per_param`.

| Precision | bytes/param | 8B model | 70B model | 405B model |
|---|---|---|---|---|
| FP32 | 4 | 32 GB | 280 GB | 1,620 GB |
| FP16 / BF16 | 2 | 16 GB | **140 GB** | 810 GB |
| FP8 / INT8 | 1 | 8 GB | **70 GB** | 405 GB |
| FP4 / INT4 | 0.5 | 4 GB | 35 GB | 203 GB |

**KV cache.** The variable term, and the one that caps concurrency. Derived properly in §6.

**Activations / workspace.** Transient per-forward-pass buffers: the hidden states, the intermediate FFN projections, attention workspace, and cuBLAS/cuDNN scratch. For decode with a modest batch these are small (hundreds of MB); for a long prefill with a large batch they are not — the FFN intermediate for Llama-3-70B is `batch × seq × 28,672 × 2 B`, which at batch 8 × 4,096 tokens is 1.9 GB *per layer's* intermediate held live. Serving engines chunk prefill precisely to bound this.

**Framework overhead.** CUDA context (a few hundred MB), the allocator's reserve, NCCL buffers if sharded, and fragmentation. Budget **3–5 GB** and treat the serving framework's own reserve (vLLM's `gpu_memory_utilization`, default 0.9) as load-bearing configuration, not cosmetic: it is what stops a mid-request allocation from failing.

If `HBM used > capacity`, you have exactly three moves, each a cost decision with a different shape:

| Move | Buys you | Costs you |
|---|---|---|
| Shard across GPUs (tensor parallel) | capacity, and aggregate bandwidth | an all-reduce per layer per token over NVLink; more GPUs on the invoice; a fault domain that is now N cards wide |
| Quantise | capacity **and** throughput (fewer bytes to stream) | an accuracy delta you must measure, and hardware-gated format support (§4) |
| Bigger-memory SKU | capacity and usually bandwidth, with no code change | a higher hourly rate; availability |

There is no fourth option. Host offload is the 52×-slower path from §1 and is an emergency measure, not a plan.

### 6. KV cache: the formula, derived

During autoregressive generation, producing token *t* requires attending over the Key and Value projections of tokens 1…*t−1* at every layer. Recomputing them each step would be O(t) extra work per token; caching them costs memory instead. That trade is the KV cache.

**What is actually stored.** For each layer, for each token, for each KV head, one K vector and one V vector of length `head_dim`:

```
bytes per token = 2          (one K vector + one V vector)
                × L          (layers)
                × H_kv       (key/value heads)
                × d_head     (dimension per head)
                × e          (bytes per element: 2 for FP16/BF16, 1 for FP8/INT8)
```

Then:

```
KV bytes = bytes_per_token × seq_len × batch
```

**The headline: KV cache grows linearly in context length and linearly in batch — so as their product.** Double the context window or double concurrency and you double the footprint.

**Where GQA comes in.** In plain multi-head attention (MHA), every query head has its own K and V head, so `H_kv = H_q` and `H_kv × d_head = hidden_dim`. That is the version most textbook formulas assume, and it gives the familiar `2 × L × hidden_dim × e` per token.

Multi-Query Attention (MQA), introduced by Noam Shazeer in "Fast Transformer Decoding: One Write-Head is All You Need" (2019), collapses this to **a single shared K/V head** for all query heads. Grouped-Query Attention (GQA), formalised by Ainslie et al. (EMNLP 2023), generalises it: query heads are partitioned into G groups, each sharing one K/V head, so `H_kv = G`. The reduction factor is exactly `H_q / H_kv`.

The physical claim is worth stating plainly because it is the whole point: **GQA does not change how many query heads attend, or the FLOPs of attention. It changes how many distinct K/V vectors must be stored and re-read per token.** Since decode's cost is dominated by re-reading that cache (arithmetic intensity exactly 1, per lesson 03.2), shrinking it by 8× shrinks both the memory footprint and the bandwidth bill by 8×.

**Worked, for Llama-3-70B** (`num_hidden_layers` 80, `hidden_size` 8192, `num_attention_heads` 64, `num_key_value_heads` 8, so `head_dim` = 8192/64 = 128):

```
MHA upper bound (if it used 64 KV heads):
  2 × 80 × 64 × 128 × 2 B = 2,621,440 B = 2.50 MiB per token

Actual, with GQA (8 KV heads):
  2 × 80 ×  8 × 128 × 2 B =   327,680 B = 320   KiB per token
                                          ─────────────────────
  reduction = 64/8 = 8×  ✓ (exactly the head-grouping ratio)

With FP8 KV cache (e = 1):
  2 × 80 ×  8 × 128 × 1 B =   163,840 B = 160   KiB per token   (another 2×)
```

**Per sequence, at context length S:**

| Context | KV bytes/seq (GQA, FP16) | KV bytes/seq (GQA, FP8) |
|---|---|---|
| 2,048 | 0.64 GB | 0.32 GB |
| 4,096 | 1.28 GB | 0.64 GB |
| 8,192 | 2.56 GB | 1.28 GB |
| 32,768 | 10.2 GB | 5.1 GB |
| 131,072 | 41.0 GB | 20.5 GB |

**Read the last row.** One single 128K-context sequence of Llama-3-70B, with GQA already applied, needs 41 GB of KV cache — more than half an H100's entire HBM, for one user. That is why long context is a capacity problem before it is anything else, and why `num_key_value_heads` is the first field you look at in a model config.

### 7. The capacity budget as a timeline: watching HBM fill

Capacity is not a static check; it is a running total that grows as a batch accumulates tokens. This is the picture to hold when a serving engine starts preempting.

```
  HBM OCCUPANCY OVER A SERVING SESSION — Llama-3-70B FP8 weights on one H200 (141 GB)
  ═══════════════════════════════════════════════════════════════════════════════════
  KV per token (GQA 8 heads × 128 dim, 80 layers, FP8 KV) = 160 KiB/token
  Weights (70B × 1 B) = 70 GB   ·   overhead ≈ 4 GB   ·   free for KV = 67 GB
  ⇒ total token budget = 67e9 / 163,840 = 409,000 tokens live, in any mix

  141 ┤██████████████████████████████████████████████████████████████████ CAPACITY
      │
  120 ┤                                                    ┌───────────── ← preempt/
      │                                       ┌────────────┘                 swap here
  100 ┤                          ┌────────────┘   KV cache (grows every step)
      │             ┌────────────┘
   80 ┤ ┌───────────┘
      ├─┴──────────────────────────────────────────────────────────────── overhead 4 GB
   74 ┤
      │
   70 ┤▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ weights
      │▓▓▓▓ FIXED — allocated at load, never moves ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
    0 └──────────────────────────────────────────────────────────────────▶ time
        t=0        prefill        decode step 200      step 800      step 1600
        load      50 seqs ×       50 seqs ×            50 seqs ×     50 seqs ×
        model     2k prompt       2.2k tokens          2.8k tokens   3.6k tokens
                  = 16 GB KV      = 17.6 GB            = 22.4 GB     = 28.8 GB

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ THE OPERATIONAL CONSEQUENCE                                                  │
  │ Admission control decides the batch at t=0 based on PROMPT length, but the   │
  │ footprint grows every decode step. A batch that fits at admission can        │
  │ exhaust HBM 1,500 tokens later. Serving engines respond by PREEMPTING —      │
  │ evicting a sequence's KV cache and recomputing or swapping it later.         │
  │ SYMPTOM: p99 latency spikes with no change in request rate.                  │
  │ CAUSE:   generation-length distribution, not arrival rate.                   │
  └──────────────────────────────────────────────────────────────────────────────┘
```

This is also the mechanism behind PagedAttention. Before vLLM, serving systems pre-allocated a contiguous KV region sized to `max_seq_len` per sequence, so a request that generated 200 tokens against a 4,096-token reservation wasted 95% of its allocation — the Kwon et al. SOSP 2023 paper's central observation is that existing systems waste KV memory to fragmentation and duplication, capping the batch size. PagedAttention borrows OS virtual memory: KV is allocated in fixed-size **blocks** (16 tokens is vLLM's default), a per-sequence **block table** maps logical positions to physical blocks, and blocks are allocated on demand. Fragmentation drops to at most one partial block per sequence, and identical prefixes (a shared system prompt, a beam-search fork) can share physical blocks by reference-counting them. The reported result is **2–4× higher serving throughput** at the same latency versus FasterTransformer and Orca — obtained purely by managing the capacity budget this lesson teaches you to compute by hand.

### 8. Compounding levers: GQA is the first, not the ceiling

GQA/MQA compress KV at the *architecture* level — a decision the model's training team made, which you inherit and can only read out of the config, not choose. But it is one of several **independently stackable** multipliers, and production systems compound them:

| Lever | Typical factor | Where it lives | Notes |
|---|---|---|---|
| GQA / MQA | 4–8× (`H_q / H_kv`) | model architecture | Read `num_key_value_heads`. Baked into the checkpoint. |
| KV quantisation (FP8/INT8) | 2× | serving engine | Independent of *weight* precision. FP8 KV is a Hopper+ path. |
| Paging (PagedAttention) | eliminates fragmentation waste | serving engine | Not a compression ratio — a waste elimination. |
| Prefix / block sharing | workload-dependent, large for shared system prompts | serving engine | Reference-counted physical blocks. |
| Sliding-window / local attention | bounds KV at window size instead of S | model architecture | Changes model behaviour; not a free lever. |

Character.AI's engineering team reports combining native int8 quantisation-aware training, MQA, and KV-cache sharing to cut their production KV footprint **more than 20× without a quality regression** — far more than GQA's 4–8× alone, precisely because they stack a precision lever and a sharing lever on top of the architectural one. **GQA is where this module's KV story starts, not where it ends.**

### 9. Bandwidth: how fast

HBM bandwidth matters because **LLM decode is memory-bound** (lesson 03.2, arithmetic intensity ≈1). To generate one token at small batch, the GPU streams essentially every weight out of HBM once. So:

```
tokens/sec  ≤  HBM_bandwidth ÷ model_bytes
```

| Model / precision | model_bytes | H100 (3.35 TB/s) | H200 (4.8 TB/s) | B200 (7.7 TB/s) |
|---|---|---|---|---|
| 70B FP16 | 140 GB | **23.9 tok/s** | 34.3 tok/s | 55.0 tok/s |
| 70B FP8 | 70 GB | 47.9 tok/s | 68.6 tok/s | 110 tok/s |
| 8B FP16 | 16 GB | 209 tok/s | 300 tok/s | 481 tok/s |
| 405B FP8 | 405 GB | (needs ≥6 GPUs) | (needs ≥3 GPUs) | (needs ≥3 GPUs) |

**Compute never appears in the formula.** An H100 could in principle do 989 TFLOP/s; generating one token from a 70B model needs about `2 × 70e9 = 140 GFLOP`, so at 24 tok/s it performs 3.4 TFLOP/s — **0.34% of peak**, exactly the roofline prediction for AI = 1. Lesson 03.4 stress-tests this number and shows what batching does to it.

### 10. The two levers, side by side

| Question | Governed by | Levers | The mistake |
|---|---|---|---|
| Does model + KV fit? | HBM **capacity** (GB) | bigger card · shard · quantise · shrink KV | budgeting weights only, forgetting KV grows with batch × context |
| How fast does it decode? | HBM **bandwidth** (TB/s) | faster memory · smaller model · **batch** (03.4) | buying FLOPs |
| How much math per second? | **FLOPS** (compute die) | precision, newer die | assuming this gates decode. It does not. |

You can be capacity-bound with bandwidth to spare, or fit comfortably while bandwidth-starved. Diagnosing *which* wall you are hitting is the core skill here, because the fix and its cost differ completely — and the DCGM signature differs too: capacity-bound shows up as a serving engine reporting a small `max_num_seqs` or logging preemptions, while bandwidth-bound shows up as `DCGM_FI_PROF_DRAM_ACTIVE` pinned near 1.0.

## Perspectives

**Developer / ML view.** GQA versus MHA and the number of KV heads are training-time architectural choices, baked into the checkpoint by the time you serve it. You do not choose them, but you must read them out of the config (`num_key_value_heads`, `num_hidden_layers`, `hidden_size`, `num_attention_heads`) to get a KV budget right. Using the plain-MHA formula on a GQA model overstates the footprint by 4–8×, which either wastes capacity you did not need to reserve or, worse, makes you cap concurrency far below what the card supports.

**Operator / platform view.** The memory-budget reflex — weights + KV + overhead against capacity, in under a minute — is the single most common "will this fit" ticket a platform engineer answers, and the difference between a confident yes/no and a production OOM. It is also a whiteboard exercise in interviews, which is why §6's arithmetic should be reproducible from memory. Add one operational instinct on top: the budget is not static (§7), so "fits at admission" is not "fits."

**Hardware view.** HBM stacking is a literal packaging decision — 5×16 GB HBM3 on H100 versus 6×24 GB HBM3e on H200 — with limits set by TSV yield, interposer perimeter and JEDEC pin rates, not by the process node. Grounding "why can't they just make HBM bigger or faster" in the stack-count/pin-rate picture is what separates "I memorised the spec sheet" from "I understand why the spec sheet says what it says" — and it lets you check a bandwidth claim by arithmetic: `stacks × 1024 bits × pin_rate ÷ 8`.

**Economics view.** Character.AI's >20× KV reduction is a direct $/token argument: every byte of KV freed is either more concurrent users per GPU or the ability to run a cheaper SKU for the same traffic. At production chat scale a KV-compression project is a line item in a cloud bill, which is why serving teams staff it. And the H100/H200 pair is the cleanest natural experiment in the market: identical compute, different memory, different price — and the price difference is justified *only* if your workload lives under the memory roof.

## Real-world use cases

- **Kwon et al., ["Efficient Memory Management for Large Language Model Serving with PagedAttention"](https://arxiv.org/abs/2309.06180) (SOSP 2023), and the [vLLM launch post](https://blog.vllm.ai/2023/06/20/vllm.html).** The paper's premise is exactly this lesson's: KV cache is huge, grows and shrinks dynamically, and when managed with contiguous pre-allocation is wasted to fragmentation and duplication — which caps the batch size and therefore throughput. PagedAttention allocates KV in fixed-size blocks with a per-sequence block table (OS paging, applied to attention), achieving near-zero waste and enabling sharing within and across requests. Reported **2–4× throughput improvement** over FasterTransformer and Orca at the same latency. What it shows: the capacity budget you compute by hand in §6 is the quantity a state-of-the-art serving system is engineered around.
- **["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) and [Part Deux](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — Character.AI Engineering.** Native int8 quantisation-aware training + MQA + KV-cache sharing combine to cut KV footprint **more than 20×** at real production chat scale, without a quality regression. The follow-up post extends the same programme. What it shows: the levers in §8 compound, and memory efficiency is treated as an ongoing cost-reduction programme, not a one-time architecture choice.
- **The H100 → H200 SKU split itself.** NVIDIA shipped the same GH100 die with a different memory package: 5×16 GB HBM3 at 3.35 TB/s versus 6×24 GB HBM3e at 4.8 TB/s, identical 989 TFLOP/s dense BF16, at a higher price. What it shows: the vendor's own product segmentation is built on the capacity/bandwidth-versus-compute distinction this lesson teaches — the market has priced these as separate goods, and reasoning about which one your workload buys is the difference between a defensible SKU recommendation and a guess.

## Worked example

**Memory budget: Llama-3-70B on one H100 (80 GB) versus one H200 (141 GB), at FP16 and FP8.**

Config (from the model's `config.json`): `num_hidden_layers` 80, `hidden_size` 8192, `num_attention_heads` 64, `num_key_value_heads` 8 → `head_dim` = 128.

**Step 1 — weights.**

```
FP16:  70e9 params × 2 B = 140 GB
FP8:   70e9 params × 1 B =  70 GB
```

**Step 2 — KV cache per token.**

```
GQA:  2 × 80 layers × 8 kv_heads × 128 head_dim × 2 B  = 327,680 B  = 320 KiB/token
FP8 KV:                                        × 1 B  = 163,840 B  = 160 KiB/token

Sanity check against the MHA upper bound:
      2 × 80 × 8192 (hidden) × 2 B = 2,621,440 B = 2.50 MiB/token
      2.50 MiB ÷ 320 KiB = 8×  ✓ equals H_q/H_kv = 64/8
```

**Step 3 — the fit verdict at FP16.**

```
H100  80 GB:  140 GB of weights ALONE > 80 GB  → DOES NOT FIT. Not even the weights.
              Shortfall 60 GB. Options: 2× H100 tensor-parallel (70 GB/card of
              weights, leaving ~6 GB/card for KV), or quantise to FP8 (70 GB).

H200 141 GB:  140 GB weights ≤ 141 GB          → weights fit, with 1 GB left.
              1 GB ÷ 320 KiB/token = 3,200 tokens of KV, total, across ALL users
              — less than one 4k-context request. NOT SERVABLE in practice.
```

**A 70B FP16 model is a ≥2-GPU model on Hopper regardless of SKU.** The H200's capacity advantage shows up for 30–40B-class FP16 models, or in giving a model that already fits far more KV headroom — which is the *useful* form of the advantage, per step 4.

**Step 4 — the fit verdict at FP8, and the batch size where HBM runs out.** This is the calculation that actually matters operationally.

```
                                        H100 80 GB        H200 141 GB
  weights (FP8)                          70.0 GB            70.0 GB
  framework overhead + CUDA ctx           4.0 GB             4.0 GB
  ───────────────────────────────────────────────────────────────────
  free for KV cache                       6.0 GB            67.0 GB
  KV per token (FP8 KV, GQA)            160 KiB            160 KiB
  ───────────────────────────────────────────────────────────────────
  total live tokens affordable           36,700           409,000
                                    (6.0e9/163,840)   (67.0e9/163,840)

  Concurrency at a given context length  = total_tokens ÷ context
      2k context:                             17 seqs           199 seqs
      4k context:                              8 seqs            99 seqs
      8k context:                              4 seqs            49 seqs
     32k context:                              1 seq             12 seqs
    128k context:                              0 (!)              3 seqs
```

**Look at the 8k row: 4 concurrent sequences on the H100, 49 on the H200 — a 12× difference in concurrency from a 76% capacity increase.** That non-linearity is the single most important number in this lesson. Because weights are a *fixed* subtraction, capacity headroom above the weights grows far faster than total capacity does: H200 has 1.76× the HBM but **11.2× the free-for-KV space** (67 GB vs 6 GB) for this model.

That is the H200 inference argument, and it is a capacity argument, not a bandwidth one.

**Step 5 — bandwidth: the decode ceiling on each.**

```
Single-stream ceiling = HBM bandwidth ÷ model bytes

  H100, 70B FP8:  3.35e12 B/s ÷ 70e9 B  = 47.9 tok/s
  H200, 70B FP8:  4.80e12 B/s ÷ 70e9 B  = 68.6 tok/s      (+43%, purely bandwidth)
  H100, 70B FP16: 3.35e12    ÷ 140e9    = 23.9 tok/s
  H200, 70B FP16: 4.80e12    ÷ 140e9    = 34.3 tok/s
```

**Step 6 — put the two levers together, which is the whole point.**

```
Aggregate throughput ≈ (bandwidth ceiling) × (achievable batch) × (realisation factor)
                        └── capacity-independent ──┘  └─ set by capacity ─┘

H100, 70B FP8, 8k ctx:  47.9 tok/s × 4 seqs  × ~0.75 ≈    144 tok/s
H200, 70B FP8, 8k ctx:  68.6 tok/s × 49 seqs × ~0.75 ≈  2,520 tok/s
                                                       ────────────
                                        ratio ≈ 17.5×, from a card that is
                                        ~23% more expensive per hour.
```

The bandwidth lever contributed 1.43× of that; **the capacity lever contributed 12×.** If you only ever quote "H200 is 43% faster for inference" you have told a small fraction of the story. (The realisation factor of ~0.75 covers KV reads on top of weight reads, attention overhead and imperfect bandwidth utilisation; lesson 03.4 unpacks it and shows where the batching curve actually flattens.)

**Step 7 — cost.**

```
H100 at ~$3.00/hr →   144 tok/s → 518,400 tok/hr → $5.79 per 1M output tokens
H200 at ~$3.70/hr → 2,520 tok/s → 9.07M tok/hr   → $0.41 per 1M output tokens
```

A 14× difference in cost per token, from the same compute die, decided entirely by memory. *(Rates are illustrative 2026-era snapshots; lesson 03.7 covers sourcing live pricing. The point is the ratio, which is set by hardware, not the absolute, which is set by the market.)*

## Practice

Rent one GPU (H100 80 GB ideal; any modern datacenter GPU works — adjust the spec numbers to your SKU). Target ~1 hour of GPU time. Results feed the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

1. **Record your SKU's memory spec and check it arithmetically.** Capacity and bandwidth from the datasheet; then verify the bandwidth against `stacks × 1024 bits × pin_rate ÷ 8` using the table in §3. Note whether your card's usable capacity differs from its physical DRAM.
2. **Load a model and measure resident memory.** Pick one that fits (an 8B or 13B at FP16 on an 80 GB card). Record `nvidia-smi` memory used before and after load, and `torch.cuda.memory_allocated()` versus `torch.cuda.memory_reserved()` after load. Confirm allocated matches `params × bytes_per_param` within a few percent, and **note the gap between allocated and reserved** — that gap is the allocator's caching behaviour and it is why `nvidia-smi` always shows more than PyTorch admits to.
3. **Compute the KV formula by hand, then verify it against the engine.** Pull `num_hidden_layers`, `hidden_size`, `num_attention_heads` and `num_key_value_heads` from the model's `config.json`. Compute `head_dim = hidden_size / num_attention_heads` and `bytes/token = 2 × L × H_kv × d_head × e`. Then start vLLM and read the startup log line reporting **GPU KV cache size** (in tokens or blocks) — `KV cache size: N tokens`. Compute `free_bytes ÷ your_bytes_per_token` yourself and compare. They should agree within a few percent; explain any gap.
4. **Compute the concurrency table** for two context lengths (e.g. 2k and 32k) as in worked-example step 4, and state the batch size at which HBM is exhausted for each.
5. **Run an HBM bandwidth microbenchmark.** Simplest reliable approach: a large `torch` elementwise op on a multi-GB tensor, timed, computing `bytes_moved ÷ seconds` and counting reads *and* writes. Or use NVIDIA's `bandwidthTest` from the CUDA samples. Record achieved GB/s.
6. **Compare achieved to spec.** Expect **70–90%** of spec on a well-formed benchmark; record your efficiency and account for the gap (DRAM refresh, ECC overhead, row-activation cost, imperfect coalescing). Cross-check with `DCGM_FI_PROF_DRAM_ACTIVE` from lesson 03.1: it should sit near 0.85–0.95 during the benchmark.
7. **(Stretch) Demonstrate the wall.** Deliberately request a KV cache larger than free HBM (raise `--max-model-len` or `--max-num-seqs` in vLLM until it refuses or preempts). Capture the error or the preemption log line. This is the failure mode you are learning to predict.

**Acceptance:** a **memory-budget breakdown** for one model on one SKU containing: weights (measured *and* hand-computed), your hand-derived bytes-per-token with the config fields it came from, KV capacity in tokens (hand-computed *and* as reported by the serving engine), the concurrency table at two context lengths, framework overhead, total versus HBM capacity, and your measured-versus-spec HBM bandwidth in GB/s and as a percentage. One table plus four sentences of interpretation is enough — but the hand-computed and measured columns must both be there, because the whole skill is knowing they agree.

## Common pitfalls

1. **Using the plain-MHA KV formula on a modern model.** Nearly everything shipped since 2023 uses GQA or MQA. Substituting `hidden_dim` for `H_kv × d_head` overstates KV by the head-grouping ratio (8× for Llama-3-70B), which makes a servable configuration look like it needs sharding it does not need. *Check:* `num_key_value_heads` in `config.json`.
2. **Assuming capacity and bandwidth move together generationally.** H100 → H200 is direct proof they do not: same compute die, +76% capacity, +43% bandwidth, +0% FLOPS. Never infer one from the other.
3. **Budgeting weights only.** A model that fits its weights can still be unservable — worked-example step 3 shows Llama-3-70B FP16 "fitting" in 141 GB with 1 GB spare, which supports 3,200 KV tokens total, less than one request. **The question is never "do the weights fit," it is "do the weights plus a useful batch's KV fit."**
4. **Treating the memory budget as static.** KV grows every decode step (§7). A batch admitted on prompt length can exhaust HBM a thousand tokens later, producing p99 latency spikes with no change in request rate. Size against expected *generation* length, not prompt length.
5. **Forgetting framework overhead and reserve.** CUDA context, activation workspace, NCCL buffers and fragmentation are real. Budget 3–5 GB, and treat vLLM's `gpu_memory_utilization` (default 0.9) as load-bearing configuration.
6. **Believing precision only affects weights.** The KV cache can be quantised independently of weight precision (FP8 KV on Hopper+, INT8 elsewhere) — Character.AI's stack does exactly this on top of MQA, and it is a large share of their >20×.
7. **Planning against physical rather than usable capacity.** H200 has 144 GB of DRAM and exposes 141 GB; B200 has 192 GB and exposes 180 GB in HGX/DGX form. A budget built on the physical figure is 2–6% optimistic before you start.
8. **Treating host offload as a capacity plan.** PCIe Gen5 x16 is ~64 GB/s against HBM's 3,350 GB/s. Streaming weights from host DRAM turns a 24 tok/s model into a sub-1 tok/s model. It is an emergency measure.

## Self-check

- **Does a 70B model at FP16 fit on one H100 (80 GB)? One H200 (141 GB)? Show the math.** Weights = 70e9 × 2 B = 140 GB. H100: 140 > 80 → no, the weights alone are 60 GB over; you need ≥2× H100 tensor-parallel or FP8/INT8 quantisation (70 GB, which then fits with ~6 GB for KV). H200: 140 ≤ 141 → the weights technically fit with ~1 GB spare, which at 320 KiB/token of GQA KV is about 3,200 tokens total — under one 4k-context request. Not servable in practice. **A 70B FP16 model is a ≥2-GPU model on Hopper regardless of SKU.**
- **How does KV-cache size scale, and what exactly does GQA change?** `KV bytes = 2 × L × H_kv × d_head × e × seq_len × batch` — linear in context length and linear in batch, so it grows as their *product*. GQA/MQA change only the constant `H_kv`: query heads share K/V heads, so the reduction is exactly `H_q / H_kv` (8× for Llama-3-70B's 64 query heads over 8 KV heads). It does not change the linear scaling law, and it does not change attention's FLOPs — only how many distinct K/V vectors must be stored and re-read.
- **Compute bytes/token for Llama-3-70B and the batch size at which one H200 running FP8 weights and FP8 KV runs out at 8k context.** `2 × 80 × 8 × 128 × 1 B = 163,840 B = 160 KiB/token`. Free HBM = 141 − 70 (weights) − 4 (overhead) = 67 GB. Total tokens = 67e9 / 163,840 = 409,000. At 8,192 tokens per sequence: 409,000 / 8,192 = **49 concurrent sequences**. On an H100 the same arithmetic gives 6 GB free → 36,700 tokens → **4 sequences.**
- **Why is the H200 (same compute die as H100) often the better inference buy?** Because inference decode is memory-bound, not compute-bound. The H200 keeps the identical GH100 die but changes the memory package: 6×24 GB HBM3e at 4.8 TB/s versus 5×16 GB HBM3 at 3.35 TB/s. Bandwidth raises the per-stream decode ceiling by 43% (`tok/s ≤ BW ÷ model_bytes`). *Capacity* does something larger and less obvious: because weights are a fixed subtraction, +76% total capacity became +1,017% free-for-KV space for a 70B FP8 model (6 GB → 67 GB), which is a ~12× concurrency increase. You pay nothing for FLOPs you could not use, and get both things that actually gate inference.
- **Where does HBM's bandwidth physically come from, and how would you check a vendor's claim?** Stacked DRAM dies joined by TSVs, mounted on a silicon interposer beside the GPU die, connected by a very wide bus — 1,024 bits per stack (16 channels × 64 bits, each split into 2 pseudo-channels) under JEDEC's HBM3 spec. Bandwidth = `stacks × 1024 bits × pin_rate ÷ 8`. HBM3 tops out at 6.4 Gb/s per pin (819 GB/s per stack); HBM3E raises this to up to 9.6 Gb/s. Check: H200 is 6 stacks × 1,024 bits × 6.25 Gb/s ÷ 8 = 4.8 TB/s ✓. If a claimed number needs a pin rate above the JEDEC maximum for the stated generation, the claim or the generation is wrong.
- **Why can't you page a model to host memory the way a CPU pages to disk?** Because the next tier down is PCIe Gen5 x16 at ~64 GB/s against HBM's 3,350 GB/s — 52× slower — and decode must stream *every weight* per token. A 70B FP16 model at 140 GB would take 2.2 seconds per token off host memory instead of 42 ms. Unlike a CPU's page cache, there is no locality to exploit: the access pattern is a full sweep of the weights every single step. Capacity is a wall, not a gradient.
- **Character.AI reports >20× KV reduction. Name the techniques and which layer each lives at.** Native int8 quantisation-aware training (precision — serving engine and training, lesson 03.5), MQA (model architecture — the `H_kv` term in this lesson's formula), and KV-cache sharing across sequences (a systems-level deduplication on top of both). The point is that they are independently stackable multipliers on the same number: architecture × precision × sharing, and GQA alone is only the first factor.
- **A serving deployment shows p99 latency spiking every few minutes with a flat request rate. First hypothesis?** KV-cache exhaustion causing preemption. The batch is admitted on prompt length but the footprint grows every decode step (§7); once free HBM runs out the engine evicts sequences and later recomputes or swaps them, which shows up as latency spikes uncorrelated with arrival rate. Check the engine's preemption counter and the KV-cache-usage gauge, and compare the generation-length distribution against the token budget you computed by hand.

## Connections & what's next

This lesson turns lesson 03.2's abstract "decode is deep memory-bound" into concrete GB and TB/s you can compute for any model/SKU pair. It sets up lesson 03.4 directly: the bandwidth ceiling here (`tok/s ≤ BW ÷ model_bytes`) is the single-stream number 03.4 extends into a full batching model, and the token-budget arithmetic here is exactly what sets 03.4's batch-size cap. It hands off to lesson 03.5 (precision), which is the multiplier on both — every capacity number here is stated at two precisions for that reason — and to lesson 03.6, where the H100-versus-H200 argument in the worked example becomes a full generational-SKU comparison. The capacity/bandwidth-as-independent-levers thread is the same one module 07 (inference serving) and module 11 (GPU cost economics) build production architecture and fleet cost models on.

Next: **[03.4 · Decode bandwidth ceilings, batching, and the prefill/decode split](04-decode-bandwidth-batching.md)** takes the bandwidth ceiling introduced here and derives, in full, why batching is the only lever that raises aggregate decode throughput — and why the KV-cache capacity computed in this lesson, not compute, is almost always what stops you batching further.

## References & further reading

**Primary sources**

1. NVIDIA, [H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — the on-die memory hierarchy used in §1: 256 KB register file per SM, 256 KB unified L1/shared with up to 228 KB usable as shared memory, 50 MB L2, 5 HBM3 stacks behind 10 × 512-bit controllers (5,120-bit bus) at 3.35 TB/s.
2. NVIDIA, [H200 Tensor Core GPU page/datasheet](https://www.nvidia.com/en-us/data-center/h200/) — canonical 141 GB usable / 4.8 TB/s, and confirmation that the H200 reuses the H100 compute die (identical 989 dense BF16 TFLOP/s, 3,958 FP8 TFLOP/s with sparsity).
3. NVIDIA, [A100 Tensor Core GPU datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-nvidia-us-2188504-web.pdf) — 80 GB HBM2e at 2.039 TB/s, 312 TFLOP/s dense BF16, and the absence of an FP8 tensor path (INT8 at 624 TOP/s dense is the Ampere quantisation route).
4. JEDEC, **JESD238 / JESD238A (HBM3)** and **JESD238B (HBM3E)** — the standards behind §3: 16 independent channels of 64 bits per stack (1,024-bit interface), 2 pseudo-channels per channel, up to **6.4 Gb/s per pin → 819 GB/s per stack** for HBM3, raised to up to 9.6 Gb/s per pin (≈1.23 TB/s per stack) for HBM3E, with 8–32 Gb per memory layer. Read for the per-stack ceiling that lets you check any vendor bandwidth claim by arithmetic. Announcement summary at [jedec.org](https://www.jedec.org/news/pressreleases/jedec-publishes-hbm3-update-high-bandwidth-memory-hbm-standard).
5. Kwon et al., ["Efficient Memory Management for Large Language Model Serving with PagedAttention"](https://arxiv.org/abs/2309.06180) (SOSP 2023) — the formal treatment of KV-cache fragmentation and paging in §7: block-based allocation, per-sequence block tables, near-zero waste, sharing within and across requests, and the reported **2–4× throughput gain** over FasterTransformer and Orca.
6. Ainslie et al., ["GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints"](https://arxiv.org/abs/2305.13245) (EMNLP 2023) — the formal source for the `H_kv × d_head` substitution in §6 and the uptraining recipe that made GQA the default for open-weight models.
7. Shazeer, "Fast Transformer Decoding: One Write-Head is All You Need" (2019) — the original MQA paper; read for the decode-bandwidth motivation behind sharing K/V projections across query heads, which is the argument GQA generalises. *(Canonical paper cited by title/author/year; the arXiv URL was not independently fetched in this pass.)*
8. Dao et al., ["FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness"](https://arxiv.org/abs/2205.14135) (NeurIPS 2022) — source of the A100 on-chip SRAM figure used to corroborate §2's derivation ("192 KB per each of 108 SMs with bandwidth estimated around 19 TB/s" against a derived 19.5 TB/s).
9. Luo et al., ["Benchmarking and Dissecting the Nvidia Hopper GPU Architecture"](https://arxiv.org/abs/2402.13499) — the measured latency figures in §1 (A100-class: shared ≈29, L1 ≈38, L2 ≈262, global ≈466 cycles) and the tier ratios that hold across Ampere and Hopper.

**Real-world engineering blogs**

10. vLLM Project, ["vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention"](https://blog.vllm.ai/2023/06/20/vllm.html) — the production framing of the SOSP paper; what it shows: fixing the capacity budget at the systems level rather than the architecture level.
11. Character.AI Engineering, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) — >20× KV reduction via combined int8 QAT + MQA + sharing at production chat scale.
12. Character.AI Engineering, ["Optimizing AI Inference at Character.AI, Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — the follow-up; memory efficiency as a continuing engineering programme rather than a one-time fix.

**Deeper dives**

13. [vLLM documentation](https://docs.vllm.ai/) — for the practice section and beyond: `gpu_memory_utilization`, block size, the startup log line reporting KV-cache size in tokens, and preemption metrics. The fastest way to check your hand-computed budget against a real engine's own accounting.
