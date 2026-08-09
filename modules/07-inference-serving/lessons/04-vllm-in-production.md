---
lesson: "07.4"
title: "vLLM in production"
module: "07"
concept: "vLLM in production"
status: not-started
est_time: "6h"
artifacts: []
---
# 07.4 · vLLM in production

> **Concept.** The production knobs — `gpu-memory-utilization`, `max-num-seqs`, `max-model-len`, `tensor-parallel-size`/`pipeline-parallel-size` — are not independent dials; they trade against a single fixed HBM budget, and when you oversubscribe it the scheduler *preempts* (recompute or swap). Tuning is finding the edge of that budget without falling off it.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

---

## Why this matters

07.3 explained *why* vLLM can pack many sequences onto a GPU. This lesson is *your job*:
picking the flags that decide how many, how long, and across how many GPUs — and knowing
what happens when demand exceeds what you provisioned.

Every one of these knobs moves cost-per-million-tokens. Set `gpu-memory-utilization` too
low and you're paying for HBM you never use (KV cache too small → low `max-num-seqs` → low
`tokens/hr` → high $/1M). Set it too high and the server OOM-crashes on a traffic spike,
which is a worse outage than a slow one. Add a second GPU with tensor parallelism when
replicas would have been cheaper and you've doubled spend for a latency win you didn't
need. This is the FinOps core of the whole module: the same H100-hour costs the same
whether it serves 2,000 or 20,000 tokens/sec — configuration is the multiplier.

The failure modes here are the ones that page you: OOM at startup, OOM under load,
latency cliffs when the scheduler starts preempting. You need to recognize each from its
signature and know which knob to turn.

---

## What's new here vs 03 / 05

- **Module 03** told you batch size is the throughput lever and KV is the constraint. Here
  you *set* the batch (`max-num-seqs`) and *size* the KV budget (`gpu-memory-utilization`,
  `max-model-len`) explicitly, and split across GPUs.
- **Module 05** gave you the SLIs (TTFT/TPOT) and `/metrics`. Here those metrics become
  the **preemption alarm**: `vllm:num_preemptions_total` climbing is the signal that you've
  oversubscribed KV, and TPOT spiking is its symptom.
- **07.3** gave you the mechanism (blocks, continuous batching). New: the operational
  envelope around it — the zero-sum HBM math and the two recovery modes when it's exceeded.

---

## Core notes

### 1. The fixed HBM budget (why the knobs are zero-sum)

On a single GPU, HBM is partitioned at startup roughly as:

```
HBM_total × gpu_memory_utilization
   = model_weights            (fixed by model + dtype/quant)
   + non-KV working memory    (activations, CUDA graphs, temp buffers)
   + KV cache pool            ← everything left over goes here
```

vLLM measures weights and working memory, then hands the **remainder** to the KV block
pool. The startup log prints it: `GPU KV cache size: N tokens` and `Maximum concurrency for
<max-model-len> tokens per request: Kx`. That KV pool is the shared resource all your
sequences draw blocks from. Three knobs pull on it:

- **`gpu-memory-utilization`** (default **0.90**; production **0.90–0.95**) — the fraction
  of HBM vLLM may use. Higher → bigger KV pool → more concurrent sequences. But the
  headroom above it must absorb allocation spikes (CUDA graph capture, a burst of long
  prefills); too little headroom and you OOM under load.
- **`max-model-len`** — max tokens (prompt + output) per sequence. This sets the *per-
  sequence* KV ceiling. Raising it doesn't allocate more per request (PagedAttention grows
  on demand) but it lowers the printed "maximum concurrency" and, if you also raise
  `max-num-batched-tokens` to match, eats working memory.
- **`max-num-seqs`** — hard cap on concurrent running sequences (V1 default **1024**, but
  the KV pool usually binds first). Together with `max-num-batched-tokens` (the per-step
  token budget) this bounds how much of the pool is live at once.

