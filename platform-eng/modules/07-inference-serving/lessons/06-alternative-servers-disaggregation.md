---
lesson: "07.6"
title: "Alternative servers and disaggregation"
module: "07"
concept: "Alternative servers and disaggregation"
status: not-started
est_time: "7h"
prev: "05-batching-economics.md"
next: "07-quantization-ops.md"
artifacts: []
sources: 16
---
# 07.6 · Alternative servers and disaggregation

> **Concept.** Pick an inference engine by workload shape, not by benchmark bragging rights; and understand disaggregated prefill/decode as the 2025–26 cost frontier.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

Lesson 05 gave you the CPM-vs-batch curve and the SLO knee for **one engine on one fused GPU pool** — vLLM, prefill and decode sharing the same batch. That curve is real, but it silently assumes the engine and the topology are fixed and only the batch size moves. This lesson removes both assumptions: it asks which engine you should even be running the sweep against, and whether prefill and decode belong on the same GPUs at all. The gap it fills is the two decisions that sit *above* the batching sweep — pick the runtime, pick the topology — before you even get to pick the batch size.

Everything version-specific below is checked against source: **vLLM v0.27.1** (`docs/features/disagg_prefill.md`, `docs/features/nixl_connector_usage.md`, `vllm/config/`), **SGLang v0.5.17** (`python/sglang/srt/server_args.py`, `python/sglang/srt/mem_cache/radix_cache.py`), **TensorRT-LLM 1.3.0-rc** (`docs/source/features/`), and **TGI's own README banner**. Several claims in circulation about all four are stale; §8 and §14 say which.

## Why this matters

vLLM is the sane default (lessons 03–05), but "we run vLLM everywhere" is a junior answer at a GPU-heavy shop. The staff-level skill is matching the engine to the **workload shape** — because the wrong engine leaves 20–100% of your CPM on the table for a specific traffic pattern, and CPM is the metric this module is built around. A RAG platform with a 4k-token shared system prompt on every request has completely different economics from a batch-summarisation job, and a different engine wins each.

The interview signal is exactly this: given a workload, name the engine and the reason in one sentence, and know when the frontier techniques (disaggregation, KV-aware routing) are worth their operational cost. That is what separates a platform engineer from someone who copied a Helm chart. The stakes are not abstract: teams operating at real scale — Moonshot AI's Kimi, DeepSeek's own production API — have published disaggregation numbers in the range of 1.6–6× effective capacity versus a fused baseline. That is not a marginal optimization; at fleet scale it is the difference between one GPU-quarter of budget and six.

The other half of "why this matters" is the failure mode of getting disaggregation *wrong*. Disaggregation moves the KV cache across the network on every request. If the fabric under it is 25 GbE instead of RDMA, you have not built a faster system — you have built one where **every single request pays a transfer tax larger than the prefill it was trying to accelerate**, and §11 gives you the one-line arithmetic that tells you which side of that line you are on before you provision anything.

You already own the vocabulary from earlier lessons — prefill vs. decode, compute- vs. memory-bound (module 03), TTFT/TPOT/queue-depth (module 05), continuous batching and the CPM curve (lesson 05). This lesson layers engine selection and the disaggregation frontier on top; it does not re-teach the phases.

## What's new here (calibration)

You already have the phase split, the SLIs, and the batching-CPM relationship. Four things are genuinely new:

- **Prefix sharing as a data structure, not a slogan.** RadixAttention (SGLang) caches KV in a radix tree keyed on token prefixes, with LRU eviction driven off a min-heap over the tree's *leaves*. §4 walks the tree, the match, the split, and the eviction. Knowing it is a tree is not the point; knowing *which node gets evicted first and why your hit rate collapsed* is.
- **Engine as workload-shaped choice, not a leaderboard.** Six runtimes, each with a shape it wins. §3 is a real decision table on the axes that actually decide it, not a throughput ranking.
- **Disaggregated serving, and what it does and does not buy.** vLLM's own documentation is blunt about this in a way most write-ups are not: *disaggregated prefill does not improve throughput.* It lets you tune TTFT and ITL **separately** and it controls tail ITL. If your pitch for disaggregation is "more tokens per second," you have the wrong mental model, and §9 fixes it.
- **The interconnect arithmetic that decides it.** The ratio of KV-transfer time to prefill time turns out to be **independent of prompt length** — it is a pure property of your model, your parallelism, and your fabric. §11 derives it in four lines. It is the single most useful thing in this lesson.

## Core concepts

### 1. What an "inference engine" actually is — five layers, often confused

Half the confusion in engine-selection conversations comes from comparing things at different layers. Draw the stack once and the comparison stops being apples-to-oranges.

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ L5  PLATFORM / CONTROL PLANE      KServe (InferenceService CRD),         │
  │     "many models, many teams,      Ray Serve, SageMaker endpoints        │
  │      GitOps, canary, autoscale"    ── does NOT run a forward pass ──     │
  └───────────────────────────┬──────────────────────────────────────────────┘
                              │ owns Deployments / revisions / traffic split
  ┌───────────────────────────▼──────────────────────────────────────────────┐
  │ L4  FLEET ROUTER                   NVIDIA Dynamo, vLLM production-stack  │
  │     "which replica gets this        router, SGLang router,               │
  │      request, and why"              KV-aware / prefix-affinity routing   │
  └───────────────────────────┬──────────────────────────────────────────────┘
                              │ HTTP, prefix-hash affinity, P/D pairing
  ┌───────────────────────────▼──────────────────────────────────────────────┐
  │ L3  SERVING RUNTIME (multi-backend)  Triton Inference Server             │
  │     "one endpoint, many model         (backends: TensorRT-LLM, vLLM,     │
  │      types, dynamic batching"          ONNX, PyTorch, Python)            │
  └───────────────────────────┬──────────────────────────────────────────────┘
                              │ in-process or gRPC to the engine
  ┌───────────────────────────▼──────────────────────────────────────────────┐
  │ L2  LLM ENGINE                     vLLM · SGLang · TensorRT-LLM · TGI    │
  │     scheduler + KV manager +        ── THIS is where continuous batching,│
  │     attention kernels + sampler        PagedAttention, chunked prefill,  │
  │                                        prefix caching, LoRA live ──      │
  └───────────────────────────┬──────────────────────────────────────────────┘
                              │ CUDA / HIP kernels
  ┌───────────────────────────▼──────────────────────────────────────────────┐
  │ L1  KERNEL + HARDWARE     FlashAttention/FlashInfer, cuBLASLt, CUTLASS,  │
  │                            NCCL, NIXL/UCX, the GPU itself                │
  └──────────────────────────────────────────────────────────────────────────┘
