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
sources: 9
---
# 07.9 · Model loading and storage

> **Concept.** Cold-start latency is a storage-architecture problem, and storage architecture is what decides whether scale-to-zero saves money or just breaks your SLO.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

Lesson 08 gave you the *decision*: scale to zero when idle savings beat the cold-start
hit, keep a warm floor otherwise. It priced the tradeoff with a placeholder cold-start
number. This lesson opens that number up — decomposes it into four components, shows
which one is the swing factor, and gives you the storage-architecture levers that turn
a 20-minute cold start into a 45-second one. Lesson 10 then asks a related but distinct
question: once a model *is* loaded, how many logical models can you serve from that one
resident copy? Both lessons are about getting more served traffic per GPU-dollar without
adding GPUs — 09 attacks the *wake-up* cost, 10 attacks the *steady-state* multiplexing.

## Why this matters

Lesson 08 sold you scale-to-zero: idle GPUs cost nothing if you turn them off.
That math only holds if you can turn them back **on** fast enough. The GPU meter
is $2–$12/hr whether the card is decoding tokens or sitting empty, so the tempting
move at 70–90% idle is to drop replicas to zero. The catch nobody costs out until
production: the first request after idle now pays the *full cold start* — schedule
a pod, pull a multi-GB image, move tens to hundreds of GB of weights into VRAM,
and warm the engine. Done wrong, that user waits **20+ minutes** for a first token.
Done right, they wait **45 seconds**. The most aggressive production systems get it
under **15 seconds**.

That spread is entirely determined by *where the weights live*. This lesson turns
cold start from a mystery latency spike into a line-item budget you can defend to
both SRE (SLO) and finance (idle-GPU savings). For the Senior Platform Engineer
role, "we made scale-to-zero viable on 70B by caching weights on local NVMe, cold
start 18 min → 50 s" is exactly the storage-economics story that separates you.

## What's new here (calibration)

Everything before this lesson assumed the server was **already running**. Batching,
PagedAttention, quantization, autoscaling — all steady-state. This lesson is about
the **transition from zero**, which has completely different cost drivers:

- **Cold start is not one number.** It's four sequential components, each with a
  different fix. Optimizing the wrong one wastes weeks.
- **Storage tier, not model size, dominates.** The same 70B model cold-starts in
  45 s or 18 min depending only on whether weights sit on local NVMe or come over
  the network from Hugging Face.
- **Scale-to-zero has a break-even you can compute.** Idle fraction, cold-starts
  per day, and cold-start duration decide whether zeroing out saves money or costs
  it (in GPU-seconds *and* in blown latency SLOs).
- **There's a tier beyond weight caching.** Production serverless platforms now
  snapshot the *entire post-load GPU state* (weights in VRAM, CUDA context, compiled
  kernels) and restore it byte-for-byte, skipping weight loading altogether. Know
  this exists even if your practice run only gets as far as NVMe caching.

## Core concepts

### The four components of a cold start

Measure scale-from-zero → first token, and it always decomposes into these, in order:

| # | Component | What happens | Typical range | Dominant when |
|---|-----------|--------------|---------------|---------------|
| 1 | **Pod schedule + GPU bind** | Scheduler places pod; device plugin binds GPU; (if node is cold) cluster-autoscaler provisions a GPU node | 5–30 s (warm node) · 2–8 min (new node) | Node pool at zero |
| 2 | **Image pull** | containerd pulls the server image + CUDA/PyTorch layers, decompresses to disk | ~0–5 s cached · 2–6 min uncached | First pod on a node |
| 3 | **Weight load** | Read weights from storage → host RAM → **VRAM** (`safetensors` mmap + H2D copy) | 15 s – 20 min | **Storage tier** |
| 4 | **Engine warmup** | CUDA context, KV-cache profiling run, CUDA-graph capture / `torch.compile`, tokenizer, sampler | 20–120 s | Always present |

