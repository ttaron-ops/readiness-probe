---
lesson: "07.9"
title: "Model loading and storage"
module: "07"
concept: "Model loading and storage"
status: not-started
est_time: "5h"
artifacts: []
---
# 07.9 · Model loading and storage

> **Concept.** Cold-start latency is a storage-architecture problem, and storage architecture is what decides whether scale-to-zero saves money or just breaks your SLO.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

Lesson 08 sold you scale-to-zero: idle GPUs cost nothing if you turn them off.
That math only holds if you can turn them back **on** fast enough. The GPU meter
is $2–$12/hr whether the card is decoding tokens or sitting empty, so the tempting
move at 70–90% idle is to drop replicas to zero. The catch nobody costs out until
production: the first request after idle now pays the *full cold start* — schedule
a pod, pull a multi-GB image, move tens to hundreds of GB of weights into VRAM,
and warm the engine. Done wrong, that user waits **20+ minutes** for a first token.
Done right, they wait **45 seconds**.

That spread is entirely determined by *where the weights live*. This lesson turns
cold start from a mystery latency spike into a line-item budget you can defend to
both SRE (SLO) and finance (idle-GPU savings). For the Senior Platform Engineer
role, "we made scale-to-zero viable on 70B by caching weights on local NVMe, cold
start 18 min → 50 s" is exactly the storage-economics story that separates you.

## What's new here

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

## Core notes

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
A faster GPU loads weights no faster if the bottleneck is a 200 MB/s network pull.

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

### Image pull vs weight load are separate cost components — fix them separately

They are two different byte streams from two different sources, and conflating them
is the #1 cold-start mistake:

- **Image pull (component 2)** = the *container* (CUDA runtime, PyTorch, vLLM,
  Python deps). A vLLM image is ~5–10 GB. Fix: pre-pull onto nodes
  (`imagePullPolicy: IfNotPresent` + a warmer DaemonSet), use a registry mirror /
  pull-through cache in-region, and **do not bake weights into the image** (below).
- **Weight load (component 3)** = the *model*. Fix: node-local NVMe cache or a
  pre-populated read-only PVC.

**Baking weights into the container image is an anti-pattern.** It feels fast
("everything's in the image") but it makes the image 140 GB+, so component 2 now
carries the weight-download cost — and you re-pull it on every new node, can't share
one copy across models, and rebuild the whole image to swap a checkpoint. Keep the
image lean (code only) and let a storage volume own the weights. The one exception:
small models (≤ a few GB) where a baked image on a fast registry beats managing a
volume.

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
node + local NVMe you can push toward the ~10–20 s range that even some interactive
tiers accept behind a "warming up…" state. **Storage architecture is the lever that
moves you across that viability line.** Same model, same GPU, same autoscaler — only
the weight-storage tier changed, and scale-to-zero went from impossible to profitable.

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

**Acceptance:** a cold-start breakdown table (cached vs uncached), each with the
three sub-components summed to a total first-token time, committed under
`practice/cost-per-token/` and referenced by the cost model. It must show weight-load
as the swing factor and state the resulting scale-to-zero verdict (viable / not) at
your workload's idle % and wake-up count.

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

## Resources

1. **Spheron — "KEDA + Knative GPU autoscaling on Kubernetes: solving LLM cold start"**
   — the canonical write-up tying scale-to-zero autoscaling to the cold-start
   breakdown and weight-cache remedies:
   https://www.spheron.network/blog/keda-knative-gpu-autoscaling-kubernetes-llm-cold-start/
2. **vLLM engine/runtime docs (v0.26.0)** — weight loading, `--load-format`,
   `HF_HUB_OFFLINE`, and CUDA-graph warmup behavior you'll read in the logs:
   https://docs.vllm.ai/en/latest/configuration/engine_args/