**The zero-sum interaction:** these draw on one budget. Raise `max-model-len` from 8K to
32K and you've quadrupled the worst-case KV per sequence — the pool now holds ~¼ the
concurrency, and if long requests actually arrive you'll preempt or OOM — so you must give
something back: lower `max-num-seqs`, or accept fewer concurrent long-context requests.
(Raising `gpu-memory-utilization` to compensate is the wrong move — it steals the headroom
that absorbs allocation spikes and trades a throughput problem for a crash.) You cannot have max
context *and* max concurrency *and* max utilization on fixed HBM; pick two and let the
third float. Size `max-model-len` to your **actual** P99 request length, not the model's
architectural max — over-sizing it silently throttles concurrency.

### 2. Symptoms: gpu-memory-utilization too high / too low

- **Too high** (e.g. 0.97–0.99): no headroom for transient allocations. Signature is
  `torch.OutOfMemoryError: CUDA out of memory` — either **at startup** during CUDA graph
  capture, or **under load** when a burst of concurrent prefills spikes working memory past
  the sliver you left. It manifests as crashes/restarts, not slowness. Also: if another
  process shares the card, vLLM's absolute reservation collides with it.
- **Too low** (e.g. 0.60): the KV pool is tiny, "maximum concurrency" is small,
  `max-num-seqs` never fills, and you preempt or queue at modest load. Signature is a GPU
  that shows low utilization *and* low throughput while requests wait — you're paying for
  HBM you fenced off. This is the quiet, expensive failure; nothing crashes, your $/1M is
  just 2× what it should be.

Tune it upward until you find the OOM edge under realistic load, then back off one step for
headroom (0.90 is the safe default; 0.92–0.95 for a dedicated card with well-characterized
traffic).

### 3. Preemption: what happens when KV runs out

Even correctly tuned, load can transiently demand more KV blocks than the pool holds —
many resident sequences each grow past their prompt at once. The scheduler cannot fail the
in-flight requests, so it **preempts**: it evicts one or more running sequences to free
their blocks for the rest, then resumes them later. Two modes:

- **Recompute** (V1 default) — discard the preempted sequence's KV blocks entirely; when
  resumed, **re-run prefill** over its tokens to rebuild the KV. Cost = wasted prefill
  compute (proportional to context length), but no data movement and it frees blocks
  instantly. V1 defaults to this because prefill is cheap relative to the alternative and
  V1's KV manager makes recompute low-overhead.
- **Swap** — copy the preempted sequence's KV blocks out to **CPU RAM** (`swap-space`,
  default **4 GiB/GPU**) and copy them back on resume. Cost = KV bytes over the PCIe bus,
  *twice* (out and in). No recompute, but the KV cache is large and PCIe is slow, so for
  long contexts swap can move gigabytes and stall the pipeline harder than just
  recomputing would.

**Trigger:** KV-cache exhaustion under concurrency — too many resident sequences growing
simultaneously. **Signature in your module-05 metrics:** `vllm:num_preemptions_total`
climbing, running-vs-waiting queue oscillating, and **TPOT spiking** for the affected
requests (they stall while preempted). A steady stream of preemptions means you're
oversubscribed: lower `max-num-seqs`, lower `max-model-len`, or add capacity. Occasional
preemption under bursts is normal and fine — it's the graceful-degradation valve, not a
bug.

### 4. Multi-GPU: tensor vs pipeline parallel, vs replicas

When a model + its KV won't fit (or won't hit latency targets) on one GPU:

- **`tensor-parallel-size` (TP)** — shard *every layer's* weights and attention across N
  GPUs; they run each token **collectively**, exchanging activations via all-reduce over
  **NVLink** each layer. Cuts weight and KV memory ~N× per GPU and cuts latency, but only
  pays off with fast interconnect — over PCIe the all-reduces dominate. Use TP when a model
  doesn't fit on one GPU (e.g. a 70B in FP16 needs ~140 GB → TP=2 on H100 80GB, or TP=4/8)
  or to lower TTFT/TPOT for a model that *does* fit. Set TP ≤ GPUs per NVLink node.