```

Three consequences you should be able to state without hedging:

1. **KServe does not compete with vLLM.** KServe is L5; it *runs* vLLM at L2 inside an `InferenceService`. "vLLM or KServe?" is a malformed question; "vLLM under KServe, or vLLM under a bare Deployment?" is the real one.
2. **Triton does not compete with vLLM either** — it is L3, and one of its backends is literally vLLM. You pick Triton when you need one endpoint in front of *heterogeneous model types* (an LLM, an embedding model, a reranker, a vision classifier), not because it makes an LLM faster.
3. **Only vLLM, SGLang, TensorRT-LLM and TGI are actually comparable**, because only they own the scheduler and the KV cache. That is the comparison §3 makes.

### 2. The five axes that actually decide the engine

Before the table, the axes — because a table whose columns you do not understand is a table you will misread.

- **Time-to-first-deploy.** How long from "here is a HuggingFace repo id" to "there is an OpenAI-compatible endpoint." For vLLM and SGLang this is one command and a weight download. For TensorRT-LLM's classic path it is a *compile* — a per-model, per-GPU, per-parallelism-config engine build, producing an artifact you must version and ship. (TensorRT-LLM's newer PyTorch backend removes the mandatory build step for many models, which is exactly the kind of thing that makes 2024-era comparisons stale.)
- **Model coverage.** vLLM's model registry is the broadest by a wide margin; TensorRT-LLM's support matrix is explicit and per-model (see its own quantization matrix in §6), so a model outside it is not "slower," it is *absent*.
- **Prefix-reuse aggressiveness.** How much repeated prompt prefix the engine turns into a cache hit, and whether the cache is a *tree* (branching reuse) or a linear chain.
- **Peak per-GPU throughput on NVIDIA silicon.** TensorRT-LLM's whole reason for existing. It is genuinely the ceiling — earned with fused kernels, NVFP4/FP8 paths, and NVIDIA-internal knowledge of the hardware.
- **Operational surface.** Number of moving parts you must run, monitor, upgrade and page on. Every layer you add above L2 costs on-call attention.

### 3. The engine comparison — the actual decision table

Verified against each project's own source tree at the versions named in §"Where this fits". `✔` = first-class and documented; `~` = supported with caveats; `✘` = not available.

| | **vLLM** (v0.27.1) | **SGLang** (v0.5.17) | **TensorRT-LLM** (1.3.0-rc) | **TGI** (v3.3.7) |
|---|---|---|---|---|
| **Layer** | L2 engine | L2 engine | L2 engine (usually under Triton/Dynamo) | L2 engine |
| **Project status** | Actively developed, PyTorch-ecosystem project, fastest-moving | Actively developed, joined the PyTorch ecosystem 2025 | Actively developed by NVIDIA | **Maintenance mode** — see §8 |
| **Time to first deploy** | `vllm serve <repo-id>` — minutes | `python -m sglang.launch_server --model-path <repo-id>` — minutes | Pre-quantized checkpoint via the LLM API is quick; classic engine build is hours and per-config | `docker run … --model-id <repo-id>` — minutes |
| **Model coverage** | Broadest in the ecosystem | Broad; day-0 support for major open releases | Explicit per-model support matrix; narrower | Frozen at maintenance-mode scope |
| **KV management** | PagedAttention, block-level, prefix caching **on by default** (`enable_prefix_caching=True`) | RadixAttention — radix tree, LRU-over-leaves eviction (§4) | Paged KV + in-flight batching (IFB) | Paged KV, prefix caching in v3 |
| **Prefix reuse shape** | Linear/block-hash chain; excellent for one shared system prompt | **Tree** — branching reuse (agent fan-out, beam-like structures) | Block reuse | Linear |
| **Multi-LoRA** | ✔ `--enable-lora`, default `max_loras=1` (raise it) | ✔ `--enable-lora`, default `max_loras_per_batch=8`, LRU adapter eviction | ✔ | ~ |
| **PD disaggregation** | ✔ Experimental; 9 connector types incl. `NixlConnector`, `MooncakeConnector`, `LMCacheConnectorV1` | ✔ `--disaggregation-mode prefill\|decode`, Mooncake transfer backend by default | ✔ UCX / NIXL / MPI backends; `trtllm-serve` + Dynamo | ✘ |
| **Quantization** | FP8 (per-tensor/block/channel), MXFP8, MXFP4, AWQ, GPTQ, compressed-tensors, bitsandbytes, online quantization at load | FP8, NVFP4, AWQ, GPTQ | NVFP4, MXFP4, FP8 (per-tensor / block / rowwise), W4A8+W4A16 AWQ/GPTQ, FP8 & NVFP4 KV cache | GPTQ, AWQ, bitsandbytes, EETQ |
| **Peak tok/s/GPU on NVIDIA** | High | High | **Highest** — the reason it exists | Middling |
| **Non-NVIDIA** | ROCm, TPU, XPU, CPU, Gaudi (via plugins) | ROCm, TPU (SGLang-Jax) | NVIDIA only | ROCm, Gaudi, Neuron |
| **Metrics prefix** | `vllm:` | `sglang:` (needs `--enable-metrics`) | Triton/`trtllm-serve` metrics | `tgi_` |
| **Pick it when** | Default. Broad models, fast iteration, K8s-native, needs LoRA/quantization variety | Prefix-heavy, **tree-structured** reuse: agents, RAG fan-out, multi-turn at scale | A small set of stable, very-high-volume models on NVIDIA where the last 20–40% of per-GPU throughput is worth a build pipeline | **Never for new work.** Recognise it in legacy stacks and plan the migration |

Two clarifications the table cannot carry:

**"TensorRT-LLM is fastest" is a true statement about a narrow question.** It is fastest *per NVIDIA GPU, on a model it supports, after you have paid the configuration cost.* It is not fastest per engineer-hour, not fastest to ship a new model, and not portable off NVIDIA. Being able to say that sentence — instead of either "TRT-LLM is best" or "TRT-LLM is a pain" — is the calibrated answer.

**The vLLM/SGLang prefix gap has narrowed to a specific shape.** vLLM ships automatic prefix caching **on by default** in current builds, so for a single long linear shared prefix (one system prompt, repeated) the two engines are close. SGLang's remaining structural advantage is *branching*: many requests sharing a common root and then diverging, which is what an agent fan-out or a multi-candidate generation looks like. A radix tree stores that shape natively; a linear cache stores each branch separately.

### 4. RadixAttention, mechanically

Start with the problem, because the mechanism only makes sense against it. A RAG endpoint sends the same 3,500-token policy preamble on every request. Prefill cost is `2 × P × T` FLOPs (module 03's arithmetic), so those 3,500 tokens cost real compute *every single time*, and they produce **exactly the same KV tensors every single time**, because attention over a prefix does not depend on anything that comes after it. Recomputing them is pure waste — but only if you can (a) recognise the repeat and (b) still have the KV blocks around.

A radix tree (a compressed prefix trie) solves both. Each edge carries a *run* of tokens; each node owns the KV blocks for its edge's tokens.

```
  RADIX TREE OVER TOKEN PREFIXES  —  edges hold token runs, nodes hold KV blocks
  ═══════════════════════════════════════════════════════════════════════════

                          [ ROOT ]
                              │
            "You are a policy assistant… <3,500 tok preamble>"
                              │            ← ONE copy of 3,500 tokens of KV
                        ┌── (A) ──┐          lock_ref = 3 (3 live requests)
                        │         │          last_access = t=41.2
      "Refund window?"  │         │  "Escalation path?"
                        │         │
                      (B)         (C)
                   4 tokens     5 tokens        ← the only per-request KV
                  lock_ref=1   lock_ref=0
                  t=41.2       t=12.7  ◀── LRU victim: oldest evictable leaf


  REQUEST ARRIVES: preamble + "Refund window for EU?"
  ───────────────────────────────────────────────────
   1. match_prefix() walks ROOT → longest matching edge run
      → matches all 3,500 preamble tokens, then 3 of 4 tokens on edge (B)
   2. PARTIAL MATCH forces a SPLIT: node (B) is cut into
         (B1) "Refund window"  ← now shared      (B2) "?"  ← old tail
      the new request hangs (B3) "for EU?" off (B1)
   3. inc_lock_ref() on every node on the matched path — locked nodes are
      NOT evictable while a request is using them
   4. Only the 4 unmatched tokens are prefilled.
      3,500 tokens of prefill compute → ZERO.

  EVICTION when the KV pool is full
  ─────────────────────────────────
   evict(num_tokens):
     heap ← min-heap over EVICTABLE LEAVES, priority = last_access_time
     while evicted < num_tokens:
         x = pop(heap)                       # oldest leaf first
         free x's KV blocks; delete x
         if x.parent has no children left and x.parent.lock_ref == 0:
             push(heap, x.parent)            # parent just became a leaf
