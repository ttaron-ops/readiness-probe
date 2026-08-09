---
lesson: "07.6"
title: "Alternative servers and disaggregation"
module: "07"
concept: "Alternative servers and disaggregation"
status: not-started
est_time: "6h"
artifacts: []
---
# 07.6 · Alternative servers and disaggregation

> **Concept.** Pick an inference engine by workload shape, not by benchmark bragging rights; and understand disaggregated prefill/decode as the 2025–26 cost frontier.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

vLLM is the sane default (lessons 03–05), but "we run vLLM everywhere" is a
junior answer at a GPU-heavy shop. The staff-level skill is matching the engine
to the **workload shape** — because the wrong engine leaves 20–100% of your CPM
on the table for a specific traffic pattern, and CPM is the metric this module is
built around. A RAG platform with a 4k-token shared system prompt on every
request has completely different economics from a batch-summarisation job, and a
different engine wins each.

The interview signal is exactly this: given a workload, name the engine and the
reason in one sentence, and know when the frontier techniques (disaggregation,
KV-aware routing) are worth their operational cost. That is what separates a
platform engineer from someone who copied a Helm chart.

You already own the vocabulary from earlier lessons — prefill vs. decode,
compute- vs. memory-bound (03), TTFT/TPOT/queue-depth (05-obs), continuous
batching and the CPM curve (05). This lesson layers engine selection and the
disaggregation frontier on top; it does not re-teach the phases.

## What's new here

- **Prefix sharing as an economic lever.** RadixAttention (SGLang) caches KV by
  a radix tree keyed on token prefixes, so a shared system prompt or few-shot
  preamble is prefilled **once** and reused across every request that shares it.
  That converts repeated prefill compute into a cache hit — a direct TTFT and
  CPM win, but *only* when prefixes actually repeat.
- **Engine as workload-shaped choice, not a leaderboard.** Six runtimes, each
  with a shape it wins. You will build a decision matrix, not a ranking.
- **Disaggregated serving.** Splitting prefill and decode onto **separate GPU
  pools** so each scales independently and runs at its own optimal batch — the
  logical conclusion of the prefill/decode split you have seen since lesson 03.

## Core notes

### The engine survey — by workload shape

**vLLM — the default, general-purpose.** PagedAttention + continuous batching,
huge model coverage, first-class OpenAI-compatible server, the reference
`bench serve`. Pick it unless a workload trait below overrides. It also has
prefix caching (`--enable-prefix-caching`), so the SGLang gap has narrowed — but
SGLang's radix tree is still more aggressive on heavily-shared prefixes.

**SGLang — shared-prefix workloads (RAG, agents, multi-turn chat).**
RadixAttention makes prefix reuse the headline feature. When many requests share
a long system prompt, few-shot block, or conversation history, SGLang prefills
it once and every subsequent request skips that compute. It also ships a strong
structured-output/constrained-decoding path. Win condition: **high prefix
overlap**. Outside that, it converges toward vLLM.

**TensorRT-LLM — peak NVIDIA per-GPU performance, at a build cost.** Compiles
model-specific engines (fused kernels, FP8/FP4, in-flight batching). Best
tokens/sec/GPU on NVIDIA hardware, therefore potentially the lowest CPM at
saturation — *if* you can absorb the heavy build/compile pipeline, per-model
engine artifacts, and slower iteration. Pick it when a small set of stable,
high-volume models justifies squeezing the last 20–40% of per-GPU throughput.
Often deployed *behind* Triton or NVIDIA Dynamo rather than exposed directly.

**Triton Inference Server — multi-framework serving.** Not an LLM engine per se;
a serving runtime that hosts many backends (TensorRT-LLM, vLLM, ONNX, PyTorch,
Python) behind one endpoint with dynamic batching, model ensembles, and
concurrent model instances. Pick it when you serve a **mixed zoo** — LLMs
alongside embeddings, rerankers, classifiers, vision — and want one serving
layer over heterogeneous frameworks.

**KServe — the Kubernetes platform layer.** A CRD-based control plane
(`InferenceService`) on K8s: it does not replace the engine, it orchestrates one
(vLLM, Triton, etc.) with autoscaling (including scale-to-zero via Knative),
canary rollout, and a standard inference protocol. Pick it when the deliverable
is a **multi-tenant platform serving many models across many teams** and you need
GitOps-friendly declarative management — your CKA and 40-cluster context maps
directly onto this. It complements, not competes with, the engines above.

**TGI (Hugging Face Text Generation Inference) — don't pick this.** Historically
a common choice, now effectively **deprecated / archived** for new deployments;
HF's own serving momentum moved elsewhere and the ecosystem consolidated on vLLM
and SGLang. Recognise it in legacy stacks; do not start new work on it. Migrate
off it to vLLM.