- **`pipeline-parallel-size` (PP)** — split the model by *layer ranges* across GPUs (or
  nodes); micro-batches flow through the stages. Communicates far less than TP (point-to-
  point, not all-reduce), so it **crosses nodes / slow links** where TP can't, but adds
  pipeline-bubble latency. Use PP to span nodes; combine as TP-within-node ×
  PP-across-nodes for very large models.
- **Separate replicas** — N independent single-GPU (or single-TP-group) vLLM instances
  behind a load balancer. No cross-GPU communication tax, linear throughput scaling, and
  independent failure domains.

**Crossover:** if the model *fits* on one GPU and you're **throughput-bound**, replicas win
— they avoid the all-reduce tax and scale throughput linearly (N replicas ≈ N× tokens/sec,
cheaper per token than N-way TP whose comms overhead makes it sub-linear). Reach for TP
when the model **doesn't fit** on one GPU, or when you're **latency-bound** on a single
request (TP lowers TTFT/TPOT; replicas don't help one request's latency at all). Reach for
PP only to cross an interconnect boundary TP can't. In practice: smallest parallelism that
makes the model fit + hits latency SLO, then scale *out* with replicas for throughput.

### 5. Version pin

This is **vLLM 0.11.x, V1 engine** (V0 removed). Recompute is the default preemption mode;
chunked prefill and prefix caching are on by default. Older guides showing `swap` as
default or a separate `--enable-chunked-prefill` toggle predate V1 — flag and ignore.

---

## Worked example

70B-class model, TP=2, deliberately oversubscribed to force preemption. (Budget-tight?
Substitute `meta-llama/Llama-3.1-8B-Instruct` on TP=1 and drop `max-model-len` low, e.g.
2048, to make KV exhaustion easy to hit.)

```bash
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 2 \
  --gpu-memory-utilization 0.92 \
  --max-model-len 8192 \
  --max-num-seqs 256 \
  --port 8000
# startup log:
#   GPU KV cache size: 45,120 tokens
#   Maximum concurrency for 8192 tokens per request: 5.5x   ← only ~5 full-context seqs fit
```

That 5.5x is the warning: ask for more than ~5 concurrent full-length requests and the
scheduler must preempt. Force it — many concurrent long-context requests:

```bash
# 40 concurrent requests, each with a long prompt + long output → KV blows past the pool
seq 40 | xargs -P40 -I{} curl -s localhost:8000/v1/completions \
  -H 'content-type: application/json' \
  -d '{"model":"meta-llama/Llama-3.3-70B-Instruct",
       "prompt":"'"$(python -c "print('Summarize this incident. '*300)")"'",
       "max_tokens":1024}' >/dev/null &
```

Watch preemption in logs and metrics:

```bash
# logs — V1 emits a warning when it preempts:
#   WARNING ... Sequence group ... is preempted by PreemptionMode.RECOMPUTE ...
#   because there is not enough KV cache space.

curl -s localhost:8000/metrics | grep -E 'num_preemptions_total|num_requests_(running|waiting)'
# vllm:num_preemptions_total{...}     37      ← climbing = oversubscribed
# vllm:num_requests_running{...}       5
# vllm:num_requests_waiting{...}      35      ← queue backing up
```

Cross-check module-05 TPOT (`vllm:time_per_output_token_seconds`): it spikes for preempted
requests. **Tuned config** = lower ambition until preemptions stop under your real P99 load
— e.g. drop `max-num-seqs` to ~16, or `max-model-len` to your actual P99 (say 4096), which
raises "maximum concurrency" and empties the waiting queue.

---

## Practice — feeds the deliverable

On **rented GPUs** (2× A100/H100 for the 70B TP=2 path; or 1× L4/A10 with the 8B
substitute):

1. Deploy with `tensor-parallel-size=2` (or TP=1 for the 8B). Record the startup KV-cache
   size and "maximum concurrency" line.
2. **Oversubscribe**: fire many concurrent long-context requests (the `xargs -P` loop) so
   aggregate KV demand exceeds the pool.
