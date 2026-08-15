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
sources: 6
---
# 07.4 · vLLM in production

> **Concept.** The production knobs — `gpu-memory-utilization`, `max-num-seqs`, `max-model-len`, `tensor-parallel-size`/`pipeline-parallel-size` — are not independent dials; they trade against a single fixed HBM budget, and when you oversubscribe it the scheduler *preempts* (recompute or swap). Tuning is finding the edge of that budget without falling off it.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

---

## Where this fits

07.3 explained the mechanism that makes a large batch physically fit on a GPU — paged,
block-based KV allocation and continuous batching. This lesson is the operational half of
the same story: the flags that decide how big that batch actually gets in your deployment,
and — critically — what the system does when real traffic asks for more than you
provisioned. Where 07.3 was "how the allocator works," this lesson is "how you drive it,"
and it's the lesson where the module's cost story stops being theoretical and becomes a
config decision you own. It sets up 07.5 directly: the tuned, non-preempting configuration
you land on here is the baseline the batching-economics sweep in 07.5 is run against — an
untuned config (oversubscribed KV, or an unnecessary TP=2) would just produce a noisy,
misleading CPM curve.

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

This is also increasingly recognized as a distinct, paid specialization rather than a
generic "know your framework" skill — Red Hat and IBM now sell managed offerings built
specifically around vLLM configuration and scheduling expertise (see Real-world use cases
below). If an enterprise-procurement line item exists for "we tune your vLLM deployment for
you," that is direct market evidence this is hireable, senior-level work, not a checklist.

---

## What's new here (calibration)

Per the module README: 03 (roofline, memory-bound decode) and 05 (TTFT/TPOT, `/metrics`)
are referenced, not re-taught. This lesson is squarely "vLLM production tuning" from the
README's explicit new-content list.

- **Module 03** told you batch size is the throughput lever and KV is the constraint. Here
  you *set* the batch (`max-num-seqs`) and *size* the KV budget (`gpu-memory-utilization`,
  `max-model-len`) explicitly, and split across GPUs.
- **Module 05** gave you the SLIs (TTFT/TPOT) and `/metrics`. Here those metrics become
  the **preemption alarm**: `vllm:num_preemptions_total` climbing is the signal that you've
  oversubscribed KV, and TPOT spiking is its symptom.
- **07.3** gave you the mechanism (blocks, continuous batching). New: the operational
  envelope around it — the zero-sum HBM math and the two recovery modes when it's exceeded.

Out of scope here: the CPM-curve construction itself (07.5, next), disaggregated
prefill/decode serving as an alternative to single-process tuning (07.6), and KV-cache
quantization as a *separate* memory lever from the config knobs covered here (07.7).

---

## Core concepts

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

The distinction between these two cases — occasional vs. steady-state — is the crux of
reading the signal correctly under real traffic, and is expanded in §5 below.

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

One thing worth acknowledging even though it's out of this lesson's scope: the TP-vs-PP
crossover point is not a fixed rule — it shifts with interconnect generation. TP's
all-reduce tax that's negligible over NVLink (900 GB/s+ on an H100 node) can dominate over
a slower or older fabric, pushing the "smallest parallelism that fits" answer toward PP or
replicas sooner than the rule of thumb above suggests. Treat the guidance here as the
NVLink-node baseline, and re-derive it if you're deploying on unfamiliar interconnect.

### 5. Steady-state vs. burst preemption — the operational read

A single preemption event tells you almost nothing by itself — it's a normal response to a
momentary spike (several long-context requests happening to grow past their prompts in the
same few steps). What matters is the **trend**, read off `vllm:num_preemptions_total` over
time:

- **Occasional, isolated preemptions under a traffic burst** — fine. This is the system's
  graceful-degradation valve working as designed: it borrowed capacity from a few sequences
  to keep the fleet from OOMing, then gave it back. Don't tune against a single event.
- **A steadily climbing preemption rate under steady-state load** — you are oversubscribed.
  This is a capacity-planning problem, not a transient one: your `max-num-seqs`/
  `max-model-len` combination promises more concurrent KV than the pool can sustain at your
  *actual* traffic mix, not just its peaks.

The practical rule: look at the *slope* of `num_preemptions_total` against a flat or
gently-varying `num_requests_running`, not the raw count. A flat count with an occasional
step is a burst; a rising rate that tracks sustained load is oversubscription and needs a
config change (or more capacity), not a shrug.

---

## Perspectives