### RadixAttention — where prefix sharing helps, and where it doesn't

**Helps (large win):** RAG with a fixed retrieval/system preamble; agent loops
that resend the same tool-definitions and scratchpad; multi-turn chat where the
growing history is a shared prefix across a session; few-shot prompts with a
common exemplar block. Anything where request N reuses request N-1's leading
tokens turns prefill into a cache hit — lower TTFT, higher effective throughput,
lower CPM.

**Does NOT help (no win, sometimes overhead):** workloads with **unique,
non-overlapping prompts** — batch document summarisation (every doc different),
translation of distinct inputs, classification of independent records, or any
high-entropy prompt stream. With no shared prefix there is nothing to cache; the
radix tree just adds bookkeeping. Here SGLang and vLLM land on essentially the
same CPM, and you pick on operational grounds. **This is self-check (a):** the
answer is the no-shared-prefix workload.

### Disaggregated prefill/decode serving (the frontier)

Recall the phase asymmetry: **prefill is compute-bound** (one big parallel pass,
sets TTFT) and **decode is memory-bandwidth-bound** (one token at a time, sets
TPOT). In a single vLLM instance they share the same GPUs and the same batch, so
they fight: a long prefill stalls decode for everyone in the batch
(head-of-line blocking), and the batch size that is optimal for decode is wrong
for prefill.

**Disaggregation** puts them on **separate GPU pools**. Prefill workers ingest
the prompt, build the KV cache, and hand it off; decode workers stream tokens
from that KV. What this buys you:

- **Independent scaling.** Size the prefill pool to prompt volume/length and the
  decode pool to concurrent generation load — two different knees, two different
  autoscalers. A prompt-heavy RAG fleet and a generation-heavy chat fleet no
  longer force one compromise batch. **This is self-check (b).**
- **Phase-appropriate hardware and batch.** Each pool runs at its own optimal
  operating point instead of the blended one, and you can even use different GPU
  SKUs per phase.
- **No cross-phase interference.** Long prefills stop injecting TTFT jitter into
  decode; TPOT stabilises.

The cost: you must **move the KV cache** between pools, which is a lot of bytes
over the network. This only pays off with a fast interconnect — **RDMA / NVLink /
InfiniBand**, transported via NVIDIA's **NIXL** (NVIDIA Inference Xfer Library) —
otherwise the transfer tax eats the gains. And it only pays off at **scale**;
below a threshold, one fused instance is simpler and cheaper.

**NVIDIA Dynamo** is the reference implementation: a datacenter-scale serving
framework with disaggregated prefill/decode, a smart router, and NIXL-based KV
transfer, typically driving vLLM / TensorRT-LLM / SGLang workers underneath.

### KV-cache-aware routing

The second Dynamo idea. A naive load balancer round-robins requests. A
**KV-aware router** instead sends a request to the worker that **already holds the
matching KV blocks** for its prefix — turning what would be a fresh prefill into
a cache hit at the routing layer. It is RadixAttention's insight lifted from
inside one engine up to the fleet: route by prefix locality, not just by load.
Combined with disaggregation, it is why prefix-heavy workloads see large gains.

**Baseten** reported roughly **2× faster inference** using NVIDIA Dynamo on a
long-context workload by combining disaggregated serving with KV-aware routing —
a concrete, citable data point that disaggregation is production-real in 2025–26,
not a paper idea.

## Worked example

A team runs a customer-support RAG assistant: every request carries a **3,500-token
shared system prompt + retrieved policy docs**, then a short user turn and a
~200-token answer. Traffic is 300 concurrent sessions.

Reasoning through the choices:

- **Prefix overlap is huge** (the 3,500-token preamble repeats on nearly every
  request) → **SGLang / RadixAttention** or vLLM with prefix caching is the
  engine layer; the shared prefix is prefilled once, so per-request TTFT and CPM
  drop sharply versus re-prefilling 3,500 tokens each time.
- **Prefill-heavy, decode-light** (3,500 in / 200 out) → strong disaggregation
  candidate: a small decode pool, a larger prefill pool, plus **KV-aware routing**
  so repeat prefixes hit a warm worker. This is the Baseten-style ~2× shape.
- **Platform delivery** across the fleet → wrap it in **KServe** for
  autoscaling/canary/scale-to-zero, with SGLang as the runtime inside the
  `InferenceService`.