```

Three properties fall straight out of that picture, and they are what you actually need on-call:

1. **Eviction is leaf-first, LRU by `last_access_time`.** An interior node — like the 3,500-token preamble at (A) — can only be evicted *after every one of its children is gone*. That is not an accident; it is exactly the right policy, because the interior node is the one whose reuse value is highest. A hot shared prefix is structurally the last thing to be dropped.
2. **`lock_ref` is a refcount, not a lock.** Nodes on the path of any in-flight request are pinned. So "my cache hit rate collapsed" under heavy load is often not eviction policy failure — it is that so much of the pool is *pinned by running requests* that there is nothing evictable left, and the allocator starts refusing admissions instead. That looks like queue growth, not like a cache miss.
3. **A partial match splits a node.** This is the operation that makes the tree adaptive: it discovers shared prefixes it was never told about. It is also where the bookkeeping cost lives — every split is tree surgery on the CPU while the GPU waits.

**Where prefix sharing does not help — and this is self-check (a).** Workloads with **unique, non-overlapping prompts**: batch document summarisation (every doc different), translation of distinct inputs, classification of independent records, any high-entropy prompt stream. With nothing shared there is nothing to cache; the tree adds pure bookkeeping. vLLM measured its own prefix caching at **under 1% throughput cost at a 0% hit rate**, which is why leaving it on is safe — but "safe to leave on" is not "a reason to switch engines." On a unique-prompt workload SGLang and vLLM land at essentially the same CPM and you choose on operational grounds.

### 5. vLLM — why it is the default, stated precisely

Not "it's popular." Four concrete properties:

- **PagedAttention** (lesson 03) makes KV a block-allocated resource, so fragmentation does not waste cache and `max_num_seqs` can run near the true KV ceiling.
- **Defaults that are already right.** In v0.27.1, `gpu_memory_utilization` defaults to **0.92**, `enable_prefix_caching` defaults to **True**, chunked prefill is on whenever possible, and the V1 preemption mode is `RECOMPUTE`. You are tuning from a sane baseline, not building one.
- **Breadth of quantization and loading paths** (lesson 07 and lesson 09 both lean on this): online FP8 quantization at load with no calibration file, FP8/NVFP4 KV cache, `--load-format runai_streamer` / `tensorizer`, and multi-LoRA — all in one binary.
- **The metrics surface module 05 already taught you.** `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`, `vllm:num_requests_waiting_by_reason`, `vllm:time_to_first_token_seconds`. Your dashboards and your KEDA triggers (lesson 08) are already written against it.

The cost of vLLM being the default is that it is also **the fastest-moving dependency in this module**. Metric names have been renamed (module 05 §7 lists which), flags have been added and deprecated, and the V0→V1 rewrite invalidated a generation of tutorials. Pin the version in every command and every README. That is not fussiness; it is the single highest-frequency source of "the tutorial doesn't work."

### 6. TensorRT-LLM — what the build actually buys, and what it costs

The engine's pitch is that NVIDIA can fuse kernels and pick numerics that a general-purpose runtime cannot. Concretely, its own docs list quantization recipes no other engine matches for breadth on NVIDIA silicon: NVFP4, MXFP4, FP8 per-tensor / block-scaling / rowwise, FP8 KV cache, NVFP4 KV cache, and W4A8 as well as W4A16 for both AWQ and GPTQ.

The hardware matrix is the load-bearing part, because it tells you which of those you can actually use on the GPUs you rent (reproduced from TensorRT-LLM's own `docs/source/features/quantization.md`):

| Architecture | NVFP4 | MXFP4 | FP8 per-tensor | FP8 block | FP8 rowwise | FP8 KV | NVFP4 KV | W4A8 AWQ | W4A16 AWQ | W4A16 GPTQ |
|---|---|---|---|---|---|---|---|---|---|---|
| Blackwell (sm100/103) | ✔ | ✔ | ✔ | ✔ | — | ✔ | ✔ | ✔ | ✔ | ✔ |
| Blackwell (sm120) | ✔ | ✔ | ✔ | — | — | ✔ | — | — | — | — |
| Hopper (sm90) | — | — | ✔ | ✔ | ✔ | ✔ | — | ✔ | ✔ | ✔ |
| Ada Lovelace (sm89) | — | — | ✔ | — | — | ✔ | — | ✔ | ✔ | ✔ |
| Ampere (sm80/86) | — | — | **—** | — | — | ✔ | — | — | ✔ | ✔ |

Read the Ampere row carefully, because it is the row people get wrong. **A100 has no FP8 compute path** (module 03 §15 derives why: no 8192-FLOP/SM/clock FP8 pipe in the 3rd-gen tensor core) — but it *can* store an FP8 KV cache, because that is a storage-and-convert operation, not a tensor-core matmul. "FP8 KV cache works on A100, FP8 weights do not" is a precise, checkable statement that sounds like a contradiction until you know the mechanism.

The cost side, stated honestly: a per-model, per-GPU-SKU, per-parallelism-configuration artifact to build, version, store and ship; a narrower model matrix; NVIDIA-only. Pick it when you serve a *small, stable, very-high-volume* set of models on NVIDIA and the last 20–40% of per-GPU throughput is worth an extra pipeline. That is a real condition that real companies meet — and one that a startup shipping a new model every fortnight does not.

### 7. Triton, Dynamo, KServe — the layers above, and when each earns its keep

**Triton Inference Server (L3)** hosts many *backends* behind one endpoint: TensorRT-LLM, vLLM, ONNX Runtime, PyTorch, plain Python. It adds dynamic batching for non-LLM models, model ensembles (chain a tokenizer → LLM → post-processor as one served unit), and concurrent model instances. **Pick it for a mixed zoo** — an LLM next to an embedding model, a reranker and a vision classifier — where you want one serving contract over heterogeneous frameworks. Do not pick it to make a single LLM faster; the LLM speed comes from the L2 engine underneath.

**NVIDIA Dynamo (L4)** is the datacenter-scale router: disaggregated prefill/decode orchestration, KV-cache-aware routing, and NIXL-based KV transfer, driving vLLM / TensorRT-LLM / SGLang workers underneath. It is the reference implementation of everything in §9–§12.

**KServe (L5)** is the Kubernetes control plane: an `InferenceService` CRD, declarative GitOps-friendly management, canary rollout, and a standard inference protocol across many models and tenants. Two deployment modes matter and people conflate them:

- **Standard Deployment** — a plain Deployment + HPA. GPU-resident. This is what KServe's own guidance steers GPU-heavy generative workloads toward.
- **Serverless (Knative)** — request-driven, scale-to-zero. Suits fast-starting predictive models far better than a multi-GB LLM, for the cold-start reasons lesson 09 quantifies.

Promising scale-to-zero for a 70B model "because KServe supports Knative" is a specific, common, and expensive mistake. Lesson 08 gives you the decision framework and lesson 09 gives you the numbers; the engine layer is not what makes it viable.

### 8. TGI — precisely what its status is

Be accurate here, because "TGI is dead" is close enough to be useful and wrong enough to be caught. TGI's own README carries a `CAUTION` banner stating the project is **in maintenance mode**: it accepts pull requests for minor bug fixes, documentation improvements and lightweight maintenance only. The same banner explicitly recommends **vLLM and SGLang** going forward, framing TGI's contribution as having pushed optimized engines toward relying on `transformers` model architectures — a job it considers done and inherited.

So: the repository is **not archived**, releases still tick over (v3.3.x), and a TGI deployment you inherit will keep running. What it will not do is grow: no new model architectures at the pace of vLLM/SGLang, no PD disaggregation, no new quantization formats. **Do not start new work on TGI. Recognise it in legacy stacks, and plan the migration to vLLM.**

One practical consequence for module 05 readers: TGI's metric names (`tgi_request_queue_duration`, `tgi_batch_current_size`, and the method-labelled `tgi_batch_forward_duration`) are what a legacy dashboard is built on, and they do not map one-to-one onto vLLM's. Migrating the engine means rewriting the recording rules, and that work is usually discovered late.

### 9. Disaggregation — the problem, and what it actually fixes

Recall the phase asymmetry. **Prefill is compute-bound**: arithmetic intensity ≈ prompt length, hundreds to thousands of FLOP/byte, far right of the roofline ridge. **Decode is memory-bandwidth-bound**: intensity ≈ batch size, one to a few hundred FLOP/byte, left of the ridge. In one fused vLLM instance they share the same GPUs *and the same iteration budget*, so they interfere in two specific ways:

1. **Head-of-line blocking.** An 8k-token prefill admitted into an iteration turns a ~5 ms decode step into a ~40 ms one. Every user already streaming sees an 8× ITL spike they did not cause. Chunked prefill bounds this (it is why vLLM enables it by default) but does not remove it — it converts one big spike into many small ones.
2. **One batch size for two jobs.** The `max_num_seqs` that maximises decode throughput is not the one that minimises prefill latency. Lesson 05's SLO knee is precisely the compromise this forces.

Disaggregation removes the coupling by removing the sharing:

```
  AGGREGATED (fused)  — one pool, one iteration budget, two jobs fighting
  ══════════════════════════════════════════════════════════════════════
   GPU pool ─┬─ iter k   [ decode A, decode B, decode C ]           5 ms
             ├─ iter k+1 [ decode A, decode B, PREFILL D 8k tok ]  40 ms  ◀── spike
             ├─ iter k+2 [ decode A, decode B, decode C, decode D ] 5 ms
             └─ …
                          A, B, C's tokens were 35 ms late. They did nothing wrong.


  DISAGGREGATED — two pools, two schedulers, KV moved between them
  ═══════════════════════════════════════════════════════════════
   client
     │  POST /v1/completions
     ▼
  ┌───────────────┐   1. route            ┌──────────────────────────────┐
  │  P/D ROUTER   │──────────────────────▶│  PREFILL POOL   (kv_producer)│
  │  (Dynamo /    │                       │  ┌────┐ ┌────┐ ┌────┐        │
  │   toy_proxy)  │◀── 2. first token ────│  │ P0 │ │ P1 │ │ P2 │        │
  └───────┬───────┘   + kv_transfer_params│  └────┘ └────┘ └────┘        │
          │                               │  BIG max_num_batched_tokens  │
          │                               │  TP tuned for compute        │
          │                               │  short-lived KV, held under  │
          │                               │  a lease (default 30 s)      │
          │                               └───────────┬──────────────────┘
          │                                           │
          │                        3. KV BLOCKS, layer by layer
          │                           NIXL over UCX → RDMA / NVLink
          │                           ~320 KiB/token for 70B @ bf16
          │                                           │
          │  4. stream tokens                         ▼
          │                               ┌──────────────────────────────┐
          └──────────────────────────────▶│  DECODE POOL    (kv_consumer)│
                                          │  ┌────┐ ┌────┐ … ┌────┐      │
                                          │  │ D0 │ │ D1 │     │ D8 │    │
                                          │  └────┘ └────┘ … └────┘      │
                                          │  BIG max_num_seqs            │
                                          │  KV-capacity-sized, not      │
                                          │  FLOP-sized                  │
                                          └──────────────────────────────┘

   ⇒ a huge prefill on P1 cannot steal an iteration from D3. There is no
     shared iteration to steal.
   ⇒ the two pools scale on DIFFERENT signals: prefill on input-token rate,
     decode on concurrent-sequence count.