**The platform/SRE operator view.** Red Hat's OpenShift AI autoscaling material frames
this whole knob set as **day-two-ops runbook material** — something you tune in response
to an incident, not just something you set once at initial deploy and forget. Reframed
that way, `gpu-memory-utilization`, `max-num-seqs`, and the preemption counters aren't
launch-day checkboxes; they're the first three things you check when paged, and the
values you should expect to revisit as traffic shape drifts over the life of a service.

**The FinOps view.** IBM/Red Hat's decision to productize "Red Hat AI Inference" as a
managed service on IBM Cloud (bare-metal H200/MI300X, built on vLLM + the llm-d scheduler)
is a signal worth taking seriously: an enterprise is now willing to pay a premium
specifically so someone else does this tuning for them. That validates this lesson's whole
premise — vLLM production tuning is a distinct, hireable specialization with a real dollar
value attached, not a "read the docs once" skill.

**The failure-mode / on-call view.** Ground the preemption section in what it actually
looks like at 3am: `vllm:num_preemptions_total` climbing on a dashboard, TPOT alerts firing
for a subset of in-flight requests, and a `WARNING ... preempted by PreemptionMode.RECOMPUTE`
line repeating in the logs. The first thing to check is not "is the GPU broken" — it's
"is this a burst or a trend" (§5), and the first lever to reach for is `max-num-seqs` or
`max-model-len`, not `gpu-memory-utilization` (see Pitfalls — that's the wrong-instinct
knob).

**The topology view.** TP-vs-PP-vs-replica tuning is not one-size-fits-all across hardware
generations — the crossover point where TP's all-reduce tax stops being negligible shifts
with interconnect (NVLink vs. a slower or cross-node fabric). This lesson's crossover rule
assumes an NVLink node; worth a one-line gut-check ("what's the interconnect here?") before
applying it on unfamiliar hardware, even though a full topology-aware treatment is out of
scope for this lesson.

---

## Real-world use cases