Contrast: the same team's **nightly ticket-summarisation batch** — every ticket
unique, no shared prefix. Here RadixAttention buys nothing; **vLLM** at a large
batch optimised purely for throughput (lesson 05's cost curve) wins, and
disaggregation is unjustified overhead. Same team, two workloads, two answers.

## Practice (light)

Reuse the lesson-05 harness and rented GPU. Goal: prove the prefix-sharing effect
in CPM terms and produce the engine decision matrix for the deliverable.

**1. Build a shared-prefix dataset.** Construct ~500 prompts that all begin with
the *same* ~2,000-token preamble followed by a short unique tail. Keep
output length fixed (e.g. 128).

**2. Baseline on vLLM (prefix caching off, then on):**
```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct --port 8000 --max-num-seqs 128
# repeat with --enable-prefix-caching to see vLLM's own prefix reuse
vllm bench serve --backend vllm --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 \
  --dataset-name sharegpt --dataset-path ./shared_prefix.json \
  --num-prompts 500 --max-concurrency 64 \
  --percentile-metrics ttft,tpot --metric-percentiles 50,99 \
  --save-result --result-filename vllm_prefix.json
```

**3. Same dataset on SGLang:**
```bash
python -m sglang.launch_server \
  --model-path meta-llama/Llama-3.1-8B-Instruct --port 30000
# SGLang ships bench_serving; or point vllm bench serve at the OpenAI-compatible
# SGLang endpoint with --backend sglang / --base-url http://localhost:30000
python -m sglang.bench_serving --backend sglang \
  --dataset-path ./shared_prefix.json --num-prompts 500 --max-concurrency 64
```

**4. Compute CPM** for each run with the lesson-05 equation and your GPU's
`hourly_rate`. Then swap in a **no-shared-prefix** dataset (all-unique prompts)
and rerun both — you should see the SGLang advantage collapse.

**Acceptance.**
- A **vLLM-vs-SGLang CPM comparison** table/chart on the shared-prefix workload
  (and the control run showing the gap vanishing with unique prompts), added to
  the cost-per-token deliverable.
- A **one-page engine decision matrix**: rows = {vLLM, SGLang, TensorRT-LLM,
  Triton, KServe, TGI}, columns = {best workload shape, key mechanism, when to
  pick, when NOT to}, with TGI explicitly marked "deprecated — don't pick".

## Self-check

**(a) When does RadixAttention / prefix-sharing NOT help — what workload?**

**Answer:** Workloads with **unique, non-overlapping prompts** and no repeated
prefix: batch summarisation of distinct documents, translation of independent
inputs, classification of unrelated records — any high-entropy prompt stream.
With nothing shared across requests there is nothing to cache, so the radix tree
adds bookkeeping without a hit-rate payoff, and SGLang lands at essentially the
same CPM as vLLM. Prefix sharing only pays when requests actually reuse leading
tokens (shared system prompt, few-shot block, conversation history).

**(b) Why disaggregate prefill from decode into separate pools — what does it let
you scale independently?**

**Answer:** Prefill is compute-bound (sets TTFT) and decode is memory-bandwidth-
bound (sets TPOT); fused in one instance they share a batch and interfere (long
prefills stall decode, and one batch size can't be optimal for both). Separate
pools let you **scale prefill capacity to prompt volume/length and decode
capacity to concurrent generation load independently** — two autoscalers, two
operating points, each phase at its own optimal batch and even its own GPU SKU.
The cost is moving the KV cache between pools, which only pays off over a fast
interconnect (RDMA/NVLink via NIXL) and at scale — Baseten reported ~2× with
NVIDIA Dynamo doing exactly this plus KV-aware routing.

**(c) Which engine for a K8s inference PLATFORM serving many models, and why?**

**Answer:** **KServe.** It is the Kubernetes control-plane/CRD layer
(`InferenceService`), not an engine — it orchestrates a runtime (vLLM, Triton,
etc.) with declarative GitOps management, per-model autoscaling including
scale-to-zero (Knative), canary rollouts, and a standard inference protocol
across many models and tenants. An engine alone (vLLM/SGLang/TRT-LLM) serves one
model well but gives you no multi-model platform layer; KServe supplies that and
runs those engines underneath. For a mixed-framework zoo behind one endpoint,
**Triton** is the runtime you'd wrap inside KServe.

## Resources

1. **MarkTechPost — "Comparing the Top 6 Inference Runtimes for LLM Serving in
   2025"** — the survey framing for the decision matrix (vLLM, SGLang,
   TensorRT-LLM, Triton, and the trade-offs).
   <https://www.marktechpost.com/2025/11/07/comparing-the-top-6-inference-runtimes-for-llm-serving-in-2025/>
2. **NVIDIA Dynamo — disaggregated serving** and **KV-cache-aware routing** docs
   — the reference architecture for separate prefill/decode pools and prefix-
   locality routing.
   <https://docs.nvidia.com/dynamo/user-guides/disaggregated-serving> ·
   <https://docs.nvidia.com/dynamo/user-guides/kv-cache-aware-routing>
3. **Baseten — "How Baseten achieved 2× faster inference with NVIDIA Dynamo"** —
   the production data point for disaggregation + KV-aware routing.
   <https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/>