```

**Now the part most write-ups get wrong.** vLLM's own `docs/features/disagg_prefill.md` states it plainly: *"Disaggregated prefill DOES NOT improve throughput."* The two documented reasons to do it are (a) tuning TTFT and ITL **separately**, because each pool gets its own parallelism strategy and batch settings, and (b) **controlling tail ITL**, because prefill can no longer be injected into a decode iteration. The doc even notes that chunked prefill with a well-chosen chunk size achieves the same tail-ITL goal — disaggregation is just a far more *reliable* way to get it, since finding the right chunk size in practice is hard.

Reconcile that with Mooncake's "59–498% effective request capacity" and it is not a contradiction: **capacity under an SLO** (goodput, module 05 §6) is a different quantity from raw tokens/s. Disaggregation lets you run each pool at a much more aggressive operating point *without* the interference that would have blown the SLO, so SLO-compliant capacity rises even when raw throughput does not. Saying that sentence correctly is a strong interview signal; saying "disaggregation gives 5× throughput" is a weak one.

### 10. How the KV actually moves — vLLM's connector model

Disaggregation is not a mode you flip on; it is a *connector* you configure on two instances that otherwise look normal. vLLM v0.27.1 ships nine connector types, including `NixlConnector` (fully async send/recv over NIXL), `MooncakeConnector`, `LMCacheConnectorV1`, `MultiConnector` (an ordered list of connectors), and `OffloadingConnector` (CPU/filesystem tiers rather than a peer GPU).

The minimum viable NIXL setup, from vLLM's own usage guide — a producer, a consumer, and a proxy:

```bash
# ── PREFILL instance (kv_producer) ─────────────────────────────────────
CUDA_VISIBLE_DEVICES=0 \
UCX_NET_DEVICES=all \
VLLM_NIXL_SIDE_CHANNEL_PORT=5600 \
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8100 \
  --kv-transfer-config '{"kv_connector":"NixlConnector",
                         "kv_role":"kv_producer",
                         "kv_load_failure_policy":"fail",
                         "kv_connector_extra_config":{"kv_lease_duration":30}}'

# ── DECODE instance (kv_consumer) ──────────────────────────────────────
CUDA_VISIBLE_DEVICES=1 \
UCX_NET_DEVICES=all \
VLLM_NIXL_SIDE_CHANNEL_PORT=5601 \
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8200 \
  --kv-transfer-config '{"kv_connector":"NixlConnector",
                         "kv_role":"kv_consumer",
                         "kv_load_failure_policy":"fail"}'

# ── The P/D proxy that pairs them ──────────────────────────────────────
python tests/v1/kv_connector/nixl_integration/toy_proxy_server.py \
  --port 8192 \
  --prefiller-hosts localhost --prefiller-ports 8100 \
  --decoder-hosts  localhost --decoder-ports  8200
```

Four details in there carry operational weight:

- **`VLLM_NIXL_SIDE_CHANNEL_PORT`** (default 5600) is the handshake channel, **required on both sides**. Each worker needs a unique port *on its host*; the same number on different hosts is fine. Under data parallelism the port is `base_port + dp_rank`. Forgetting this is the most common "the handshake hangs" failure.
- **`VLLM_NIXL_SIDE_CHANNEL_HOST`** (default `localhost`) must be set when the pools are on different machines — which, in production, they always are.
- **`kv_lease_duration`** (default **30 s**) is how long the prefiller holds a finished request's KV blocks waiting for the decoder to read them. Heartbeats extend the lease while the request is queued on the decoder; if neither a heartbeat nor a read arrives before expiry, the blocks are freed and the request fails or falls back. **A decode pool that is saturated will queue requests past the lease**, and the symptom is transfer failures that look like a network problem but are really a *capacity* problem on the far side. That coupling is the operational cost of two pools.
- **Transport selection is a NIXL backend choice**, defaulting to UCX. Configure it with `kv_connector_extra_config.backends` (`["UCX"]`, `["GDS"]`, `["LIBFABRIC"]`) and tune UCX via `UCX_TLS` / `UCX_NET_DEVICES` — **not** via NCCL variables. `NCCL_IB_HCA` and `NCCL_SOCKET_IFNAME` have no effect on NixlConnector, and setting them and expecting a change is a genuinely wasted afternoon.

One design detail worth knowing because it changes the arithmetic in §11: vLLM's worker-side connector performs **layer-by-layer** KV store and load, interleaved with the attention module. The transfer of layer *n*'s KV overlaps the compute of layer *n+1*. So the raw byte-time is an upper bound on the *exposed* latency, not the exposed latency itself — which is exactly why the ratio below is a screening test rather than a prediction.

### 11. The interconnect arithmetic — the make-or-break calculation

This is the most useful thing in the lesson. It answers "will disaggregation help on my fabric?" before you provision anything.

**Step 1 — bytes to move.** Module 05 §5's formula, per token of prompt:

```
  kv_bytes_per_token = 2 (K and V) × layers × kv_heads × head_dim × bytes_per_elem

  Llama-3.1-8B  (32 layers,  8 KV heads [GQA], head_dim 128, bf16):
      2 × 32 × 8 × 128 × 2 = 131,072 B  = 128 KiB/token
  Llama-3.1-70B (80 layers,  8 KV heads [GQA], head_dim 128, bf16):
      2 × 80 × 8 × 128 × 2 = 327,680 B  = 320 KiB/token
  Llama-3.1-70B, same but FP8 KV cache:                    160 KiB/token
