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
sources: 10
---
# 07.6 · Alternative servers and disaggregation

> **Concept.** Pick an inference engine by workload shape, not by benchmark bragging rights; and understand disaggregated prefill/decode as the 2025–26 cost frontier.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

Lesson 05 gave you the CPM-vs-batch curve and the SLO knee for **one engine on one fused GPU pool** — vLLM, prefill and decode sharing the same batch. That curve is real, but it silently assumes the engine and the topology are fixed and only the batch size moves. This lesson removes both assumptions: it asks which engine you should even be running the sweep against, and whether prefill and decode belong on the same GPUs at all. The gap it fills is the two decisions that sit *above* the batching sweep — pick the runtime, pick the topology — before you even get to pick the batch size.

## Why this matters

vLLM is the sane default (lessons 03–05), but "we run vLLM everywhere" is a junior answer at a GPU-heavy shop. The staff-level skill is matching the engine to the **workload shape** — because the wrong engine leaves 20–100% of your CPM on the table for a specific traffic pattern, and CPM is the metric this module is built around. A RAG platform with a 4k-token shared system prompt on every request has completely different economics from a batch-summarisation job, and a different engine wins each.

The interview signal is exactly this: given a workload, name the engine and the reason in one sentence, and know when the frontier techniques (disaggregation, KV-aware routing) are worth their operational cost. That is what separates a platform engineer from someone who copied a Helm chart. The stakes are not abstract: teams operating at real scale — Moonshot AI's Kimi, DeepSeek's own production API — have published disaggregation numbers in the range of 1.6–6× effective capacity versus a fused baseline. That is not a marginal optimization; at fleet scale it is the difference between one GPU-quarter of budget and six.

You already own the vocabulary from earlier lessons — prefill vs. decode, compute- vs. memory-bound (module 03), TTFT/TPOT/queue-depth (module 05), continuous batching and the CPM curve (lesson 05). This lesson layers engine selection and the disaggregation frontier on top; it does not re-teach the phases.

## What's new here (calibration)

You already have the phase split, the SLIs, and the batching-CPM relationship. Three things are genuinely new:

- **Prefix sharing as an economic lever.** RadixAttention (SGLang) caches KV by a radix tree keyed on token prefixes, so a shared system prompt or few-shot preamble is prefilled **once** and reused across every request that shares it. That converts repeated prefill compute into a cache hit — a direct TTFT and CPM win, but *only* when prefixes actually repeat.
- **Engine as workload-shaped choice, not a leaderboard.** Six runtimes, each with a shape it wins. You will build a decision matrix, not a ranking.
- **Disaggregated serving.** Splitting prefill and decode onto **separate GPU pools** so each scales independently and runs at its own optimal batch — the logical conclusion of the prefill/decode split you have seen since module 03, and of the single-batch compromise lesson 05's knee forced you to accept.

## Core concepts

### The engine survey — by workload shape