3. Capture a **preemption event**: the `PreemptionMode.RECOMPUTE` log line **and** the
   `vllm:num_preemptions_total` delta, plus the TPOT spike from module-05 metrics.
4. **Tune**: adjust `max-num-seqs` / `max-model-len` (and, if you saw startup or under-load
   OOM, `gpu-memory-utilization`) until preemptions stop under your target load. Record
   before/after throughput (tokens/sec) — this is a direct $/1M input.

**Acceptance (deliverable):** a **documented preemption event** (log line + counter delta +
TPOT impact) and a **tuned config** (the flag set that eliminates steady-state preemption
at your target concurrency) written into the cost-per-token workbook, with the tuned
throughput feeding the $/1M-tokens figure and a one-line note on the TP-vs-replica choice
you'd make for scale-out.

---

## Self-check

**(a) What's the symptom of setting `gpu-memory-utilization` too high, and too low?**

**Answer:** Too high (≥~0.97) leaves no headroom for transient allocations, so you get
`CUDA out of memory` **crashes** — at startup during CUDA graph capture, or under load when
concurrent prefills spike working memory past the sliver left free. Too low (e.g. 0.6)
shrinks the KV pool, so "maximum concurrency" is small, `max-num-seqs` never fills, and you
queue/preempt at modest load with the GPU showing low utilization *and* low throughput —
you're paying for fenced-off HBM. High fails loudly (outage); low fails quietly (2× your
$/1M). Tune up to the OOM edge under realistic load, back off one step; 0.90 default,
0.92–0.95 on a dedicated, well-characterized card.

**(b) When is swap preemption worse than recompute?**

**Answer:** Swap copies the preempted sequence's KV blocks out to CPU RAM and back over
PCIe — moving the *entire* KV twice. For **long contexts** that KV is gigabytes, and PCIe
is slow, so the transfer can stall the pipeline harder and longer than simply discarding
the blocks and re-running prefill (recompute) would — prefill is compute the GPU is good
at, and it frees blocks instantly with no bus traffic. So swap is worse precisely when KV
is large relative to prompt-recompute cost and interconnect bandwidth is the bottleneck —
which is why V1 defaults to **recompute**. Swap only wins when prompts are very long
(recompute expensive) *and* CPU RAM bandwidth to spare — an increasingly rare case.

**(c) Tensor parallelism vs running separate replicas — when each, and where's the
crossover?**

**Answer:** Replicas are independent full-model instances behind a load balancer: no cross-
GPU comms, linear throughput scaling, cheapest per token — use them when the model **fits
on one GPU** and you're **throughput-bound**. Tensor parallelism shards every layer across
N GPUs communicating via all-reduce over NVLink each layer: it cuts per-GPU weight+KV
memory and lowers single-request latency, but the comms tax makes throughput scaling sub-
linear. Use TP when the model **doesn't fit** on one GPU (e.g. 70B FP16 → TP=2+) or when
you're **latency-bound** on individual requests (TP lowers TTFT/TPOT; replicas can't help
one request). Crossover: fits + throughput-bound → replicas; doesn't fit or latency-bound →
TP (smallest TP that fits/meets SLO), then scale out with replicas.

---

## Resources

1. **vLLM Optimization and Tuning** — https://docs.vllm.ai/en/stable/configuration/optimization/
   — the canonical current reference for `max-num-seqs`, `max-num-batched-tokens`, chunked
   prefill, and preemption behavior. Primary source.
2. **Conserving Memory** — https://docs.vllm.ai/en/latest/configuration/conserving_memory/
   — `gpu-memory-utilization`, `max-model-len`, quantization, and TP/PP as memory levers;
   the OOM-avoidance playbook.
3. **"Inside vLLM: Anatomy of a High-Throughput Inference System"** —
   https://vllm.ai/blog/2025-09-05-anatomy-of-vllm — current (V1) production-deployment
   walkthrough tying the scheduler, preemption, and parallelism together.