```

**Step 2 — transfer time and prefill time, for a prompt of `T` tokens.**

```
  transfer_time =  T × kv_bytes_per_token
                   ──────────────────────
                        B_link                    [bytes/s of the fabric]

  prefill_time  =  2 × P × T                      [P = parameter count]
                   ─────────────────────
                   N_gpu × FLOPS × MFU            [prefill pool's compute]
```

**Step 3 — take the ratio, and notice what cancels.**

```
  transfer_time     kv_bytes_per_token × N_gpu × FLOPS × MFU
  ─────────────  =  ────────────────────────────────────────
   prefill_time              B_link × 2 × P

                     ▲
                     └── T is GONE. The prompt length cancels.
```

**The transfer-to-prefill ratio does not depend on prompt length.** It is a fixed property of your model, your prefill pool's parallelism, and your fabric. That is why this is a screening test you can run on a whiteboard.

**Step 4 — plug in real hardware.** Llama-3.1-70B (`P = 70e9`, 320 KiB/token bf16 KV), prefill pool of **8× H100** at FP8 dense peak 1,979 TFLOP/s each, assume 50% MFU on prefill:

```
  numerator = 327,680 B × 8 × 1.979e15 FLOP/s × 0.50 = 2.594e21
  denominator = B_link × 2 × 70e9                    = B_link × 1.4e11

  ratio = 1.853e10 / B_link
```

| Fabric | Link bandwidth (bytes/s) | transfer ÷ prefill | Verdict |
|---|---|---|---|
| NVLink 4 intra-node (~450 GB/s/dir) | 4.5e11 | **0.04** | Free. Disaggregation is essentially untaxed. |
| InfiniBand NDR 400 Gb/s | 5.0e10 | **0.37** | Viable. 37% overhead, and layer-wise overlap hides much of it. |
| InfiniBand HDR 200 Gb/s | 2.5e10 | **0.74** | Marginal. You are paying nearly a second prefill in transfer. |
| 100 GbE (no RDMA) | 1.25e10 | **1.48** | **Broken.** Transfer costs more than the prefill it replaced. |
| 25 GbE | 3.13e9 | **5.9** | Catastrophic. TTFT is now dominated by the network. |

Three readings of that table that are worth more than the table:

1. **RDMA is not a nice-to-have, it is the precondition.** The line between "viable" and "broken" sits between 200 Gb/s IB and 100 GbE. Every production disaggregation disclosure — Mooncake, DeepSeek, LMSYS's 96-H100 reproduction — runs on RDMA fabric. That is not vendor preference; it is this ratio.
2. **More prefill parallelism makes disaggregation *harder* to justify, not easier.** `N_gpu` is in the numerator: a TP=8 prefill pool finishes the prefill 8× faster while the KV volume is unchanged, so the transfer's *relative* cost goes up 8×. Counterintuitive, correct, and a great follow-up question to be ready for.
3. **An FP8 KV cache halves the numerator**, dropping the 400 Gb/s IB case from 0.37 to 0.19. Lesson 07's quantization lever is also a *disaggregation-feasibility* lever — the two compound.

Finally, the honest caveat: because vLLM transfers KV **layer by layer, overlapped with compute**, the exposed latency is materially below the raw ratio. Treat these numbers as *"is this within an order of magnitude of workable"*, not as a forecast. A ratio of 0.04 means stop worrying; 1.48 means stop planning.

### 12. KV-cache-aware routing — RadixAttention's idea, lifted to the fleet

A round-robin load balancer treats replicas as interchangeable. For a prefix-caching engine they are emphatically not: replica 3 may hold the 3,500-token preamble's KV in its radix tree while replica 7 does not. Sending the request to 7 turns a free cache hit into 3,500 tokens of prefill.

A **KV-aware router** hashes the request's prefix, tracks which worker holds which prefix blocks, and routes for *cache affinity* — subject to a load ceiling, so a single hot prefix does not pin all traffic to one worker. It is the same insight as §4 (KV is shareable) applied one layer up (KV is *locatable*).

```
  NAIVE ROUND-ROBIN                    KV-AWARE ROUTING
  ─────────────────                    ────────────────
   req(preamble+q1) ──▶ W1              req(preamble+q1) ──▶ W1  [prefix P: miss→fill]
   req(preamble+q2) ──▶ W2              req(preamble+q2) ──▶ W1  [prefix P: HIT ]
   req(preamble+q3) ──▶ W3              req(preamble+q3) ──▶ W1  [prefix P: HIT ]
   req(preamble+q4) ──▶ W1              req(preamble+q4) ──▶ W2  [W1 at load cap →
                                                                  seed P on W2   ]
   4 requests, 4 cold prefills          4 requests, 2 cold prefills
   cache hit rate 0%                    cache hit rate 50% and climbing
```

The load-ceiling term is not optional. Pure affinity routing on a workload with one dominant prefix produces a perfectly cached, perfectly overloaded single replica while the rest idle — a textbook hot-shard. Any router worth deploying blends prefix-match length with current queue depth. This is exactly the mechanism behind Baseten's published Dynamo results (§Real-world use cases), which is why they pair *latency* improvements with *throughput* improvements: fewer prefills is both.

### 13. When NOT to disaggregate — the scale threshold, with a reason

Disaggregation carries fixed costs that do not shrink with your fleet:

- **An RDMA fabric** you must provision, tune (`UCX_TLS`, `UCX_NET_DEVICES`) and monitor.
- **Two autoscalers** on two different signals, plus the lease-expiry coupling from §10 that makes decode-pool saturation appear as prefill-side transfer failures.
- **Two failure domains** and a router that must handle each independently.
- **A P/D router** in the request path — one more hop to instrument, one more thing to page on.

Those costs are roughly constant. The benefit scales with traffic. So there is a break-even fleet size, and every public disclosure sits far above it: DeepSeek runs prefill units of 4 nodes (EP32) and decode units of 18 nodes (EP144); LMSYS's open reproduction used 12 nodes / 96 H100s split 3 prefill : 9 decode; Mooncake runs across thousands of nodes. **Proposing disaggregation for a 4-GPU service is the canonical junior mistake in this area** — and the question that exposes it is always *"at what GPU count does the fabric and orchestration overhead pay for itself here?"*

The order of levers for a small fleet, cheapest first: prefix caching (already on) → chunked prefill tuning via `max_num_batched_tokens` (module 05 §12's dial) → a bigger GPU or FP8 → more replicas → *then* disaggregation.

### 14. A two-year research lineage, one continuous idea

Worth naming explicitly, because it is a strong interview answer on its own and it explains why the tools look the way they do:

```
  2023 ── PagedAttention (vLLM)        KV is a MANAGED resource
          block allocation, block      "stop wasting it to fragmentation"
          tables, no contiguity                │
                                               ▼
  2024 ── RadixAttention (SGLang)      KV is a SHARED resource
          radix tree over prefixes,    "stop recomputing what you already have"
          LRU-over-leaves eviction              │
                                               ▼
  2024 ── DistServe / Mooncake /       KV is a TRANSPORTABLE resource
  –2025   DeepSeek P/D, Dynamo         "stop making one pool do two jobs"
          NIXL, KV-aware routing
```

Three steps, each solving the previous one's residual limitation, each now running in production at real scale. The through-line — *the KV cache is the resource worth architecting around, not the GPU* — is the thesis of this entire module.

## Perspectives

**The frontier-lab-scale view.** DeepSeek's disclosed production configuration runs prefill on 4-node/EP32 deployment units and decode on 18-node/EP144 units on H800s, sustaining roughly 73.7k input tok/s and 14.8k output tok/s per node. Moonshot AI's Mooncake reports 59–498% effective-capacity gains over non-disaggregated baselines while still meeting SLOs, across thousands of nodes and over 100 billion tokens/day. At that scale disaggregation is close to structural necessity, because a fused topology cannot reach the throughput-per-dollar these fleets need. Note the *shape* of the split — 4 prefill nodes to 18 decode nodes, roughly 1:4.5 — which is itself the answer to "what does independent scaling buy you": a ratio you could never have hit with one pool.

**The platform-vendor view.** Baseten is a mid-size inference platform, not a frontier lab, and its published Dynamo results are the more realistic story for what a Staff Platform Engineer at a normal-sized shop ships: concrete P95/P99/RPS deltas from adopting disaggregation plus KV-aware routing on an *existing* fleet, not a from-scratch hyperscale build. "Should we turn this on for the fleet we have" is the decision you will actually face.

**The engine-maintainer view.** vLLM's own docs saying "disaggregated prefill DOES NOT improve throughput" is the most useful sentence in this lesson, and it comes from the people with the least incentive to undersell their own feature. When a project's documentation contradicts the marketing narrative around it, the documentation is the one to trust — and being the person in the room who has read it is disproportionately valuable.

**The vendor-neutral engine-selection view.** Published engine leaderboards decay fast in a space this competitive, and they decay *asymmetrically*: a benchmark run against vLLM v0.6 and SGLang v0.4 is not merely old, it is old in a direction. Treat any comparison article as a starting hypothesis to re-run on your own workload — the same discipline lesson 05 demanded of a CPM number. The specific tell: if a 2026-dated comparison still lists TGI favourably without mentioning maintenance mode, it was not re-verified.

**The FinOps view.** Every technique in this lesson is a *fixed-cost-vs-variable-cost* trade dressed up as an architecture decision. Prefix caching converts variable prefill compute into fixed memory. Disaggregation converts a variable interference tax into fixed fabric and orchestration cost. TensorRT-LLM converts variable per-GPU throughput into fixed build-pipeline engineering. In each case the right question is the same one: *how much traffic do I have to amortise the fixed part against?*

## Real-world use cases

- **vLLM — `docs/features/disagg_prefill.md`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/docs/features/disagg_prefill.md>. Documents the two motivations (tune TTFT/ITL separately; control tail ITL), states explicitly that disaggregated prefill **does not improve throughput**, notes that a well-chosen chunked-prefill chunk size achieves the same tail-ITL goal but is hard to tune in practice, and enumerates the nine supported connectors. **What it shows:** the authoritative correction to the most common misconception about this technique, from the engine's own maintainers — and the fact that the feature is still marked *experimental* in the engine most people would deploy it on.

- **Baseten — "2x faster inference with KV cache-aware routing"** (2025), built on NVIDIA Dynamo — <https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/>. Reported: **48% decrease in P95 latency, 49% decrease in P99, 61% more requests/sec and 62% more output tokens/sec** on a long-context workload from combining disaggregated serving with KV-aware routing; up to 5× as many requests on the same GPU count for some workloads. **What it shows:** a named platform vendor's own production numbers with the specific percentiles a Staff engineer would be asked to defend — and the pairing of latency *and* throughput gains, which is the KV-aware-routing signature (fewer cold prefills improves both at once).

- **Moonshot AI — Mooncake, the serving system behind Kimi** (USENIX FAST '25, arXiv 2407.00079) — <https://arxiv.org/abs/2407.00079> · <https://www.usenix.org/system/files/fast25-qin.pdf>. A KVCache-centric disaggregated architecture separating prefill and decode clusters and additionally pooling underused CPU/DRAM/SSD/NIC capacity as a KV cache tier. Reports **59–498% effective request-capacity increase** versus non-disaggregated baselines on real production traces while continuing to meet SLOs; thousands of nodes, 100B+ tokens/day. **What it shows:** the deepest public account of disaggregation economics at genuine hyperscale, and the clearest evidence that the *KV cache* — not GPU count — is the resource worth architecting around. Note the metric is capacity **under SLO**, not raw throughput, which is exactly how it reconciles with vLLM's "no throughput gain" statement.

- **DeepSeek — V3/R1 Inference System overview** (Open Source Week, 2025) — <https://github.com/deepseek-ai/open-infra-index/blob/main/202502OpenSourceWeek/day_6_one_more_thing_deepseekV3R1_inference_system_overview.md>. Discloses the production split directly: prefill deployment units of 4 nodes (Expert-Parallel-32), decode units of 18 nodes (EP144), on H800s, roughly 73.7k input tok/s and 14.8k output tok/s per node. **What it shows:** a frontier lab publishing its own real topology rather than a benchmark — and the 1:4.5 prefill:decode node ratio, which is the concrete form of "independent scaling."

- **LMSYS Org — "Deploying DeepSeek with PD Disaggregation and Large-Scale Expert Parallelism on 96 H100 GPUs"** (2025) — <https://www.lmsys.org/blog/2025-05-05-large-scale-ep/>. A fully open-source SGLang reproduction: 12 nodes / 96 H100s split 3 prefill : 9 decode, reaching 52.3k input tok/s and 22.3k output tok/s per node — close to DeepSeek's own figures — at an implied self-hosted cost around **$0.20 per million output tokens**, roughly a fifth of DeepSeek's own API price at the time. **What it shows:** the paper-to-practice bridge, with public code, and a directly usable reference architecture. Also the clearest single data point on the *scale threshold*: 96 GPUs, not 4.

- **Hugging Face — TGI README maintenance-mode banner** — <https://github.com/huggingface/text-generation-inference>. A `CAUTION` block stating the project accepts only minor bug fixes, documentation improvements and lightweight maintenance, and recommending vLLM and SGLang going forward. **What it shows:** the primary source for this lesson's TGI guidance — and a correction to the looser "TGI is archived/dead" claim: the repo is live and still releasing (v3.3.x), it is simply frozen in scope. Precision matters when someone in the room owns a TGI deployment.

## Worked example

**The workload.** A customer-support RAG assistant. Every request carries a **3,500-token shared preamble** (system prompt + retrieved policy docs) plus a short user turn, and generates a ~200-token answer. Steady state is 300 concurrent sessions. Model: Llama-3.1-70B. SLO: TTFT-p99 ≤ 800 ms, TPOT-p99 ≤ 50 ms.

### Step 1 — characterise the shape

```
  input:output ratio      = 3,700 : 200  ≈ 18.5 : 1     → strongly PREFILL-heavy
  prefix overlap          = 3,500 / 3,700 ≈ 95%          → strongly SHAREABLE
  prefix structure        = one linear preamble, no fan-out
  KV per request at end   = 3,900 tok × 320 KiB = 1.22 GB   (bf16)
```

Both properties matter and they point at *different* levers. Prefill-heavy says "disaggregation is worth evaluating." 95% prefix overlap says "prefix caching may make the prefill problem disappear entirely, in which case disaggregation is solving a problem you no longer have." **Evaluate the cheap lever first.**

### Step 2 — price the prefix cache

Without caching, every request prefills 3,700 tokens:

```
  FLOPs/request = 2 × 70e9 × 3,700 = 5.18e14 = 518 TFLOP
  On 4× H100 at FP8 (1,979 TFLOP/s dense each) and 50% MFU:
      518e12 / (4 × 1.979e15 × 0.50) = 131 ms of pure prefill compute
```

With a warm prefix cache, only the ~200 unique tokens prefill:

```
  FLOPs/request = 2 × 70e9 × 200 = 2.8e13 = 28 TFLOP
      28e12 / (4 × 1.979e15 × 0.50) = 7.1 ms          ← 18× less prefill compute
```

**That is an 18× reduction in prefill work for a configuration flag.** At 95% overlap, prefix caching is not an optimization, it is the design. Every other decision in this example is downstream of it.

### Step 3 — does disaggregation still pay?

Run §11's ratio for this deployment. Prefill pool 4× H100, 70B, bf16 KV (320 KiB/token), 50% MFU, 400 Gb/s InfiniBand (5.0e10 B/s):

```
  ratio = (327,680 × 4 × 1.979e15 × 0.50) / (5.0e10 × 2 × 70e9)
        = 1.297e21 / 7.0e21
        = 0.185      → 18.5% transfer overhead relative to prefill
```

Workable on IB. But now stack it against Step 2: with the prefix cache warm, the *prefill you are disaggregating away* is 7.1 ms, while the KV you must transfer is still the **full 1.22 GB** — the cache hit saved you the compute, not the bytes. On 400 Gb/s that transfer is `1.22e9 / 5.0e10 = 24 ms`, i.e. **3.4× the prefill it was meant to unblock.**

```
  WITHOUT prefix caching:   prefill 131 ms   transfer 24 ms   → ratio 0.19  ✔
  WITH    prefix caching:   prefill 7.1 ms   transfer 24 ms   → ratio 3.4   ✘
```

**Prefix caching and disaggregation partly cannibalise each other on this workload**, and the direction is not obvious until you do the arithmetic. Once the shared preamble is a cache hit, there is very little prefill left to move off the decode GPUs, and the KV transfer becomes the dominant term. This is precisely the kind of second-order interaction that separates a defensible design from a list of good ideas stapled together.

*(Caveat, stated because it is load-bearing: vLLM's layer-by-layer overlapped transfer reduces the exposed cost below the raw 24 ms, and a disaggregation-aware deployment can keep prefix caches on the prefill pool so the transfer is the only cost. The conclusion here is "the case is much weaker than it looked, measure it," not "it is impossible.")*

### Step 4 — the decision

```
  ENGINE      vLLM (prefix caching on by default) or SGLang.
              Prefix structure is LINEAR, not branching, so SGLang's radix
              tree has no structural advantage here. Choose on operational
              grounds: vLLM if the rest of the platform is vLLM.

  ROUTING     KV-aware / prefix-affinity routing across replicas — MANDATORY.
              Round-robin across N replicas divides the hit rate by ~N until
              every replica has independently paid for the preamble.
              This is the single highest-leverage decision in the design.

  TOPOLOGY    FUSED, not disaggregated. Step 3 shows the case is weak once
              caching lands, and the fleet is nowhere near the ~100-GPU
              threshold where the fabric and two-autoscaler cost amortises.

  PLATFORM    KServe Standard Deployment (not Knative/serverless): 70B weights
              make cold start prohibitive (lesson 09). minReplicaCount ≥ 2.

  BATCH       max_num_batched_tokens sized for the 200-token uncached tail,
              not the 3,700-token nominal prompt — a cache hit means the
              scheduler only ever sees the tail.
```

### Step 5 — the contrast case

Same team, different job: **nightly ticket summarisation**. Every ticket unique, no shared prefix, no human waiting.

```
  prefix overlap  ≈ 0%      → RadixAttention buys nothing; APC buys nothing
  latency SLO     = none    → run at the throughput-maximising batch, not the knee
  ENGINE          = vLLM, max_num_seqs pushed to the KV ceiling
  TOPOLOGY        = fused; disaggregation is unjustified overhead
  PLATFORM        = a Job, not a Service. Scale to zero between runs — and this
                    time the cold start genuinely does not matter (lesson 08).
```

**Same team, same model, same cluster, two different right answers.** That is the entire point of the decision table: "which engine is fastest" has no answer until you say for what.

## Practice

Reuse the lesson-05 harness and rented GPU. Two goals: prove the prefix-sharing effect in CPM terms, and produce the engine decision matrix for the deliverable. Pin **vLLM v0.27.1** and record the version in every artifact.

**1. Build a shared-prefix dataset and a control.** Generate ~500 prompts that all begin with the *same* ~2,000-token preamble followed by a short unique tail, and a matched control set of ~500 prompts with no shared prefix and the same total token count. Hold output length fixed at 128. The control is not optional — it is what turns "SGLang was faster" into "SGLang was faster *because of prefix reuse*."

**2. Baseline on vLLM, prefix caching off then on:**

```bash
# Prefix caching is ON by default in v0.27.x — turn it OFF for the baseline.
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 --max-num-seqs 128 --no-enable-prefix-caching

vllm bench serve --backend vllm --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 \
  --dataset-name sharegpt --dataset-path ./shared_prefix.json \
  --num-prompts 500 --max-concurrency 64 \
  --percentile-metrics ttft,tpot --metric-percentiles 50,99 \
  --save-result --result-filename vllm_noapc.json

# Then repeat with caching on (drop the flag) → vllm_apc.json
```

While each run is in flight, capture the prefix-cache counters (module 05 §7) so the hit rate is measured, not assumed:

```promql
rate(vllm:prefix_cache_hits_total[1m]) / rate(vllm:prefix_cache_queries_total[1m])
```

These are counted in **tokens**, not requests. On the shared-prefix set you should see the ratio climb toward ~0.95; on the control it should sit near zero. **If it does not, stop and fix that before drawing any conclusion from the throughput numbers** — a flat hit rate with a throughput change means something else moved.

**3. Same two datasets on SGLang:**

```bash
python -m sglang.launch_server \
  --model-path meta-llama/Llama-3.1-8B-Instruct --port 30000 --enable-metrics

python -m sglang.bench_serving --backend sglang \
  --dataset-path ./shared_prefix.json --num-prompts 500 --max-concurrency 64
```

Read `sglang:cache_hit_rate` and `sglang:token_usage` alongside, so both engines' hit rates are on the record.

**4. Compute CPM** for all four runs (2 engines × 2 datasets) with the lesson-05 equation and your GPU's real hourly rate. **The expected shape:** a large vLLM-APC-on vs APC-off gap on the shared-prefix set, a much smaller vLLM-vs-SGLang gap on the same set, and *both* gaps collapsing to noise on the control set.

**5. Run the §11 ratio for a hypothetical disaggregation of this same model** on three fabrics (NVLink intra-node, 400 Gb/s IB, 25 GbE) and write the one-sentence verdict for each. This is a paper exercise — you are not provisioning RDMA for a course — but the arithmetic is the deliverable.

**Acceptance.**
- A **four-cell CPM table** (vLLM-APC-off / vLLM-APC-on / SGLang, × shared-prefix / control), each cell carrying its **measured prefix-cache hit rate**, added to the cost-per-token deliverable. A CPM number without its hit rate does not pass — that pairing is the whole experiment.
- A **one-page engine decision matrix**: rows = {vLLM, SGLang, TensorRT-LLM, Triton, KServe, TGI}, columns = {layer in the L1–L5 stack, best workload shape, key mechanism, when to pick, when NOT to}, with TGI explicitly marked "maintenance mode — do not start new work."
- A **three-row interconnect table** from step 5 with the transfer÷prefill ratio and verdict per fabric, plus the GPU count at which you would revisit the decision.

## Common pitfalls

1. **Comparing engines that live at different layers.** "vLLM or KServe?" and "Triton or vLLM?" are malformed. *Mechanism:* §1 — KServe is a control plane that runs an engine; Triton is a multi-backend runtime whose backends include vLLM. Fix the layer, then compare.

2. **Recommending SGLang for prefix sharing without checking whether vLLM's APC already covers the pattern.** *Mechanism:* vLLM ships prefix caching on by default; the residual gap is specifically *branching* reuse (trees), not a single linear shared prefix. Recommending an engine migration for a benefit you already have is a stale-information error with real migration cost attached.

3. **Believing disaggregation raises throughput.** *Mechanism:* vLLM's own docs say it does not. It separates TTFT and ITL tuning and controls tail ITL; the big published capacity numbers are **goodput under SLO**, which rises because you can now run each pool harder without interference. Quoting "5× throughput" is the tell that someone read a headline and not a doc.

4. **Disaggregating without RDMA.** *Mechanism:* §11. At 100 GbE the ratio is 1.48 and at 25 GbE it is 5.9 — the transfer costs more than the prefill it was supposed to unblock, so TTFT gets *worse* and the failure looks like "the new architecture is slow" rather than "the fabric is wrong." Run the ratio before you provision.

5. **Disaggregating a 4-GPU service.** *Mechanism:* §13 — an RDMA fabric, two autoscalers, two failure domains and a router are roughly fixed costs, and every public disclosure that justifies them sits at 96–1000+ GPUs. Ask "at what GPU count does this pay for itself?" and if the answer is larger than the fleet, the answer is no.

6. **Setting NCCL environment variables to tune a NixlConnector transfer.** *Mechanism:* §10 — NIXL's default transport is UCX, so `NCCL_IB_HCA` and `NCCL_SOCKET_IFNAME` are inert. The knobs are `UCX_TLS`, `UCX_NET_DEVICES`, and `kv_connector_extra_config.backends`. Symptom: hours of tuning with zero measurable change.

7. **Missing the `kv_lease_duration` coupling.** *Mechanism:* §10 — the prefiller holds finished KV for 30 s by default, extended by heartbeats while the decoder queues. A saturated decode pool queues past the lease, the blocks are freed, and you get transfer failures on the *prefill* side whose real cause is capacity on the *decode* side. Alert on decode queue depth, not just on transfer errors.

8. **Promising Knative scale-to-zero for a large generative model because KServe supports it.** *Mechanism:* KServe's own guidance steers GPU-heavy generative workloads to Standard Deployment; lesson 09 quantifies why (weight-load time dominates and is storage-bound).

9. **Round-robin routing in front of prefix-caching replicas.** *Mechanism:* §12 — the hit rate is divided by roughly the replica count until each replica independently pays for the preamble, and it degrades further every time a replica is recycled. You bought a prefix cache and then load-balanced it away.

10. **Citing an engine leaderboard as ground truth because it looks neutral.** *Mechanism:* these projects ship weekly; a benchmark is a dated snapshot of two moving targets. Re-run the comparison on your workload — the same discipline lesson 05 demanded of any CPM number.

## Self-check

**(a) When does RadixAttention / prefix-sharing NOT help — what workload?**

**Answer:** Workloads with **unique, non-overlapping prompts**: batch summarisation of distinct documents, translation of independent inputs, classification of unrelated records — any high-entropy prompt stream. With nothing shared across requests there is nothing to cache, so the radix tree does tree maintenance (match, split, LRU heap operations) for a zero hit rate. vLLM measured its own prefix caching at **under 1% throughput cost at 0% hit rate**, so leaving it enabled is safe, but SGLang and vLLM land at essentially the same CPM and the choice moves to operational grounds. Prefix sharing pays only when requests actually reuse *leading* tokens — a shared system prompt, a few-shot exemplar block, or conversation history. And note the second-order case from the worked example: even at 95% overlap, prefix caching removes the *compute*, not the *bytes*, which is why it weakens rather than strengthens the case for disaggregation.

**(b) Why disaggregate prefill from decode into separate pools — what does it let you scale independently, and what does it NOT buy?**

**Answer:** Prefill is compute-bound (intensity ≈ prompt length, sets TTFT); decode is memory-bandwidth-bound (intensity ≈ batch size, sets TPOT). Fused in one instance they share an iteration budget, so an 8k prefill turns a 5 ms decode step into a ~40 ms one for everyone in the batch, and one `max_num_seqs` has to serve two conflicting optima. Separate pools let you scale **prefill capacity to input-token rate and decode capacity to concurrent-sequence count independently** — two autoscalers, two operating points, two parallelism strategies, even two GPU SKUs. DeepSeek's disclosed 4-node-prefill : 18-node-decode ratio is what that independence looks like in production. **What it does not buy is raw throughput** — vLLM's own docs state that explicitly. The documented benefits are tuning TTFT and ITL separately and controlling tail ITL. The large published capacity gains (Mooncake's 59–498%) are capacity **under SLO**, i.e. goodput: you can run each pool harder without interference. The cost is moving KV across the network, which only works over RDMA/NVLink (see (e)), and only amortises at scale.

**(c) Which engine for a Kubernetes inference PLATFORM serving many models, and why is the question slightly wrong?**

**Answer:** **KServe** — but the question conflates layers. KServe is not an engine; it is the L5 control plane (an `InferenceService` CRD) that *orchestrates* an engine, giving you declarative GitOps management, per-model autoscaling, canary rollout, and a standard inference protocol across many models and tenants. An engine alone (vLLM/SGLang/TRT-LLM) serves one model well and supplies no platform layer; KServe supplies that and runs those engines underneath. For a mixed zoo — LLM plus embeddings plus rerankers plus vision — you put **Triton** (L3, multi-backend) inside the KServe `InferenceService`. And note KServe's own guidance: use **Standard Deployment** for GPU-heavy generative workloads, not the Knative/serverless path, whose scale-to-zero assumption breaks on multi-GB weight loads.

**(d) A colleague proposes disaggregating prefill/decode for a 4-GPU RAG service. What's the objection?**

**Answer:** The fixed costs do not shrink with the fleet: an RDMA fabric to provision and tune, two autoscalers on two signals, two failure domains, a P/D router in the request path, and the `kv_lease_duration` coupling that turns decode saturation into prefill-side transfer failures. Every public result that justifies those costs comes from a far larger fleet — DeepSeek's 4-node prefill / 18-node decode units, LMSYS's 96 H100s, Mooncake's thousands of nodes. At 4 GPUs there is nothing to amortise against. Additionally, for a *RAG* service specifically, prefix caching is the cheaper and larger lever (the worked example shows 18× less prefill compute at 95% overlap), and it partly removes the very prefill load disaggregation was meant to relocate. The right question back is **"at what GPU count does the fabric and orchestration overhead pay for itself here?"** — and the ordered ladder before disaggregation is: prefix caching → `max_num_batched_tokens` tuning → FP8 → more replicas.

**(e) Derive whether disaggregation is viable on a given fabric, from first principles.**

**Answer:** Transfer time is `T × kv_bytes_per_token / B_link`; prefill time is `2 × P × T / (N_gpu × FLOPS × MFU)`. Take the ratio and **`T` cancels** — the transfer-to-prefill ratio is independent of prompt length, so it is a fixed property of model, prefill parallelism and fabric:

```
  ratio = (kv_bytes_per_token × N_gpu × FLOPS × MFU) / (B_link × 2 × P)
```

For Llama-3.1-70B (320 KiB/token bf16 KV), an 8× H100 FP8 prefill pool at 50% MFU: NVLink-4 → 0.04 (free); 400 Gb/s IB → 0.37 (viable); 100 GbE → 1.48 (transfer costs more than the prefill); 25 GbE → 5.9 (catastrophic). **RDMA is the precondition, not a preference.** Two non-obvious corollaries: (1) *more* prefill parallelism makes disaggregation harder to justify, because `N_gpu` is in the numerator — a TP=8 pool finishes prefill 8× faster while the KV volume is unchanged; (2) an FP8 KV cache halves the numerator, so lesson 07's quantization lever is also a disaggregation-feasibility lever. Caveat: vLLM transfers KV layer-by-layer overlapped with compute, so the exposed cost is below the raw ratio — treat this as a screening test, not a forecast.

**(f) What is TGI's actual status, and what does that mean operationally?**

**Answer:** **Maintenance mode**, per a `CAUTION` banner in its own README: only minor bug fixes, documentation improvements and lightweight maintenance are accepted, and the banner recommends vLLM and SGLang going forward. The repository is **not archived** and still cuts releases (v3.3.x), so an inherited deployment keeps working — but it will not gain new model architectures, PD disaggregation, or new quantization formats. Operationally: do not start new work on it; plan a migration; and budget for the fact that migrating the engine means **rewriting your dashboards and alerts**, because TGI's `tgi_*` metric names do not map one-to-one onto vLLM's `vllm:*` names (module 05 §7 has the mapping table). Being precise about "maintenance mode, not archived" is worth doing — someone in the room usually owns one.

## Connections & what's next

This lesson closes the loop lesson 05 opened: the CPM curve there assumed one engine and one fused pool; here you learned when that assumption itself is the thing to change — a different engine for a different prefix-overlap profile, or two pools instead of one for a different phase balance, gated by an interconnect calculation you can do on a whiteboard. It also completes the KV-cache arc running since module 03: PagedAttention (lesson 03) made the KV cache a *managed* resource inside one engine; RadixAttention (§4) made it *shareable* across requests within that engine; disaggregation and KV-aware routing (§9–§12) made it a fleet-wide *transportable* resource.

It also plants two threads the next three lessons pick up. The `kv_lease_duration` coupling in §10 is a preview of lesson 08's core theme — control loops whose failure mode is a *timing* mismatch, not a capacity one. And §11's FP8-KV-halves-the-transfer observation is the first place quantization stops being a per-GPU numerics decision and starts being an architecture enabler.

Next: [**07.7 · Quantization ops**](07-quantization-ops.md). It is largely orthogonal to everything here — the numeric format is a per-GPU decision that stacks on top of whichever engine and topology you chose — so expect the deliverable to combine both: your engine and topology choice from this lesson, quantized, swept across batch sizes as in lesson 05.

## References & further reading

**Primary sources (verified against the repository at the versions stated)**

1. **vLLM — `docs/features/disagg_prefill.md`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/docs/features/disagg_prefill.md> — the two documented motivations, the explicit "**disaggregated prefill DOES NOT improve throughput**" statement, the note that chunked prefill can achieve the same tail-ITL goal but is hard to tune, the nine connector types, and the Connector / LookupBuffer / Pipe abstractions with layer-by-layer KV store and load. **Correction to earlier versions of this lesson:** disaggregation was previously framed as a throughput technique; the engine's own docs say it is a latency-shaping and interference-removal technique, and the large published gains are goodput-under-SLO.
2. **vLLM — `docs/features/nixl_connector_usage.md`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/docs/features/nixl_connector_usage.md> — the exact `--kv-transfer-config` JSON for `kv_producer` / `kv_consumer`, `VLLM_NIXL_SIDE_CHANNEL_PORT` (default 5600) and `_HOST` (default `localhost`), the `base_port + dp_rank` rule, `kv_lease_duration` (default 30 s) and `decoder_kv_blocks_ttl` (default 480 s), backend selection via `kv_connector_extra_config.backends`, and the explicit note that NCCL environment variables do not apply to NixlConnector.
3. **vLLM — `vllm/config/cache.py`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/vllm/config/cache.py> — the current defaults quoted in §5: `gpu_memory_utilization = 0.92` and `enable_prefix_caching = True`. **Correction:** older material (including this module's earlier drafts) quotes `gpu_memory_utilization = 0.90`.
4. **SGLang — `python/sglang/srt/mem_cache/radix_cache.py`** (v0.5.17) — <https://github.com/sgl-project/sglang/blob/main/python/sglang/srt/mem_cache/radix_cache.py> — the source for §4: `match_prefix` / `insert` / `evict`, the `lock_ref` refcount that pins in-flight nodes, and the eviction loop's min-heap over **evictable leaves** with a parent pushed back onto the heap once its last child is freed. Read as the authoritative answer to "which node gets evicted first."
5. **SGLang — `python/sglang/srt/server_args.py`** (v0.5.17) — <https://github.com/sgl-project/sglang/blob/main/python/sglang/srt/server_args.py> — `--disaggregation-mode {null,prefill,decode}`, `--disaggregation-transfer-backend` (default `mooncake`), `--disaggregation-bootstrap-port` (8998), `--disaggregation-ib-device`, and the LoRA block (`--max-loras-per-batch` default 8, `--lora-eviction-policy lru`) used in §3's table.
6. **TensorRT-LLM — `docs/source/features/quantization.md`** (1.3.0-rc) — <https://github.com/NVIDIA/TensorRT-LLM/blob/main/docs/source/features/quantization.md> — the recipe list and the per-architecture hardware support matrix reproduced in §6, including the Ampere row showing FP8 KV cache supported while FP8 weight/activation is not.
7. **TensorRT-LLM — `docs/source/features/disagg-serving.md`** (1.3.0-rc) — <https://github.com/NVIDIA/TensorRT-LLM/blob/main/docs/source/features/disagg-serving.md> — the aggregated-vs-disaggregated execution-timeline framing, the MPI/UCX/NIXL multi-backend KV exchange over RDMA/NVLink, `TRTLLM_NIXL_KVCACHE_BACKEND` (UCX default, LIBFABRIC from v0.16.0), and the statement that the advantage is largest for long inputs with moderate output lengths.
8. **Hugging Face — Text Generation Inference, README** — <https://github.com/huggingface/text-generation-inference> — the maintenance-mode `CAUTION` banner and the explicit recommendation of vLLM and SGLang, which is the primary source for §8. **Correction:** the repo is in maintenance mode and still releasing (v3.3.x), not archived.
9. **Zheng et al. — "SGLang: Efficient Execution of Structured Language Model Programs"** (NeurIPS 2024, arXiv 2312.07104) — <https://arxiv.org/abs/2312.07104> — the RadixAttention paper. *(arxiv.org is unreachable from this environment's egress proxy; the mechanism in §4 is verified against the SGLang source in entry 4 rather than the paper PDF, and the paper is cited for provenance of the idea.)*
10. **LMSYS Org — "Fast and Expressive LLM Inference with RadixAttention and SGLang"** — <https://www.lmsys.org/blog/2024-01-17-sglang/> — the original announcement, more accessible than the paper for a first read.
11. **NVIDIA Dynamo — disaggregated serving and KV-cache-aware routing docs** — <https://docs.nvidia.com/dynamo/user-guides/disaggregated-serving> · <https://docs.nvidia.com/dynamo/user-guides/kv-cache-aware-routing> — the reference architecture for separate prefill/decode pools and prefix-locality routing.
12. **KServe — Knative Serverless installation guide and control-plane architecture** — <https://kserve.github.io/website/docs/admin-guide/serverless> · <https://kserve.github.io/website/docs/concepts/architecture/control-plane> — the source for the Standard-vs-Serverless deployment distinction in §7.

**Real-world engineering**

13. **Baseten — "2x faster inference with KV cache-aware routing"** — <https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/> — the audited 48%/49% P95/P99 latency reductions and 61%/62% RPS and output-throughput gains cited in Real-world use cases and self-check (b).
14. **Qin et al. (Moonshot AI / Tsinghua) — "Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving"** (USENIX FAST '25, arXiv 2407.00079) — <https://arxiv.org/abs/2407.00079> · <https://www.usenix.org/system/files/fast25-qin.pdf> — 59–498% effective request capacity under SLO, the CPU/DRAM/SSD KV tier, and the hyperscale operating numbers.
15. **DeepSeek-AI — "DeepSeek-V3/R1 Inference System Overview"** (Open Source Week, 2025) — <https://github.com/deepseek-ai/open-infra-index/blob/main/202502OpenSourceWeek/day_6_one_more_thing_deepseekV3R1_inference_system_overview.md> — the disclosed 4-node/EP32 prefill and 18-node/EP144 decode units and per-node throughput.
16. **LMSYS Org — "Deploying DeepSeek with PD Disaggregation and Large-Scale Expert Parallelism on 96 H100 GPUs"** — <https://www.lmsys.org/blog/2025-05-05-large-scale-ep/> — the open-source reproduction: 3 prefill : 9 decode nodes, 52.3k input / 22.3k output tok/s per node, ~$0.20 per million output tokens self-hosted.