**vLLM — the default, general-purpose.** PagedAttention + continuous batching, huge model coverage, first-class OpenAI-compatible server, the reference `bench serve`. Pick it unless a workload trait below overrides. It also has prefix caching (`--enable-prefix-caching`, on by default in V1 — see lesson 03's Automatic Prefix Caching section), so the SGLang gap has narrowed — but SGLang's radix tree is still more aggressive on heavily-shared prefixes.

**SGLang — shared-prefix workloads (RAG, agents, multi-turn chat).** RadixAttention makes prefix reuse the headline feature: the runtime and frontend language were co-designed around it from the paper onward (SGLang: Efficient Execution of Structured Language Model Programs, NeurIPS 2024 — up to 6.4× higher throughput than then-state-of-the-art on prefix-heavy workloads in the paper's own benchmarks). When many requests share a long system prompt, few-shot block, or conversation history, SGLang prefills it once and every subsequent request skips that compute. It also ships a strong structured-output/constrained-decoding path. Win condition: **high prefix overlap**. Outside that, it converges toward vLLM.

**TensorRT-LLM — peak NVIDIA per-GPU performance, at a build cost.** Compiles model-specific engines (fused kernels, FP8/FP4, in-flight batching). Best tokens/sec/GPU on NVIDIA hardware, therefore potentially the lowest CPM at saturation — *if* you can absorb the heavy build/compile pipeline, per-model engine artifacts, and slower iteration. Pick it when a small set of stable, high-volume models justifies squeezing the last 20–40% of per-GPU throughput. Often deployed *behind* Triton or NVIDIA Dynamo rather than exposed directly; NVIDIA's own docs describe it as integrating with the LLM API for single- to multi-node deployments with the parallelism strategies you already know from lesson 04 (TP/PP).

**Triton Inference Server — multi-framework serving.** Not an LLM engine per se; a serving runtime that hosts many backends (TensorRT-LLM, vLLM, ONNX, PyTorch, Python) behind one endpoint with dynamic batching, model ensembles, and concurrent model instances. Pick it when you serve a **mixed zoo** — LLMs alongside embeddings, rerankers, classifiers, vision — and want one serving layer over heterogeneous frameworks.

**KServe — the Kubernetes platform layer.** A CRD-based control plane (`InferenceService`) on K8s: it does not replace the engine, it orchestrates one (vLLM, Triton, etc.) with autoscaling (including scale-to-zero via Knative's serverless mode), canary rollout, and a standard inference protocol. KServe's own docs are explicit that the Knative/serverless path suits predictive workloads with fast scale-to-zero, while GPU-heavy generative workloads more often run the Standard Deployment mode — a distinction worth knowing before you promise scale-to-zero for a 70B model. Pick KServe when the deliverable is a **multi-tenant platform serving many models across many teams** and you need GitOps-friendly declarative management — your CKA and 40-cluster context maps directly onto this. It complements, not competes with, the engines above.

**TGI (Hugging Face Text Generation Inference) — don't pick this for new work.** Historically a common choice; HF's own serving momentum and the ecosystem's default have moved to vLLM and SGLang, and TGI sees materially less new production adoption. You will see comparison pieces online that still list TGI v3 favorably on specific long-prompt/prefix-caching benchmarks — treat those claims skeptically and re-benchmark before trusting them; they do not change this module's guidance. Recognise TGI in legacy stacks; do not start new work on it. Migrate off it to vLLM.

### RadixAttention — where prefix sharing helps, and where it doesn't

**Helps (large win):** RAG with a fixed retrieval/system preamble; agent loops that resend the same tool-definitions and scratchpad; multi-turn chat where the growing history is a shared prefix across a session; few-shot prompts with a common exemplar block. Anything where request N reuses request N−1's leading tokens turns prefill into a cache hit — lower TTFT, higher effective throughput, lower CPM.

**Does NOT help (no win, sometimes overhead):** workloads with **unique, non-overlapping prompts** — batch document summarisation (every doc different), translation of distinct inputs, classification of independent records, or any high-entropy prompt stream. With no shared prefix there is nothing to cache; the radix tree just adds bookkeeping. Here SGLang and vLLM land on essentially the same CPM, and you pick on operational grounds. **This is self-check (a):** the answer is the no-shared-prefix workload.

**The gap has narrowed, not closed.** vLLM's own Automatic Prefix Caching (lesson 03 §4) covers most of the same ground — it costs under 1% throughput at a 0% hit rate, so there's no real downside to leaving it on. SGLang's radix tree is still the more aggressive implementation on heavily-shared, tree-structured prefixes (many branches off one shared root, as in agent fan-out), but for a single linear shared system prompt the two engines are close enough that the decision often comes down to your team's other requirements (structured-output support, ecosystem fit), not raw prefix-cache throughput.

### Disaggregated prefill/decode serving (the frontier)

Recall the phase asymmetry: **prefill is compute-bound** (one big parallel pass, sets TTFT) and **decode is memory-bandwidth-bound** (one token at a time, sets TPOT). In a single vLLM instance they share the same GPUs and the same batch, so they fight: a long prefill stalls decode for everyone in the batch (head-of-line blocking), and the batch size that is optimal for decode is wrong for prefill — the exact tension lesson 05's SLO knee was forcing you to compromise on.

**Disaggregation** puts them on **separate GPU pools**. Prefill workers ingest the prompt, build the KV cache, and hand it off; decode workers stream tokens from that KV. What this buys you:

- **Independent scaling.** Size the prefill pool to prompt volume/length and the decode pool to concurrent generation load — two different knees, two different autoscalers. A prompt-heavy RAG fleet and a generation-heavy chat fleet no longer force one compromise batch. **This is self-check (b).**
- **Phase-appropriate hardware and batch.** Each pool runs at its own optimal operating point instead of the blended one, and you can even use different GPU SKUs per phase.
- **No cross-phase interference.** Long prefills stop injecting TTFT jitter into decode; TPOT stabilises.

The cost: you must **move the KV cache** between pools, which is a lot of bytes over the network. This only pays off with a fast interconnect — **RDMA / NVLink / InfiniBand**, transported via NVIDIA's **NIXL** (NVIDIA Inference Xfer Library) — otherwise the transfer tax eats the gains. And it only pays off at **scale**; below a threshold, one fused instance is simpler and cheaper. Concretely: the production numbers below (Mooncake, DeepSeek) come from fleets of dozens to hundreds of GPUs, not single-digit deployments. Proposing disaggregation for a 4-GPU service is a common junior mistake — the coordination overhead (a KV-transfer fabric, two autoscalers, two failure domains) is real fixed cost that a small fleet cannot amortise.

**NVIDIA Dynamo** is the reference implementation: a datacenter-scale serving framework with disaggregated prefill/decode, a smart router, and NIXL-based KV transfer, typically driving vLLM / TensorRT-LLM / SGLang workers underneath.

### KV-cache-aware routing

The second Dynamo idea. A naive load balancer round-robins requests. A **KV-aware router** instead sends a request to the worker that **already holds the matching KV blocks** for its prefix — turning what would be a fresh prefill into a cache hit at the routing layer. It is RadixAttention's insight lifted from inside one engine up to the fleet: route by prefix locality, not just by load. Combined with disaggregation, it is why prefix-heavy workloads see large gains — this is the exact mechanism behind the Baseten numbers in Real-world use cases below.

### A two-year research lineage, one continuous idea

It is worth naming the throughline explicitly, because it is a strong interview answer in its own right: PagedAttention (lesson 03, 2023) established that the KV cache is a first-class resource to be managed like memory, not an incidental buffer. RadixAttention (SGLang, early 2024) took the next step — treat KV as *shareable* across requests within one engine, indexed by prefix. Disaggregation (DistServe-era research through 2024, Mooncake and DeepSeek's production systems in 2024–25) took the step after that — treat KV as a resource to be *transported* between specialized pools, not just cached in place. Three papers, one continuously maturing idea, about two years apart, each now running in production at real scale.

## Perspectives

**The frontier-lab-scale view.** DeepSeek's own disclosed production configuration runs prefill on 4-node/EP32 deployment units and decode on 18-node/EP144 units — hundreds of GPUs, not a handful — and Moonshot AI's Mooncake (the system behind Kimi) reports 59–498% effective-capacity gains over non-disaggregated baselines while still meeting SLOs, running across thousands of nodes processing over 100 billion tokens a day. At this scale, disaggregation is not an optimization to consider — it is close to a structural necessity, because a fused topology simply cannot hit the throughput-per-dollar these fleets require. What counts as "scale enough to disaggregate" is itself calibrated by these numbers: hundreds of GPUs serving one model family, not a single-digit deployment.

**The platform-vendor view.** Baseten is a mid-size inference platform, not a frontier lab, and its published Dynamo results are the more realistic story for what a Staff Platform Engineer at a normal-sized shop actually ships: concrete, auditable P95/P99/RPS deltas (see Real-world use cases) from adopting disaggregation plus KV-aware routing on an existing fleet, not a from-scratch hyperscale build. This is the version of the decision you are more likely to actually face — "should we turn this on for our existing vLLM fleet" rather than "how do we architect for 10,000 GPUs."

**The research-lineage view.** As laid out in Core concepts, RadixAttention and disaggregation are not two unrelated frontier techniques — they are two points on the same two-year arc of treating the KV cache as a routable, shareable, transportable resource rather than a private per-request buffer. Tying PagedAttention (lesson 03) → RadixAttention (this lesson) → disaggregation + KV-aware routing (this lesson) into one sentence is a strong way to demonstrate you understand *why* the field moved this direction, not just *what* the current tools do.

**The vendor-neutral engine-selection view.** Comparison articles are useful precisely because they are not written by the vendor whose engine they favor — but that neutrality is not automatic, and it decays fast in a space this competitive. A comparison piece that still lists TGI favorably on a specific benchmark, for instance, is not necessarily wrong on that benchmark, but it should not override this module's operational guidance (don't start new work on TGI) without you re-running the comparison yourself on your own workload. Treat published engine leaderboards as a starting hypothesis, not a final answer — the same discipline lesson 05 asked of a CPM curve applies here.

## Real-world use cases

- **Baseten — "2x faster inference with KV cache-aware routing"** (2025), built on NVIDIA Dynamo. The audited numbers: **48% decrease in P95 latency, 49% decrease in P99 latency, 61% more requests/sec, and 62% more output tokens/sec** on a long-context workload, from combining disaggregated serving with KV-aware routing. Baseten also reports serving up to 5× as many requests on the same GPU count for some workloads. What it shows: this is not a paper result — it is a named platform vendor's own production numbers, with the specific latency percentiles a Staff engineer would actually be asked to defend. <https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/>

- **Moonshot AI — Mooncake, the serving system behind Kimi** (USENIX FAST '25 / arXiv 2407.00079). A KVCache-centric disaggregated architecture that separates prefill and decode clusters and also pools underused CPU/DRAM/SSD/NIC capacity as a KV cache tier. Reports **59–498% effective request-capacity increase** versus non-disaggregated baselines on real production traffic traces, while continuing to meet SLOs; runs across thousands of nodes serving over 100 billion tokens/day. What it shows: the most technically detailed public disclosure of disaggregation economics at genuine hyperscale, and the clearest evidence that the KV cache itself — not just GPU count — is the resource worth architecting around. <https://arxiv.org/abs/2407.00079> (also <https://www.usenix.org/system/files/fast25-qin.pdf>)

- **DeepSeek — V3/R1 Inference System technical disclosure** (DeepSeek's own GitHub, Open Source Week 2025). Discloses the production prefill/decode split directly: prefill deployment units of 4 nodes each (Expert-Parallel-32), decode deployment units of 18 nodes each (Expert-Parallel-144), on H800 GPUs, sustaining roughly 73.7k input tok/s and 14.8k output tok/s per node. What it shows: a named frontier lab publishing its own real disaggregation topology and throughput numbers, not a third-party benchmark or a marketing claim. <https://github.com/deepseek-ai/open-infra-index/blob/main/202502OpenSourceWeek/day_6_one_more_thing_deepseekV3R1_inference_system_overview.md>

- **LMSYS Org — "Deploying DeepSeek with PD Disaggregation and Large-Scale Expert Parallelism on 96 H100 GPUs"** (2025). An independent, fully open-source SGLang reproduction of DeepSeek-style disaggregation: 12 nodes (96 H100s total) split 3 prefill : 9 decode nodes, reaching 52.3k input tok/s and 22.3k output tok/s per node — nearly matching DeepSeek's own reported throughput — at an implied self-hosted cost of about $0.20 per million output tokens, roughly a fifth of DeepSeek's own API price at the time, and up to 5× the output throughput of a vanilla tensor-parallel deployment on the same hardware. What it shows: the paper-to-practice bridge — a third party reproducing frontier-lab-style disaggregation with fully public code, which is directly useful as a reference architecture. <https://www.lmsys.org/blog/2025-05-05-large-scale-ep/>

## Worked example

A team runs a customer-support RAG assistant: every request carries a **3,500-token shared system prompt + retrieved policy docs**, then a short user turn and a ~200-token answer. Traffic is 300 concurrent sessions.

Reasoning through the choices:

- **Prefix overlap is huge** (the 3,500-token preamble repeats on nearly every request) → **SGLang / RadixAttention** or vLLM with prefix caching is the engine layer; the shared prefix is prefilled once, so per-request TTFT and CPM drop sharply versus re-prefilling 3,500 tokens each time.
- **Prefill-heavy, decode-light** (3,500 in / 200 out) → strong disaggregation candidate: a small decode pool, a larger prefill pool, plus **KV-aware routing** so repeat prefixes hit a warm worker. This is the Baseten-style shape — long context, latency-sensitive, and the P95/P99/RPS deltas Baseten reported apply directly to a workload built this way.
- **Platform delivery** across the fleet → wrap it in **KServe** for autoscaling/canary, with SGLang as the runtime inside the `InferenceService`. Note KServe's own guidance: this is a Standard Deployment (GPU-resident) service, not a Knative scale-to-zero one — a 3,500-token-prefill model is not a good scale-to-zero candidate given cold-start cost (lesson 08/09).

Contrast: the same team's **nightly ticket-summarisation batch** — every ticket unique, no shared prefix. Here RadixAttention buys nothing; **vLLM** at a large batch optimised purely for throughput (lesson 05's cost curve) wins, and disaggregation is unjustified overhead — this traffic never approaches the GPU counts where Mooncake- or DeepSeek-scale disaggregation pays for its own coordination cost. Same team, two workloads, two answers.

## Practice

Reuse the lesson-05 harness and rented GPU. Goal: prove the prefix-sharing effect in CPM terms and produce the engine decision matrix for the deliverable.

**1. Build a shared-prefix dataset.** Construct ~500 prompts that all begin with the *same* ~2,000-token preamble followed by a short unique tail. Keep output length fixed (e.g. 128).

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

**4. Compute CPM** for each run with the lesson-05 equation and your GPU's `hourly_rate`. Then swap in a **no-shared-prefix** dataset (all-unique prompts) and rerun both — you should see the SGLang advantage collapse.

**Acceptance.**
- A **vLLM-vs-SGLang CPM comparison** table/chart on the shared-prefix workload (and the control run showing the gap vanishing with unique prompts), added to the cost-per-token deliverable.
- A **one-page engine decision matrix**: rows = {vLLM, SGLang, TensorRT-LLM, Triton, KServe, TGI}, columns = {best workload shape, key mechanism, when to pick, when NOT to}, with TGI explicitly marked "legacy only — don't pick for new work."

## Common pitfalls

- **Treating engine choice as a leaderboard/benchmark-bragging-rights question.** The whole point of the decision matrix is that "which engine is fastest" has no single answer — different workload, different team, same company, two right answers (see Worked example). A candidate who names one engine as universally best hasn't internalized the lesson.
- **Assuming disaggregation is worth it once you know it exists.** The Mooncake and DeepSeek numbers are eye-catching, but they were achieved at GPU counts most teams never operate at (hundreds of GPUs, dedicated KV-transfer fabric, two autoscalers). Proposing disaggregation for a 4-GPU deployment — paying its fixed coordination cost with no fleet scale to amortise it against — is a common junior mistake; the giveaway question is always "at what GPU count does the interconnect and orchestration overhead pay for itself here?"
- **Forgetting that RadixAttention and vLLM's own prefix caching have converged significantly.** Recommending SGLang purely for prefix sharing without checking whether vLLM's Automatic Prefix Caching (lesson 03 §4) already covers your access pattern is a stale-information mistake — the gap that existed in 2023–24 has narrowed for linear (non-branching) shared prefixes.
- **Citing an engine comparison article as ground truth because it "looks neutral."** A comparison piece that still lists TGI favorably on a cherry-picked benchmark is not automatically wrong, but it is not automatically current or representative either — re-run the comparison on your own workload before trusting a third-party leaderboard, the same discipline lesson 05 demanded of any CPM number.
- **Promising scale-to-zero for a GPU-resident, long-prefill service just because KServe supports Knative.** KServe's own docs steer GPU-heavy generative workloads toward Standard Deployment, not the serverless/Knative path — scale-to-zero cold-start cost (lesson 09) usually dominates for a model with a multi-thousand-token warm prefix.

## Self-check

**(a) When does RadixAttention / prefix-sharing NOT help — what workload?**

**Answer:** Workloads with **unique, non-overlapping prompts** and no repeated prefix: batch summarisation of distinct documents, translation of independent inputs, classification of unrelated records — any high-entropy prompt stream. With nothing shared across requests there is nothing to cache, so the radix tree adds bookkeeping without a hit-rate payoff, and SGLang lands at essentially the same CPM as vLLM. Prefix sharing only pays when requests actually reuse leading tokens (shared system prompt, few-shot block, conversation history).

**(b) Why disaggregate prefill from decode into separate pools — what does it let you scale independently?**

**Answer:** Prefill is compute-bound (sets TTFT) and decode is memory-bandwidth-bound (sets TPOT); fused in one instance they share a batch and interfere (long prefills stall decode, and one batch size can't be optimal for both). Separate pools let you **scale prefill capacity to prompt volume/length and decode capacity to concurrent generation load independently** — two autoscalers, two operating points, each phase at its own optimal batch and even its own GPU SKU. The cost is moving the KV cache between pools, which only pays off over a fast interconnect (RDMA/NVLink via NIXL) and at scale — Baseten reported 48–49% P95/P99 latency reductions and 61–62% more RPS/output-tok/s with NVIDIA Dynamo doing exactly this plus KV-aware routing.

**(c) Which engine for a K8s inference PLATFORM serving many models, and why?**

**Answer:** **KServe.** It is the Kubernetes control-plane/CRD layer (`InferenceService`), not an engine — it orchestrates a runtime (vLLM, Triton, etc.) with declarative GitOps management, per-model autoscaling including scale-to-zero for suitable workloads (Knative), canary rollouts, and a standard inference protocol across many models and tenants. An engine alone (vLLM/SGLang/TRT-LLM) serves one model well but gives you no multi-model platform layer; KServe supplies that and runs those engines underneath. For a mixed-framework zoo behind one endpoint, **Triton** is the runtime you'd wrap inside KServe.

**(d) A colleague proposes disaggregating prefill/decode for a 4-GPU RAG service. What's the objection?**

**Answer:** The production disaggregation numbers that justify the technique (Mooncake's 59–498% capacity gain, DeepSeek's 4-node-prefill/18-node-decode topology, LMSYS's 96-H100 reproduction) all come from fleets of dozens to hundreds of GPUs, where the fixed cost of a KV-transfer fabric (RDMA/NVLink via NIXL), two separate autoscalers, and two failure domains is amortised across enough traffic to be worth it. At 4 GPUs that fixed coordination cost has almost nothing to amortise against — a single fused vLLM instance (with prefix caching if the workload has shared prefixes) is simpler, cheaper, and very likely faster to ship. The right question to ask back is "at what GPU count does the interconnect/orchestration overhead pay for itself here," and 4 is well below it.

## Connections & what's next

This lesson closes the loop lesson 05 opened: the CPM curve there assumed one engine and one fused pool; here you learned when that assumption itself is the thing to change — a different engine for a different prefix-overlap profile, or two pools instead of one for a different phase balance. It also completes the KV-cache arc that has run since module 03: PagedAttention (lesson 03) made the KV cache a manageable resource inside one engine; RadixAttention (this lesson) made it shareable across requests within that engine; disaggregation and KV-aware routing (this lesson) made it a fleet-wide, transportable resource. [Lesson 07 — Quantization ops](07-quantization-ops.md) is next, and it is largely orthogonal to everything here — FP8 vs. FP16 is a per-GPU numerics decision that stacks on top of whichever engine and topology you chose in this lesson, so expect the deliverable to combine both: your engine/topology choice from here, quantized, swept across batch sizes as in lesson 05.

## References & further reading

- **Primary sources**
  - Zheng et al. — "SGLang: Efficient Execution of Structured Language Model Programs" (NeurIPS 2024, arXiv 2312.07104) — the RadixAttention paper itself; read this before citing RadixAttention's numbers secondhand. <https://arxiv.org/abs/2312.07104>
  - LMSYS Org — "Fast and Expressive LLM Inference with RadixAttention and SGLang" — the original announcement blog, more accessible than the paper for a first read. <https://www.lmsys.org/blog/2024-01-17-sglang/>
  - NVIDIA Dynamo — disaggregated serving docs and KV-cache-aware routing docs — the reference architecture for separate prefill/decode pools and prefix-locality routing. <https://docs.nvidia.com/dynamo/user-guides/disaggregated-serving> · <https://docs.nvidia.com/dynamo/user-guides/kv-cache-aware-routing>
  - NVIDIA TensorRT-LLM — official documentation (architecture, LLM API, parallelism strategies). <https://nvidia.github.io/TensorRT-LLM/overview.html> (mirrored at <https://docs.nvidia.com/tensorrt-llm/index.html>)
  - KServe — "Knative Serverless Installation Guide" and "Control Plane" architecture docs — the source for the Standard-vs-Serverless deployment distinction used in this lesson. <https://kserve.github.io/website/docs/admin-guide/serverless> · <https://kserve.github.io/website/docs/concepts/architecture/control-plane>

- **Real-world engineering blogs**
  - Baseten — "2x faster inference with KV cache-aware routing" — the audited P95/P99/RPS/throughput deltas cited in Real-world use cases and self-check (b). <https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/>
  - Qin et al. (Moonshot AI / Tsinghua) — "Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving" (USENIX FAST '25, arXiv 2407.00079) — the deepest technical disclosure of disaggregation economics at hyperscale. <https://arxiv.org/abs/2407.00079> · <https://www.usenix.org/system/files/fast25-qin.pdf>
  - DeepSeek-AI — "DeepSeek-V3/R1 Inference System Overview" (Open Source Week, 2025) — the frontier lab's own disclosed prefill/decode topology and throughput. <https://github.com/deepseek-ai/open-infra-index/blob/main/202502OpenSourceWeek/day_6_one_more_thing_deepseekV3R1_inference_system_overview.md>
  - LMSYS Org — "Deploying DeepSeek with PD Disaggregation and Large-Scale Expert Parallelism on 96 H100 GPUs" — the open-source paper-to-practice reproduction. <https://www.lmsys.org/blog/2025-05-05-large-scale-ep/>

- **Deeper dives**
  - Hao AI Lab (UC San Diego) — "Disaggregated Inference: 18 Months Later" — the original DistServe research team's retrospective on how prefill/decode disaggregation moved from a 2023–24 research idea to a 2025 production default across nearly every major serving framework. <https://haoailab.com/blogs/distserve-retro/>