The trap: teams see "20-minute cold start" and reach for a bigger GPU or more
replicas. Neither touches components 2–3, which are **I/O-bound, not compute-bound**.
A faster GPU loads weights no faster if the bottleneck is a 200 MB/s network pull —
as Runpod puts it bluntly in their own FlashBoot post-mortem, "you cannot reduce
weight-to-VRAM transfer time below the storage tier's bandwidth." This is a physical
constraint, not a scheduling problem, and it's why the fix is always architectural
(move the bytes closer / use a faster reader), never "add compute."

### Where the minutes actually go: weight load is the swing factor

Model weights are the biggest bytes you move, and the storage tier sets the bandwidth.
Weight size at rest (this is why quantization from Lesson 07 pays off *twice* — VRAM
*and* load time):

| Model | fp16 | fp8/int8 | int4 (AWQ/GPTQ) |
|-------|------|----------|-----------------|
| 8B | 16 GB | 8 GB | ~5 GB |
| 70B | 140 GB | 70 GB | ~35 GB |

Now divide by realistic read bandwidth:

| Storage source | Bandwidth (real) | 70B fp16 (140 GB) | 70B fp8 (70 GB) | 8B fp16 (16 GB) |
|----------------|------------------|-------------------|-----------------|-----------------|
| HF Hub, uncached (internet) | 100–300 MB/s | **8–23 min** | 4–12 min | 55 s – 2.7 min |
| Network PVC (RWX, e.g. EFS/Filestore) | 300 MB/s – 1 GB/s | 2.5–8 min | 1–4 min | 16–55 s |
| Cloud block PVC (EBS gp3/io2) | 0.5–2 GB/s | 70 s – 4.5 min | 35 s – 2.5 min | 8–32 s |
| **Local NVMe SSD (node-local cache)** | **2–6 GB/s** | **23–70 s** | **12–35 s** | **3–8 s** |
| Already in host page cache (warm node) | 5–12 GB/s | 12–28 s | 6–14 s | 1.5–3 s |

Two structural facts drive the whole lesson:

1. **Uncached HF pull is the worst case and the default.** A fresh pod with no
   volume does `download → disk → RAM → VRAM`. For 70B that's a *20-minute* first
   token. This alone kills scale-to-zero.
2. **Local NVMe collapses component 3 to seconds.** A DaemonSet or init pattern that
   pre-stages weights onto each GPU node's local SSD (or an already-attached
   read-only PVC) turns the same 70B into a *sub-minute* load.

### Purpose-built loaders beat the default reader — same storage, faster bytes

The bandwidth table above assumes a naive reader. In practice the *loader* itself is
a lever independent of the storage tier, because the default path (HF's Python
loader → `safetensors` mmap → per-tensor H2D copy) leaves throughput on the table.
Two production tools attack this directly, and both are now wired natively into
vLLM:

- **CoreWeave's Tensorizer** streams a serialized model directly from S3/HTTP (or
  local disk) straight into GPU memory with a zero-copy path, skipping the
  intermediate "download to local filesystem, then load" step entirely. CoreWeave's
  own benchmarks show **>5x faster average latency versus the default Hugging Face
  loader when scaling from zero**, at wire-speed (~5 GB/s) reads from S3 — i.e. it
  gets object-storage bandwidth into a similar range as a local NVMe cache without
  you having to manage a cache at all.
- **NVIDIA's Run:ai Model Streamer** (open-source Python SDK, integrated into both
  vLLM and SGLang) concurrently reads weight shards from storage while streaming
  them into GPU memory, overlapping I/O with the H2D copy instead of doing them
  serially. Independent benchmarks from Microsoft's own Azure AKS engineering blog
  corroborate roughly a **6x load-time reduction** pairing the streamer with Azure
  Blob Storage, and Google Cloud has published GKE integration guidance — evidence
  this is becoming a cross-cloud default rather than one vendor's trick.

The practical takeaway: **don't rely on the reader that ships as the framework
default.** For any workload where cold-start time matters, swap in a
concurrency/zero-copy-aware loader before you invest in exotic storage hardware —
it's often the higher-leverage, lower-effort fix.

### Image pull vs weight load are separate cost components — fix them separately

They are two different byte streams from two different sources, and conflating them
is the #1 cold-start mistake:

- **Image pull (component 2)** = the *container* (CUDA runtime, PyTorch, vLLM,
  Python deps). A vLLM image is ~5–10 GB. Fix: pre-pull onto nodes
  (`imagePullPolicy: IfNotPresent` + a warmer DaemonSet), use a registry mirror /
  pull-through cache in-region, and **do not bake weights into the image** (below).
- **Weight load (component 3)** = the *model*. Fix: node-local NVMe cache, a
  pre-populated read-only PVC, or a streaming loader (above).

**Baking weights into the container image is an anti-pattern.** It feels fast
("everything's in the image") but it makes the image 140 GB+, so component 2 now
carries the weight-download cost — and you re-pull it on every new node, can't share
one copy across models, and rebuild the whole image to swap a checkpoint. Keep the
image lean (code only) and let a storage volume or streaming loader own the weights.
The one exception: small models (≤ a few GB) where a baked image on a fast registry
beats managing a volume.

### Weight format matters too: why safetensors is the default worth defaulting to

The mmap-and-stream path above assumes the on-disk format supports it. `safetensors`
(vs. legacy PyTorch `.bin`/pickle checkpoints) is fast *because* it's a flat,
header-plus-raw-tensor-bytes layout that supports zero-copy mmap and lazy, tensor-at-
a-time loading — no arbitrary Python object graph to deserialize. It's also the
*safe* choice: a joint 2023 security audit by Trail of Bits, commissioned by Hugging
Face, EleutherAI, and Stability AI, found no critical remote-code-execution flaw in
the format (unlike pickle-based checkpoints, which execute arbitrary code on load) and
the fixes from that audit shipped upstream. Fast and safe turning out to be the same
format choice is not a coincidence — pickle's danger comes from the same
arbitrary-deserialization flexibility that also makes it slow and hard to
stream/mmap. If you inherit a pipeline still shipping `.bin` checkpoints, converting
to `safetensors` is close to a free win on both cold-start time and supply-chain risk.

### The next tier beyond weight caching: GPU memory snapshotting

Everything above still pays the cost of moving bytes from storage into VRAM at
least once per cold start. Serverless GPU platforms are now pushing past that
floor entirely. Modal's GPU memory snapshotting captures the **full post-load GPU
state** — weights already resident in VRAM, the CUDA context, compiled kernels/CUDA
graphs — as a byte-for-byte snapshot after first initialization, then *restores*
that snapshot on subsequent cold starts instead of re-running load + warmup at all.
Modal reports cutting a vLLM server's cold start from roughly two minutes to about
ten seconds this way (~10x), and a follow-up post benchmarking a Mistral 3 model
showed a similar ~10x cut (from ~118 s to ~12 s median). This is a materially more
aggressive technique than anything in the table above: it doesn't optimize component
3 (weight load), it **skips it**, collapsing components 3 and most of 4 into a single
memory-restore operation. Treat this as "know it exists, and know why it's a step
beyond NVMe caching" — building it yourself is a serverless-platform engineering
project, not a config flag, so it's out of scope for the practice below, but naming
it correctly is worth real interview credit.

### Why storage architecture *makes* scale-to-zero viable

Scale-to-zero's saving is the idle GPU-hours you stop paying for. Its cost is paid
on every wake-up, in two currencies:

1. **GPU-seconds** — you pay for the card during the whole cold start (it's
   allocated, just not serving).
2. **Latency SLO** — the waking request eats the full cold start as TTFT.

The SLO currency is usually the binding constraint and it's brutal: **a 20-minute
cold start means scale-to-zero is off the table** for any interactive workload,
full stop — no user waits 20 minutes. A **45-second** cold start is inside the
tolerance of a batch/async or "spin up on queue depth" pattern, and with a warm
node + local NVMe (or a streaming loader) you can push toward the ~10–20 s range
that even some interactive tiers accept behind a "warming up…" state, and snapshot
restore pushes it under 15 s. **Storage architecture is the lever that moves you
across that viability line.** Same model, same GPU, same autoscaler — only the
weight-storage tier changed, and scale-to-zero went from impossible to profitable.

## Perspectives