- **Roblox / LinkedIn production scale** (same source as 07.3) — the 50% latency reduction
  and 4B tokens/week (Roblox) and 50+ internal use cases (LinkedIn) aren't free; they
  require exactly this kind of config tuning at scale. *What it shows:* the flags in this
  lesson are what production-scale adopters are actually turning, not academic detail.
  ([Red Hat](https://www.redhat.com/en/topics/ai/how-vllm-accelerates-ai-inference-3-enterprise-use-cases))
- **Red Hat Developer — "Autoscaling vLLM with OpenShift AI"** — covers the same
  preemption/config-tuning space from an OpenShift operator's day-two-ops point of view,
  including a follow-up performance-validation piece comparing KEDA-based autoscaling
  against default request-concurrency autoscaling. *What it shows:* a platform team's
  companion perspective to the raw config knobs — how they get wired into an autoscaler's
  scale-out decision.
  ([Red Hat Developer, Oct 2025](https://developers.redhat.com/articles/2025/10/02/autoscaling-vllm-openshift-ai))
- **IBM Newsroom — "IBM Announces Red Hat AI Inference and Red Hat OpenShift Virtualization
  Service on IBM Cloud"** (May 2026) — confirms vLLM plus the llm-d scheduler is now a
  productized, fully-managed enterprise offering running on bare-metal H200/MI300X.
  *What it shows:* this tuning skill is now a packaged commercial capability with a
  procurement line item — direct evidence of its market value. *(2026 snapshot — a very
  recent announcement; expect the product name/positioning to evolve.)*
  ([IBM Newsroom](https://newsroom.ibm.com/2026-05-12-ibm-announces-red-hat-ai-inference-and-red-hat-openShift-virtualization-service-on-ibm-cloud))

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

Note what you did *not* do to fix this: you did not raise `gpu-memory-utilization`. That
would shrink your OOM headroom to buy a little more KV pool — trading a (recoverable,
visible) throughput problem for a (catastrophic, crash-and-restart) one. The fix lives in
demand-side knobs (`max-num-seqs`, `max-model-len`), not supply-side ones, once utilization
is already at a sane operating point.

---

## Practice

Feeds the deliverable at [`../practice/cost-per-token/README.md`](../practice/cost-per-token/README.md).
On **rented GPUs** (2× A100/H100 for the 70B TP=2 path; or 1× L4/A10 with the 8B
substitute):

1. Deploy with `tensor-parallel-size=2` (or TP=1 for the 8B). Record the startup KV-cache
   size and "maximum concurrency" line.
2. **Oversubscribe**: fire many concurrent long-context requests (the `xargs -P` loop) so
   aggregate KV demand exceeds the pool.
3. Capture a **preemption event**: the `PreemptionMode.RECOMPUTE` log line **and** the
   `vllm:num_preemptions_total` delta, plus the TPOT spike from module-05 metrics. Note
   whether the pattern you triggered is a burst or a steady-state trend (§5) — you
   deliberately forced the latter here.
4. **Tune**: adjust `max-num-seqs` / `max-model-len` (and, if you saw startup or under-load
   OOM, `gpu-memory-utilization`) until preemptions stop under your target load. Record
   before/after throughput (tokens/sec) — this is a direct $/1M input.

**Acceptance (deliverable):** a **documented preemption event** (log line + counter delta +
TPOT impact) and a **tuned config** (the flag set that eliminates steady-state preemption
at your target concurrency) written into the cost-per-token workbook, with the tuned
throughput feeding the $/1M-tokens figure and a one-line note on the TP-vs-replica choice
you'd make for scale-out.

---

## Common pitfalls

- **Raising `gpu-memory-utilization` to fix a preemption problem.** One of the most common
  wrong instincts: preemption means the KV pool ran out, so "give it more memory" sounds
  right — but it steals the OOM headroom instead, trading a visible, recoverable throughput
  problem for a crash. Fix preemption with the demand-side knobs (`max-num-seqs`,
  `max-model-len`) first; only raise utilization if you have *confirmed* headroom to spare
  and OOM has never fired.
- **Treating tensor parallelism as a free latency win.** Teams add `--tensor-parallel-size
  2` "for speed" on a model that already fits comfortably on one GPU, paying an all-reduce
  tax on every layer for a request pattern that was never latency-bound in the first place.
  Apply the crossover rule: TP earns its keep when the model doesn't fit, or you're
  latency-bound on individual requests — otherwise replicas are strictly cheaper per token.
- **Not distinguishing burst preemption from steady-state preemption.** A single
  preemption event under a load spike is the system working correctly — the
  graceful-degradation valve. A *climbing rate* under sustained load is oversubscription.
  Conflating the two means either panicking over normal behavior or, worse, ignoring a real
  capacity problem because "preemption is supposed to happen sometimes."
- **Sizing `max-model-len` to the model card's advertised context window instead of your
  measured P99 request length.** A model that advertises 128K context doesn't mean your
  traffic uses anywhere near that. Setting `max-model-len` to the architectural max
  silently throttles concurrency — the "maximum concurrency" figure in the startup log
  collapses for capability nobody in your actual traffic is using.
- **Forgetting swap-space defaults exist.** `swap-space` defaults to 4 GiB/GPU and governs
  whether preempted sequences are recomputed or copied to CPU RAM. Some teams are surprised
  which mode they're actually getting in production — always confirm you're on V1's
  recompute default (or have deliberately opted into swap for a specific long-context
  reason) rather than assuming.

---

## Self-check

- **What's the symptom of setting `gpu-memory-utilization` too high, and too low?**
  **Answer:** Too high (≥~0.97) leaves no headroom for transient allocations, so you get
  `CUDA out of memory` **crashes** — at startup during CUDA graph capture, or under load when
  concurrent prefills spike working memory past the sliver left free. Too low (e.g. 0.6)
  shrinks the KV pool, so "maximum concurrency" is small, `max-num-seqs` never fills, and you
  queue/preempt at modest load with the GPU showing low utilization *and* low throughput —
  you're paying for fenced-off HBM. High fails loudly (outage); low fails quietly (2× your
  $/1M). Tune up to the OOM edge under realistic load, back off one step; 0.90 default,
  0.92–0.95 on a dedicated, well-characterized card.

- **When is swap preemption worse than recompute?**
  **Answer:** Swap copies the preempted sequence's KV blocks out to CPU RAM and back over
  PCIe — moving the *entire* KV twice. For **long contexts** that KV is gigabytes, and PCIe
  is slow, so the transfer can stall the pipeline harder and longer than simply discarding
  the blocks and re-running prefill (recompute) would — prefill is compute the GPU is good
  at, and it frees blocks instantly with no bus traffic. So swap is worse precisely when KV
  is large relative to prompt-recompute cost and interconnect bandwidth is the bottleneck —
  which is why V1 defaults to **recompute**. Swap only wins when prompts are very long
  (recompute expensive) *and* CPU RAM bandwidth to spare — an increasingly rare case.

- **Tensor parallelism vs running separate replicas — when each, and where's the
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

- **You see `vllm:num_preemptions_total` tick up by 3 during a 30-second traffic spike,
  then flatten. Do you change your config?**
  **Answer:** No, not on this evidence alone. An isolated bump during a burst is the
  graceful-degradation valve doing its job — a few sequences briefly evicted and resumed,
  not a capacity problem. The signal that demands a config change is a *sustained,
  climbing rate* under steady-state load, not a single event. Reacting to every burst by
  loosening `max-num-seqs` or `max-model-len` would leave you under-provisioned for normal
  traffic just to avoid a self-correcting transient.

- **A teammate proposes fixing steady-state preemption by bumping `gpu-memory-utilization`
  from 0.90 to 0.97. What's wrong with that fix?**
  **Answer:** It attacks the wrong side of the budget. Preemption under steady-state load
  means the *demand* (concurrent KV from `max-num-seqs`/`max-model-len`) exceeds the
  *supply* the pool was sized for at the current utilization — raising utilization grows
  the pool marginally but eats the headroom reserved for transient allocation spikes (CUDA
  graph capture, prefill bursts), trading a visible, recoverable throughput problem for
  crash risk under the next spike. The correct fix is demand-side: lower `max-num-seqs` or
  `max-model-len` to match sustainable concurrency, or add capacity.

---

## Connections & what's next

Builds directly on 07.3's block-manager mechanism — preemption is what happens when that
allocator's pool runs dry under real concurrency, and recompute/swap are the two ways it
recovers. Reuses module 05's `/metrics` (`num_preemptions_total`,
`time_per_output_token_seconds`) as the instrumentation for reading preemption events. The
tuned, non-preempting config this lesson lands on is the input to 07.5's batching-economics
sweep — running that sweep against an untuned (preempting, or unnecessarily TP'd) config
would produce a misleading CPM curve.

Next: **07.5 — Batching economics** is the module's cost pivot. It takes the throughput
numbers a correctly-tuned vLLM deployment (this lesson) can sustain and turns them into the
actual $/million-tokens curve — sweeping concurrency, finding the CPM-vs-latency knee under
an SLO, and building the centerpiece chart of the module's deliverable. Where this lesson
asked "what configuration keeps the GPU from crashing or degrading," 07.5 asks "at that
configuration, exactly how much does a token cost, and where's the point past which more
batch stops being worth it."

---

## References & further reading

**Primary sources**
1. **vLLM Optimization and Tuning** — https://docs.vllm.ai/en/stable/configuration/optimization/
   — the canonical current reference for `max-num-seqs`, `max-num-batched-tokens`, chunked
   prefill, and preemption behavior.
2. **vLLM — Conserving Memory** — https://docs.vllm.ai/en/latest/configuration/conserving_memory/
   — `gpu-memory-utilization`, `max-model-len`, quantization, and TP/PP as memory levers;
   the OOM-avoidance playbook.
3. **"Inside vLLM: Anatomy of a High-Throughput LLM Inference System"** —
   https://blog.vllm.ai/2025/09/05/anatomy-of-vllm.html — current (V1) production-deployment
   walkthrough tying the scheduler, preemption, and parallelism together; cross-ref with
   07.3.

**Real-world engineering blogs**
4. **Red Hat — "How vLLM accelerates AI inference: 3 enterprise use cases"** —
   https://www.redhat.com/en/topics/ai/how-vllm-accelerates-ai-inference-3-enterprise-use-cases
   — Roblox/LinkedIn production numbers, equally relevant here as evidence tuning pays off
   at scale. 2025 snapshot.
5. **Red Hat Developer — "Autoscaling vLLM with OpenShift AI"** —
   https://developers.redhat.com/articles/2025/10/02/autoscaling-vllm-openshift-ai — the
   day-two-ops operator's companion to this lesson's config knobs.
6. **IBM Newsroom — "IBM Announces Red Hat AI Inference and Red Hat OpenShift
   Virtualization Service on IBM Cloud"** —
   https://newsroom.ibm.com/2026-05-12-ibm-announces-red-hat-ai-inference-and-red-hat-openShift-virtualization-service-on-ibm-cloud
   — vLLM + llm-d as a productized managed offering on bare-metal H200/MI300X. May 2026
   snapshot.

**Deeper dives**
7. **Audrey Wang — "Understanding vLLM Scheduling: Token Budgets, Chunked Prefill, and
   Policies"** —
   https://audreywongkg.medium.com/understanding-vllm-scheduling-token-budgets-chunked-prefill-and-policies-2c879e3980e3
   — third-party deep dive on scheduler internals, useful for how `max_num_batched_tokens`
   and chunked prefill interact under the hood.