**The neocloud tool-builder view (CoreWeave).** CoreWeave built and open-sourced
Tensorizer specifically to sell faster cold starts as a GPU-cloud product
differentiator. What used to be a nice-to-have — "our platform loads models fast" —
is now table stakes in the neocloud market; a provider that can't demonstrate
sub-minute cold starts on a 70B is at a competitive disadvantage in RFPs against one
that can.

**The serverless-platform view (Modal, Runpod).** GPU memory snapshotting represents
a step change beyond weight-caching, not an incremental improvement to it: it
skips weight loading rather than accelerating it, which changes the cold-start
budget math from "which storage tier" to "do we even re-load." Flag this as *beyond*
what this lesson's practice covers, but strategically important — it's where the
frontier is moving.

**The cross-cloud/vendor-neutral view (NVIDIA).** Run:ai Model Streamer's
integration across AWS/Azure/GCP blob storage *and* both vLLM and SGLang makes it
the closest thing to an emerging standard in this space — not tied to one neocloud's
proprietary stack. If you're picking a default loader to recommend on a platform team,
this cross-vendor, cross-engine support is a strong argument for making it the
headline tool rather than a vendor-specific one.

**The physical-constraint view (Runpod).** Whatever loader or cache tier you pick,
you cannot move bytes from storage into VRAM faster than that storage tier's
bandwidth allows. FlashBoot-style "keep pre-warmed workers around" sidesteps the
loading problem rather than solving it — and it reintroduces the idle-cost tradeoff
from Lesson 08 (a pre-warmed worker is a worker you're paying for). Useful as a
sanity check: any pitch to "eliminate cold start" is really either (a) moving bytes
onto faster storage, (b) reading them more efficiently, or (c) paying to keep them
already loaded — there is no fourth option.

## Real-world use cases

- **CoreWeave — "Decrease PyTorch Model Load Times with CoreWeave's Tensorizer"**
  ([coreweave.com/blog](https://www.coreweave.com/blog/coreweaves-tensorizer-decrease-pytorch-model-load-times))
  — what it shows: a neocloud's own weight-serialization format/loader (open source,
  now integrated into vLLM) cutting scale-from-zero latency >5x versus the default
  HF loader by streaming zero-copy from S3/HTTP at wire speed.
- **NVIDIA Developer Blog — "Reducing Cold Start Latency for LLM Inference with
  NVIDIA Run:ai Model Streamer"**
  ([developer.nvidia.com/blog](https://developer.nvidia.com/blog/reducing-cold-start-latency-for-llm-inference-with-nvidia-runai-model-streamer/))
  — what it shows: an open-source, engine-agnostic (vLLM + SGLang) streaming loader
  with results corroborated independently on Azure Blob Storage (~6x) by Microsoft's
  own AKS engineering blog — evidence this pattern is now cross-cloud infrastructure,
  not one company's benchmark.
- **Modal — "GPU Memory Snapshots: Supercharging sub-second startup" /
  "Modal + Mistral 3: 10x faster cold starts with GPU snapshotting"**
  ([modal.com/blog/gpu-mem-snapshots](https://modal.com/blog/gpu-mem-snapshots),
  [modal.com/blog/mistral-3](https://modal.com/blog/mistral-3)) — what it shows: the
  next tier beyond weight caching — snapshotting full post-load GPU state (VRAM
  contents, CUDA context, compiled kernels) and restoring it, cutting a ~2-minute
  cold start to ~10 seconds (~10x) by skipping weight loading on restore.
- **Runpod — "Cold starts were never the real problem"**
  ([runpod.io/blog](https://www.runpod.io/blog/serverless-gpu-cold-starts-flashboot))
  — what it shows: a serverless-GPU vendor's own retrospective on why pre-warmed-pool
  strategies (FlashBoot) trade cold-start latency for idle cost rather than
  eliminating the tradeoff, and the blunt framing that weight-to-VRAM transfer time
  is bounded below by storage-tier bandwidth, full stop.

## Worked example

**Scenario.** Llama-3.1-70B, fp8, on 1× H100 (80 GB) at $2.50/GPU-hr (committed
cloud). Interactive-ish internal tool: **80% idle** over the day, roughly **12
wake-ups/day** (bursty usage after idle gaps). vLLM `0.26.0`.

**Cold-start breakdown, uncached (HF pull) vs cached (local NVMe):**

| Component | Uncached (HF pull) | Cached (local NVMe) |
|-----------|--------------------|--------------------|
| 1 Pod schedule + GPU bind (warm node) | 15 s | 15 s |
| 2 Image pull (8 GB vLLM image) | 3 min (uncached) | 3 s (node-cached) |
| 3 Weight load (70 GB fp8) | 8 min (150 MB/s) | 20 s (3.5 GB/s NVMe) |
| 4 Engine warmup (CUDA graph + profile) | 60 s | 60 s |
| **Total (first token)** | **~12.5 min** | **~98 s** |

**Does scale-to-zero beat always-on?** GPU-cost side only, per day:

- Always-on: 24 h × $2.50 = **$60.00/day**.
- Scale-to-zero active GPU-hours = 24 × (1 − 0.80) = 4.8 h → $12.00.
- Cold-start GPU overhead = 12 wakeups × cold-start-hours × $2.50.
  - Uncached: 12 × (12.5/60) × $2.50 = 12 × 0.208 × $2.50 = **$6.25/day**.
  - Cached: 12 × (98/3600) × $2.50 = 12 × 0.0272 × $2.50 = **$0.82/day**.
- Scale-to-zero total: uncached **$18.25/day**, cached **$12.82/day**.

Both *look* cheaper than $60 on GPU-cost alone — but the **uncached case is a lie**:
its 12.5-min TTFT violates any interactive SLO, so in practice you'd be forced back
to always-on and pay the full $60. The **cached case delivers the $47/day (79%)
saving for real**, because a 98 s cold start is inside a "spin-up on demand"
tolerance. The storage tier didn't just shave latency — it's the difference between
a paper saving and a banked one.

**Feeding the deliverable.** Amortized into cost-per-million-tokens: the cached
cold-start overhead is $0.82/day over, say, 40M tokens/day of real traffic =
**$0.02/M tokens** — a rounding error. The uncached overhead ($6.25/day) plus the
forced-always-on penalty is the term that would blow up your unit economics.

## Practice

**Goal:** measure end-to-end cold start (scale-from-zero → first token) *with* cached
weights vs *without*, and break it into image-pull / weight-load / engine-warmup.
Use a 70B if you have the GPU; otherwise a smaller model **proportionally** (8B is
fine — the *ratios* are the lesson).

1. **Baseline (uncached).** Deploy vLLM `0.26.0` as a Knative/KEDA scale-to-zero
   service with **no weight volume** (let it pull from HF on start). Scale to zero,
   then send one request and timestamp each phase:
   ```bash
   # image: vllm/vllm-openai:v0.26.0 ; model downloads to emptyDir on first start
   kubectl get events --sort-by=.lastTimestamp   # Pulling/Pulled → image-pull span
   kubectl logs <pod> | grep -iE "Starting to load|Loading weights|took|CUDA graph|init engine|Application startup complete"
   # wall-clock: t0 = pod Scheduled ; t_ready = first 200 on /v1/completions
   ```
   vLLM logs "Loading weights took N seconds" and CUDA-graph capture timing — use
   those to split component 3 from component 4.
2. **Cached run.** Mount weights from a node-local NVMe path (or a pre-populated
   read-only PVC): set `HF_HUB_OFFLINE=1` and point `--model /mnt/nvme/<model>`.
   Pre-pull the image onto the node. Repeat the exact measurement.
3. **Break it down.** For each run, attribute wall-clock to: (2) image pull,
   (3) weight load, (4) engine warmup. `nvidia-smi dmon` or the vLLM logs confirm
   when VRAM fills (component 3) vs when graphs capture (component 4).
4. **(Stretch)** Swap the default loader for a faster one (`--load-format
   runai_streamer` if your vLLM build has it, or a Tensorizer-serialized checkpoint)
   and re-measure component 3 against the NVMe-cache baseline — is the loader or the
   storage tier your bigger lever at your scale?

**Acceptance:** a cold-start breakdown table (cached vs uncached), each with the
three sub-components summed to a total first-token time, committed under
`practice/cost-per-token/` and referenced by the cost model. It must show weight-load
as the swing factor and state the resulting scale-to-zero verdict (viable / not) at
your workload's idle % and wake-up count.

## Common pitfalls

1. **Reaching for a bigger/faster GPU to fix a cold start.** Components 2–3 are
   I/O-bound; a faster H100 doesn't pull bytes off S3 any faster. Fix the storage
   path, not the compute.
2. **Baking weights into the container image "for simplicity."** It moves the
   weight-download cost into the image-pull component, forces a re-pull per new
   node, and turns a checkpoint swap into a full image rebuild. Keep weights in a
   separate, cacheable volume or stream.
3. **Conflating image-pull bandwidth with weight-load bandwidth.** They're
   different byte streams from different sources needing different fixes (registry
   mirror/pre-pull vs. NVMe cache/streaming loader) — diagnosing "cold start is
   slow" without splitting the two wastes remediation effort on the wrong one.
4. **Treating NVMe caching as the ceiling.** It's the standard production fix, but
   GPU-memory snapshotting (Modal) is a materially more aggressive next tier that
   skips weight loading entirely — know it exists so you don't over-claim "we've
   optimized this as far as it goes" in an interview or a design doc.
5. **Underestimating first-pod-on-a-node image pull in an aggressively
   consolidating cluster.** Lesson 08's Karpenter consolidation actively removes
   idle GPU nodes to save cost — which means the *next* wake-up is more likely to
   land on a brand-new node with a cold image cache, compounding component 2 right
   when you thought you'd already paid that cost once.

## Self-check

**(a)** At X% idle, what cold-start budget must scale-to-zero beat to save money vs
always-on — reason it out?
**Answer:** On GPU-cost alone, savings/day = `24·f·C − N·t_cold·C` where `f` = idle
fraction, `C` = $/GPU-hr, `N` = wake-ups/day, `t_cold` = cold-start hours. That's
positive when **`t_cold < 24·f / N`** — e.g. f=0.8, N=12 → `t_cold < 1.6 h`, so on
pure GPU cost almost anything wins. The *real* budget is set by the **latency SLO**,
not this inequality: an interactive tier caps `t_cold` at seconds-to-low-minutes
regardless of the cost math, because no user waits out a 12-min wake-up. So the
binding budget is `min(24·f/N, SLO tolerance)`, and the SLO term almost always binds.

**(b)** Where do the minutes of a cold start actually go — name the components?
**Answer:** Four sequential phases: (1) pod schedule + GPU bind (seconds warm,
minutes if a GPU *node* must be provisioned); (2) container **image pull** (the
CUDA/PyTorch/vLLM image, ~5–10 GB); (3) **weight load** — read the model from
storage into VRAM, the swing factor, 15 s to 20 min purely by storage tier; and
(4) **engine warmup** — CUDA context, KV-cache profiling, CUDA-graph/`torch.compile`
capture, tokenizer/sampler init (20–120 s, roughly fixed). Image pull and weight
load are separate byte streams with separate fixes.

**(c)** How does a PVC/local-SSD weight cache change the scale-to-zero economics?
**Answer:** It attacks component 3 — the biggest, most variable term. Pulling 70B
fp8 (70 GB) from HF at ~150 MB/s is ~8 min; the same bytes off local NVMe at
~3.5 GB/s is ~20 s, a ~25× cut that drops total cold start from ~12 min to under
2 min. That moves the workload **across the SLO viability line**: the always-forced
"stay on" penalty disappears and the banked idle-GPU saving (e.g. 79% at 80% idle)
becomes real. The cache doesn't just speed things up — it's the enabling condition
that makes scale-to-zero economically legitimate at all.

**(d)** Beyond a faster storage tier, what's another way to cut weight-load time
without changing where the bytes live?
**Answer:** Change the *loader*, not the storage. The default HF `safetensors`
loader does a fairly naive sequential read-then-copy; purpose-built loaders like
CoreWeave's Tensorizer (zero-copy stream from S3/HTTP straight into GPU memory) or
NVIDIA's Run:ai Model Streamer (concurrent read-while-streaming into GPU memory)
extract more throughput from the *same* storage tier — CoreWeave reports >5x versus
the default loader, Run:ai Model Streamer roughly 6x on Azure Blob per Microsoft's
own benchmark. For a platform team, swapping the loader is often cheaper than
building and maintaining a new storage/caching layer.

**(e)** What's the failure mode of relying on `safetensors` format specifically, and
why is it still the right default?
**Answer:** There isn't much of a failure mode — that's the point. `safetensors` is
both the *fast* choice (flat layout, zero-copy mmap, lazy tensor-at-a-time loading,
no arbitrary object graph) and the *safe* choice (a joint Trail of Bits audit
commissioned by Hugging Face/EleutherAI/Stability AI found no critical RCE flaw,
unlike pickle-based `.bin` checkpoints which execute arbitrary code on
deserialization). The one real risk is inheriting a legacy pipeline still shipping
pickle checkpoints and not converting — that costs you both load speed and a
supply-chain attack surface for no benefit.

## Connections & what's next

This lesson closes the loop Lesson 08 opened: 08 told you *when* to scale to zero,
09 told you *how* to make the wake-up survivable. Together they're one line item in
the cost-per-million-tokens deliverable — the cold-start term is usually a rounding
error when cached and a dealbreaker when it isn't, and your CPM report should show
both numbers so the tradeoff is legible to someone who isn't you.

Next: [**10 · Multi-model serving with LoRA**](10-multi-model-lora.md). Loading is a
per-replica, per-wake-up cost; Lesson 10 asks the complementary question — once a
replica *is* warm, how many logical models can ride that one loaded copy before you
need a second GPU at all? Where this lesson collapses the cost of *starting*, Lesson
10 collapses the cost of *multiplying* — and a well-run platform needs both.

## References & further reading

- **Primary sources**
  - vLLM engine/runtime docs (v0.26.0) — weight loading, `--load-format`,
    `HF_HUB_OFFLINE`, and CUDA-graph warmup behavior you'll read in the logs:
    https://docs.vllm.ai/en/latest/configuration/engine_args/
  - NVIDIA Developer Blog — "Reducing Cold Start Latency for LLM Inference with
    NVIDIA Run:ai Model Streamer" — the primary source for the streaming-loader
    technique, natively wired into vLLM and SGLang:
    https://developer.nvidia.com/blog/reducing-cold-start-latency-for-llm-inference-with-nvidia-runai-model-streamer/
  - Hugging Face / EleutherAI — "Safetensors audited as really safe and becoming
    the default" (Trail of Bits security audit writeup) — why `safetensors` is
    both the fast and the safe format choice:
    https://github.com/huggingface/blog/blob/main/safetensors-security-audit.md
    (companion post: https://blog.eleuther.ai/safetensors-security-audit/)

- **Real-world engineering blogs**
  - CoreWeave — "Decrease PyTorch Model Load Times with CoreWeave's Tensorizer" —
    zero-copy S3/HTTP streaming into GPU memory, >5x vs. default loader:
    https://www.coreweave.com/blog/coreweaves-tensorizer-decrease-pytorch-model-load-times
  - Modal — "GPU Memory Snapshots: Supercharging sub-second startup" and
    "Modal + Mistral 3: 10x faster cold starts with GPU snapshotting" — the
    next-tier technique beyond weight caching:
    https://modal.com/blog/gpu-mem-snapshots · https://modal.com/blog/mistral-3
  - Runpod — "Cold starts were never the real problem" — the pre-warmed-pool
    tradeoff and the "bandwidth is a physical floor" framing:
    https://www.runpod.io/blog/serverless-gpu-cold-starts-flashboot
  - Microsoft Azure AKS Engineering Blog — "Stream Model Weights to NVIDIA GPU
    (vLLM) from Azure Blob Storage using the RunAI Model Streamer" — independent
    cross-cloud corroboration of the Run:ai Model Streamer numbers:
    https://blog.aks.azure.com/2026/07/13/runai-streamer-vllm

- **Deeper dives**
  - Modal — "Fast, lazy container loading in Modal.com" (talk-derived post) — the
    general lazy-loading pattern behind fast image pulls, beyond just weight
    loading: https://modal.com/blog/jono-containers-talk
