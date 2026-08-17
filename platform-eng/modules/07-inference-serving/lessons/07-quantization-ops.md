---
lesson: "07.7"
title: "Quantization ops: the cost lever"
module: "07"
concept: "Quantization ops: the cost lever"
status: not-started
est_time: "6h"
prev: "06-alternative-servers-disaggregation.md"
next: "08-autoscaling-inference.md"
artifacts: []
sources: 14
---
# 07.7 · Quantization ops: the cost lever

> **Concept.** Quantization is the single largest lever on cost-per-million-tokens — but the saving is only honest once you have *measured* the accuracy delta, not asserted it.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

06 taught you to pick an **engine** and a **topology** for a workload shape — a structural decision about where compute happens. This lesson is a **numerical-format** decision layered on top of whatever you picked: given the same GPUs and the same engine, which precision do you actually ship? It sits right after engine selection and right before autoscaling (08), and the two are more connected than they look — a smaller quantized checkpoint loads faster from cold (09) and frees the VRAM that autoscaling's batch-size headroom depends on. Quantization is also the second of this module's two explicit "cost decisions you defend at the checkpoint," after 05's batching curve.

Module 03.5 already gave you the **formats**: the encoding rule, E4M3's 448 maximum and why it is not 240, BF16-vs-FP16's opposite bets on range and precision, the per-SM FLOP/clock table that makes FP8 exactly 2× BF16 in silicon, and the hardware gate (FP8 needs Hopper+, FP4 needs Blackwell). **This lesson does not re-teach any of that.** What it adds is everything between "the format exists" and "the endpoint is serving it": the *mapping* from float to grid, where the scales come from, which methods compute them and how, what granularity buys, and how all of that turns into a defensible number on a cost report.

Everything version-specific below is checked against source: **vLLM v0.27.1** (`docs/features/quantization/`, `vllm/config/cache.py`, `vllm/model_executor/layers/quantization/__init__.py`), **TensorRT-LLM 1.3.0-rc** (`docs/source/features/quantization.md`), and the reference implementations of **GPTQ** (`IST-DASLab/gptq`), **AWQ** (`mit-han-lab/llm-awq`) and **SmoothQuant** (`mit-han-lab/smoothquant`).

## Why this matters

The stakes are money and trust, and they pull in opposite directions.

On the money side: on an H100, moving a Llama-3.1-8B endpoint from BF16 to FP8 roughly **halves the weight footprint and the weight-streaming time per decode step**, and — with an FP8 KV cache alongside — roughly **doubles the concurrent-sequence ceiling**. §12 derives what that does to cost-per-million-tokens, and the answer is a 40–50% cut at a fixed GPU price. That is not a micro-optimisation; on a fleet it is the difference between a serving line item that survives a budget review and one that does not.

On the trust side: ship it blindly and you can quietly degrade a code-generation product by several points of pass@1, because your calibration set was generic prose and your evaluation was generic prose too. Worse — and this is the failure mode almost nobody catches — **you can "measure" the accuracy delta on 500 samples, see +0.2 points, write "no measurable loss" in the report, and be reporting pure noise.** §14 does the binomial arithmetic: at `--limit 500`, the 95% confidence interval on a difference between two runs is about **±6 percentage points**. A 0.2-point delta is not evidence of anything.

The differentiator for a senior platform engineer is not "we turned on FP8." It is: *"we cut CPM 43%; the accuracy delta was within noise at the sample size we could afford, so we additionally ran a 3,000-sample code slice where the noise floor is ±2.4 points, and it moved 0.3 — here is the interval."* One of those is a slide. The other is an engineering decision that survives a follow-up question.

## What's new here (calibration)

You have the formats from 03.5 and the throughput/cost framing from 05. Five things are genuinely new:

1. **The affine map itself.** A format is not a quantization scheme. `int8` is eight bits; *what those eight bits mean* is set by a scale and a zero-point that someone had to compute. §1 is that map, and every method in this lesson is a different answer to "how do you choose the scale."
2. **Granularity as the dominant accuracy lever.** Per-tensor vs per-channel vs per-token vs per-group is a bigger accuracy effect than which *algorithm* you ran. §3 quantifies it.
3. **Quantization as an OPS decision, not a hardware fact.** 03.5 told you FP8 tensor cores exist. Here you choose FP8 / INT8 / W4A16 *as a fleet policy*, driven by GPU generation, VRAM budget, and — the part people miss — **batch size**, because §12 shows the winner literally changes at a computable crossover.
4. **The calibration set as a first-class artifact.** Static activation quantization derives scales from sample data. That sample data is now a thing you own, version, and can get wrong — and llm-compressor's own best-practice guidance says so.
5. **Statistical power as a gate.** "Measure the delta" is not enough if the measurement cannot resolve the delta. §14 gives you the sample size you actually need.

## Core concepts

### 1. What quantization actually is — the affine map

Strip away the vocabulary. Quantization is a function that takes a real number `x` and returns one of a small set of representable values, plus the inverse function that gets you back.

For **integer** formats the map is affine:

```
  QUANTIZE      q = clamp( round( x / s ) + z ,  q_min , q_max )
  DEQUANTIZE    x̂ = s · ( q − z )

    s  = SCALE       a positive real. How much of x one integer step covers.
    z  = ZERO-POINT  an integer. Which q maps to x = 0.
    q_min, q_max     the grid: int8 symmetric → [−127, 127]; int4 → [−7, 7]
                     (asymmetric int8 uses [−128, 127] or [0, 255])
```

For **float** formats (FP8 E4M3) the map is just a scale — no zero-point, because the format already represents zero and signs natively:

```
  QUANTIZE      q = to_fp8_e4m3( x / s )        clamped at ±448
  DEQUANTIZE    x̂ = s · q
```

Two kinds of error come out of this, and they behave completely differently:

- **Rounding error** — `|x̂ − x| ≤ s/2` for integers. Bounded, roughly uniform, averages out over a dot product. This is the *benign* error.
- **Clipping error** — if `|x| > s · q_max`, the value saturates and the error is **unbounded**. This is the error that destroys models, and it is why a single outlier in a tensor is so expensive: to avoid clipping it, you must raise `s`, which coarsens the grid for every other value in the same scale group.

That tension is the whole subject. **Every quantization method in this lesson is a different strategy for choosing `s` so that clipping error and rounding error trade off well.** Round-to-nearest picks `s` from the max. GPTQ keeps the grid and *compensates* the error elsewhere. AWQ rescales the tensor so the important values land in the good part of the grid. SmoothQuant moves the difficulty from activations into weights. Once you see them as four answers to one question, the taxonomy stops being a list to memorise.

A worked instance, so the numbers are concrete. Take an int8 symmetric per-tensor map over a weight tensor whose absolute max is 0.42:

```
  s = max|W| / q_max = 0.42 / 127 = 0.003307
  grid spacing = 0.003307 → any weight is representable to ±0.00165

  A typical weight w = 0.0113:
      q = round(0.0113 / 0.003307) = round(3.417) = 3
      x̂ = 3 × 0.003307 = 0.009921        error = −0.00138   (−12.2% relative)

  Now introduce ONE outlier at 4.2 (10× the previous max):
  s = 4.2 / 127 = 0.033071                    ← grid is 10× coarser
      q = round(0.0113 / 0.033071) = round(0.342) = 0
      x̂ = 0                              error = −0.0113    (−100%: annihilated)
```

**One outlier, and a typical value goes from 12% error to being quantized to zero.** That single calculation is the mechanism behind every technique below. Hold onto it.

### 2. The taxonomy: `WxAy`, and where the scales come from

Notation you will see everywhere: **`WxAy`** = `x`-bit weights, `y`-bit activations. `W8A8` quantizes both. `W4A16` quantizes weights to 4 bits and leaves activations at 16 (this is what "weight-only" means).

The distinction matters because **the two paths use different hardware**:

```
  ══════════════════════════════════════════════════════════════════════════
   PATH A — WEIGHT-ONLY  (W4A16: AWQ, GPTQ)     "make the model SMALLER"
  ══════════════════════════════════════════════════════════════════════════

   OFFLINE (once, on a calibration set)          SERVING (every forward pass)
   ─────────────────────────────────             ────────────────────────────
   W (bf16)                                       x (bf16, UNTOUCHED)
     │                                                 │
     │ calibration data ──▶ [ per-group scale search ] │
     │                       GPTQ: Hessian from        │
     │                             calib activations   │
     │                       AWQ:  activation-aware    │
     │                             per-channel rescale │
     ▼                                                 │
   [ W_int4 + scales/zeros per 128-weight group ]      │
   ~4.25 bits/weight on disk and in HBM                │
     │                                                 │
     │  ── at kernel time ──                           │
     ▼                                                 ▼
   DEQUANT to bf16 in registers ──────────▶  [ bf16 × bf16 GEMM ]
   (Marlin/Machete kernels fuse this                    │
    into the GEMM's inner loop)                         ▼
                                                     y (bf16)

   ⇒ HBM traffic for weights:  ÷ 4       ← THE WIN
   ⇒ tensor-core rate:         bf16      ← NO compute win. The matmul is
                                            still a 16-bit matmul.
   ⇒ extra work:               dequant in the inner loop (a real cost at
                                            large batch)

  ══════════════════════════════════════════════════════════════════════════
   PATH B — WEIGHT + ACTIVATION  (W8A8: FP8, INT8)   "make the model FASTER"
  ══════════════════════════════════════════════════════════════════════════

   OFFLINE                                        SERVING
   ───────                                        ───────
   W (bf16)                                       x (bf16)
     │                                                 │
     │ RTN or GPTQ                                     │  ┌── STATIC:  s_x from
     ▼                                                 │  │   calibration, FIXED
   [ W_fp8/int8 + per-channel scale s_w ]              │  │   ▸ needs a calib set
     │                                                 │  │   ▸ zero runtime cost
     │                                                 │  │   ▸ CAN BE WRONG
     │                                                 ▼  │
     │                                        [ quantize x ]◀┘
     │                                          s_x   │      DYNAMIC: s_x from
     │                                                │      THIS tensor's amax,
     │                                                │      computed per forward
     │                                                │      ▸ no calib set
     │                                                │      ▸ small runtime cost
     ▼                                                ▼      ▸ cannot be stale
   ┌──────────────────────────────────────────────────────┐
   │  FP8 / INT8 TENSOR-CORE GEMM                         │
   │  narrow in, FP32 accumulate  (03.5 §6)               │
   │  acc  ──── × (s_w · s_x) ────▶  y (bf16)             │
   │           the DE-SCALE, folded into the epilogue     │
   └──────────────────────────────────────────────────────┘

   ⇒ HBM traffic for weights:  ÷ 2
   ⇒ tensor-core rate:         2× (Hopper FP8: 8192 vs 4096 FLOP/SM/clk)  ← WIN
   ⇒ risk:                     activations are the hard tensor (§4)
```

**Read the two "⇒ tensor-core rate" lines against each other, because that one difference generates the entire deployment rule.** W4A16 buys bytes and nothing else. W8A8 buys bytes *and* FLOPs. Which one wins therefore depends on whether your operating point is bytes-bound or FLOPs-bound — i.e. on your batch size. §12 computes the crossover.

### 3. Granularity — the biggest lever, and the cheapest

Go back to §1's outlier calculation. The damage came from one scale being shared by values with wildly different magnitudes. So the obvious fix is: **share the scale over a smaller group.**

| Granularity | One scale covers | Scale storage overhead | Typical use |
|---|---|---|---|
| **Per-tensor** | the entire weight matrix (or activation tensor) | ~0 | Simplest; vLLM's online `fp8` weight default; most vulnerable to outliers |
| **Per-channel** (weights) | one output channel — a row of `W` | `out_features` scales, ~0.01% | Standard for W8A8 weights; costs nothing, recovers most of the loss |
| **Per-token** (activations) | one token's activation vector | `batch × seq` scales, computed at runtime | The dynamic-activation default; naturally adapts to per-token magnitude swings |
| **Per-group** (weights) | 128 contiguous weights along the input dim | 4 → 4.25 effective bits at `g=128` | The W4A16 workhorse (`q_group_size: 128`) |
| **Per-block** | a 2-D tile — 128×128 weights, 1×128 activations | ~0.2% | FP8 block scaling; DeepSeek-V3's recipe (03.5 §8) |
| **Micro-block** | 32 elements, shared E8M0 exponent | 4 → 4.25 bits (MXFP4) | Blackwell MX formats; mandatory below 8 bits |

Compute the W4A16 overhead once, because you will need it for capacity planning:

```
  int4 weights, group size 128, one fp16 scale + one fp16 zero-point per group:

     bits per weight = 4 + (16 + 16) / 128 = 4 + 0.25 = 4.25 bits

  Llama-3.1-8B (8.03e9 params):  8.03e9 × 4.25 / 8 = 4.27 GB
  Llama-3.1-70B (70.6e9):        70.6e9 × 4.25 / 8 = 37.5 GB
```

So "4-bit" is really 4.25-bit, and a "70B on one 80 GB H100" claim is 37.5 GB of weights with ~38 GB left for KV cache and workspace — genuinely comfortable, unlike the FP8 version's ~70 GB weights and ~6 GB of headroom.

**The practical rule:** if a quantization result looks worse than published, check the granularity before you change the algorithm. Per-tensor → per-channel on weights is close to free and recovers most of a naive scheme's loss. It is the first thing to try and the last thing people try.

### 4. Why activations are the hard tensor — outlier channels

03.5 §4 told you weights are easy (tight, zero-centred, two or three orders of magnitude end to end) and activations are hard (asymmetric, long-tailed). The mechanism deserves one more "why," because it determines which methods exist.

Transformer activations contain **outlier channels**: specific feature dimensions — the *same* dimensions across essentially every token and every input — whose magnitudes run one to two orders of magnitude above the rest. They are not random noise; they are a systematic, persistent property of the trained network, and they appear in a small number of channels (typically well under 1%).

Now combine that with granularity. Activation quantization is usually **per-token** — one scale per token vector — because that is what a runtime can compute cheaply. But a token vector *contains* the outlier channels. So one scale must cover both the outlier at 60.0 and the ordinary features at 0.3, and §1's arithmetic applies directly: the ordinary features get crushed.

```
  ONE TOKEN'S ACTIVATION VECTOR, per-token int8 scale
  ═══════════════════════════════════════════════════

  channel:  0    1    2  … 1387 …  2044  2045  2046  2047
  |x|:     0.31 0.28 0.42 … 61.4 …  0.35  0.29  0.33  0.40
                              ▲
                              └── OUTLIER CHANNEL. Present on nearly every
                                  token, in the SAME channel index.

  s = 61.4 / 127 = 0.4835
      ordinary value 0.31  →  round(0.31/0.4835) = round(0.641) = 1
                           →  x̂ = 0.4835        error +56%
      ordinary value 0.28  →  round(0.579) = 1  →  0.4835   error +73%
      ordinary value 0.42  →  round(0.869) = 1  →  0.4835   error +15%

  ⇒ ~2,047 useful channels are quantized to 3 distinct levels because of one.
```

**That is why W8A8 is hard and W4A16 is comparatively easy.** Weight-only quantization never touches this tensor at all. Every W8A8 method — SmoothQuant, static-scale INT8, FP8 — is fundamentally a strategy for coping with this picture.

FP8 E4M3 copes with it *structurally*, and this is the cleanest argument for why FP8 is the default on Hopper+: it is a **floating-point** grid, so its resolution is *relative*, not absolute. At ε = 0.125 it holds 0.31 and 61.4 to the same ~12.5% relative accuracy, without either one setting a scale that ruins the other. INT8's grid is absolute — uniform steps of size `s` across the whole range — so it has no such defence and needs SmoothQuant to build one. **Same bit width, completely different failure behaviour, entirely because one is floating point and one is not.**

### 5. The methods — one table, real mechanisms

| Method | Bits (W/A) | What it quantizes | Calibration | Mechanism in one line | Published accuracy delta | Hardware |
|---|---|---|---|---|---|---|
| **RTN** (round-to-nearest) | any | whichever you point it at | None for weights | `s = max\|W\| / q_max`, round | LLaMA-7B Wiki2 PPL 5.68 → **6.29** at 4-bit; → **25.54** at 3-bit *(GPTQ repo)* | anywhere |
| **FP8 dynamic** (`W8A8-FP`, E4M3) | 8/8 | weights (per-channel or per-tensor) + activations (per-token, at runtime) | **None** | RTN on weights; activation scale from *this* forward pass's amax | gsm8k 5-shot **0.768 ± 0.027** on Llama-3-8B-Instruct-FP8-Dynamic, 250 samples *(vLLM docs)* | **Ada, Hopper, Blackwell**. Turing/Ampere fall back to W8A16 via FP8 Marlin |
| **FP8 static / block** | 8/8 | same, but activation scales fixed offline (or per-128-block) | small set (or none for block) | Fixed `s_x`; block scaling uses 128×128 weight / 1×128 activation tiles | — (recipe-dependent) | Hopper + Blackwell (block); sm120 lacks block scaling |
| **SmoothQuant** (enables INT8 `W8A8`) | 8/8 | activations, via a weight↔activation rebalance | **512 samples** (Pile val, in the reference impl) | Divide activations by per-channel `s_j`, multiply weights by it — a mathematically equivalent transform that moves difficulty into the (easy) weights | Llama-2-7B Wiki2 PPL 5.474 → **5.515** (α=0.85); Llama-3-8B 6.138 → **6.258**; Llama-3-70B 2.857 → **2.982** *(SmoothQuant repo)* | Turing → Hopper. **Not on Blackwell sm≥100** — INT8 is unsupported there |
| **GPTQ** | 4/16 (also 3, 8) | weights only, per-group | **128 samples** default (`--nsamples 128`) | Layer-wise second-order: build `H = 2/n · Σ xxᵀ` from calibration activations, quantize column by column, **propagate each column's error into the not-yet-quantized columns** via `H⁻¹` | LLaMA-7B Wiki2 5.68 → **6.09** at 4-bit (vs 6.29 RTN); 3-bit 25.54 → **8.07** *(GPTQ repo)* | Turing → Blackwell |
| **AWQ** | 4/16 | weights only, per-group | **~128–512 samples** (Pile val, 512-token blocks) | Find the ~1% of weight channels seeing the largest activations; rescale by `s = x_max^α` with **α grid-searched over 20 values** minimising block-output MSE | MLSys 2024 Best Paper; competitive with or better than GPTQ on instruction/chat tasks | Turing → Blackwell (not Volta) |
| **W4A16 in aggregate** | 4/16 | — | yes | — | **~99.36% recovery** vs BF16 across the Llama-3.1 family *(Neural Magic / IST Austria, >500k evals — see References for a provenance caveat)* | — |
| **W8A8 in aggregate** (FP8 or INT8) | 8/8 | — | varies | — | **~99.75% recovery** *(same study, same caveat)* | — |
| **FP8 / NVFP4 KV cache** | — | the KV cache only | optional | Independent lever: halves (or quarters) KV bytes per token | recipe-dependent | FP8 KV works even on **Ampere** — it is storage, not a tensor-core op |

Two entries deserve unpacking, because they are the ones that get misquoted.

**RTN's 3-bit row is the argument for calibration existing at all.** LLaMA-7B goes from 5.68 perplexity to **25.54** under naive round-to-nearest at 3 bits — the model is destroyed. GPTQ brings the same 3-bit configuration back to 8.07. At 4 bits the gap is much smaller (6.29 vs 6.09), which tells you something operationally useful: **the more aggressive the bit width, the more the algorithm matters.** At 8 bits, RTN is fine and that is exactly why FP8 needs no calibration.

**SmoothQuant's numbers are perplexity, not benchmark accuracy, and the deltas are tiny but not zero.** Llama-3-8B goes 6.138 → 6.258, about **+2.0% perplexity**. Llama-3-70B goes 2.857 → 2.982, about **+4.4%** — note the *larger* model degrades *more* here, which cuts against the folk belief that bigger models are always more quantization-robust. Also note the α column in the source table varies by model (0.6 for Falcon-7B, 0.85 for the Llamas, 0.9 for Llama-2-70B): **α is a per-model hyperparameter someone tuned**, not a constant, and shipping the default 0.5 on a model whose published α is 0.85 is a real way to lose accuracy for no reason.

### 6. GPTQ, mechanically

The problem GPTQ solves: round-to-nearest quantizes each weight independently and *ignores that the weights are used together*. Two weights whose errors point the same way compound; two whose errors cancel do not matter. RTN cannot know the difference because it never looks at the data.

GPTQ looks at the data — once, offline, through the second-order structure of the layer's output error.

```
  For each linear layer, independently:

  1. COLLECT.  Run the calibration set (default 128 samples) through the model.
               For this layer, accumulate the input second-moment matrix:

                     H  =  (2/n) · Σ  x xᵀ           [columns × columns]

               H is the Hessian of the layer's squared output error with
               respect to its weights. It says which INPUT DIMENSIONS the
               layer's output is most sensitive to.

  2. DAMPEN.   H[i,i] += percdamp · mean(diag(H))      percdamp default 0.01
               Without this, H is often singular (dead input channels) and
               the Cholesky below fails.

  3. INVERT.   H⁻¹ via Cholesky, then take the upper Cholesky factor of H⁻¹.
               This gives the per-column error-propagation coefficients.

  4. SWEEP columns left to right, in blocks of 128 (`blocksize=128`):

        for each column i:
            w = W[:, i]                       the un-quantized column
            d = Hinv[i, i]
            q = quantize(w)                   round onto the int4 grid
            err = (w − q) / d                 the error, scaled by sensitivity

            W[:, i+1:] −= err ⊗ Hinv[i, i+1:]     ◀── THE WHOLE IDEA
            ▲
            └── push this column's rounding error into the columns NOT YET
                quantized, so they can absorb it. Later columns are chosen
                to CANCEL earlier columns' error.

  5. (optional) --act-order: quantize columns in DECREASING order of
     diag(H) — most-sensitive first, so they are rounded while the most
     error-absorption budget remains. On LLaMA-7B this moved Wiki2 PPL from
     7.15 to 6.09; on OPT-66B 3-bit, from 14.16 to 9.95.
```

Two operational consequences:

- **GPTQ is layer-local.** It never touches the loss, never backpropagates, and quantizes one layer at a time on a single GPU. That is why it runs in minutes-to-hours rather than requiring a training cluster — and why it cannot fix an error that only shows up as a whole-model behaviour.
- **`--act-order` was a large win and is now standard.** If you are comparing your own GPTQ result to a published one and it looks bad, check whether the published run used `--act-order` and `--true-sequential`. This is a common apples-to-oranges failure.

### 7. AWQ, mechanically

AWQ starts from a different observation: **not all weights matter equally, and which ones matter is determined by the activations, not by the weights.** A weight channel that multiplies a large activation contributes more to the output than one that multiplies a small activation, regardless of the weight's own magnitude.

So AWQ protects the salient channels — roughly the top 1% by activation magnitude — not by keeping them in higher precision (which would break the kernel's uniformity) but by **scaling them up before quantization and scaling the input down correspondingly**, which is mathematically a no-op but moves those channels into a better part of the grid.

```
  For each transformer block:

  1. Run calibration data (Pile validation, 512-token blocks) and record
     the per-input-channel activation magnitude:   x_max[j] = mean |x[:, j]|

  2. GRID-SEARCH the exponent α over 20 values, α ∈ {0/20, 1/20, …, 19/20}:

        for α in that grid:
            s = x_max ^ α                          per-input-channel scale
            s = s / sqrt(max(s) · min(s))           normalise so mean-ish = 1

            W' = quantize( W · s ) / s              apply, quantize, undo
            loss = mean( (block(x) − block'(x))² )  ◀── END-TO-END BLOCK MSE,
                                                        not per-weight error
        keep the α with the lowest loss

  3. Fold s into the preceding operation (the layernorm's weight, or the
     previous linear's output), so serving costs nothing extra.

  Read what α means:
      α = 0    →  s = 1        no protection at all (plain RTN)
      α = 1    →  s = x_max    full activation-proportional protection
      0 < α< 1 →  partial. The search finds the balance for THIS block.
```

Three things worth noticing:

1. **The objective is block-output MSE, not weight error.** AWQ explicitly does not try to minimise `|W − Ŵ|`; it minimises how much the *block's output* moves. That is why it can beat a method that has strictly better weight reconstruction.
2. **It is a search, not a formula.** Twenty forward passes per block per candidate. Which means the calibration data appears in the objective *twenty times per block* — AWQ is, if anything, more calibration-sensitive than GPTQ, not less.
3. **The scale folds away.** After step 3 there is no runtime cost. AWQ produces a normal W4A16 checkpoint; the cleverness is entirely offline.

`AutoAWQ`, the library most tutorials point at, is **deprecated** — vLLM's own AWQ documentation carries that warning and directs you to `llm-compressor`'s AWQ examples instead. If a runbook in your repo says `pip install autoawq`, it is out of date.

### 8. SmoothQuant, mechanically — moving difficulty, not removing it

§4 showed activations are hard and weights are easy. SmoothQuant's insight is that a linear layer lets you *move* difficulty between them for free, because the following transformation is exactly equivalent:

```
      Y = X · W                                     original
      Y = (X · diag(s)⁻¹) · (diag(s) · W)           mathematically identical

    for any positive s. Divide the activations by s, multiply the weights by s.
```

Choose `s` per input channel to flatten the activation outliers, and put the resulting bulge into the weights, which have the dynamic-range headroom to absorb it. The reference implementation's formula:

```
        s_j  =  max|X_j| ^ α   /   max|W_j| ^ (1 − α)              α ∈ [0, 1]

    α = 0   → all difficulty pushed into the WEIGHTS
    α = 1   → all difficulty left in the ACTIVATIONS
    α = 0.5 → the repo's default; the balanced split
    α = 0.8 → what llm-compressor's INT8 recipe actually uses
              (SmoothQuantModifier(smoothing_strength=0.8))
```

And the free part: `diag(s)⁻¹` is folded into the **preceding LayerNorm's weight and bias** (or RMSNorm weight), so it costs zero at inference.

```
  BEFORE SMOOTHING                       AFTER SMOOTHING (α = 0.8)
  ────────────────                       ─────────────────────────

  activation |X| per channel             activation |X·s⁻¹| per channel
    60 ┤        █                          6 ┤
       │        █                            │  ▄▄▄█▄▄▄▄▄▄▄▄▄▄▄▄
       │        █   ◀── one channel          │  ████████████████
     0 ┼▁▁▁▁▁▁▁▁█▁▁▁▁▁▁▁▁▁                 0 ┼────────────────────
       ⇒ int8 grid ruined for everyone       ⇒ int8 grid usable

  weight |W| per channel                 weight |W·s| per channel
   0.4 ┤ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄               0.9 ┤ ▄▄▄▄▄▄▄█▄▄▄▄▄▄▄▄
       │ ████████████████                   │ ████████████████
     0 ┼─────────────────                 0 ┼─────────────────
       ⇒ was easy                           ⇒ still easy — weights had
                                               the range to spare
```

Two consequences for you as an operator:

- **α is per-model.** SmoothQuant's own published table uses 0.6 for Falcon-7B, 0.7 for Falcon-40B, 0.8 for Mistral/Mixtral, 0.85 for the Llamas, 0.9 for Llama-2-70B. llm-compressor's default recipe uses 0.8. Using 0.5 because it is the function's default is leaving accuracy on the table for free.
- **SmoothQuant is a *pre*-processing step, not a quantizer.** llm-compressor's INT8 W8A8 recipe is literally a two-element list: `SmoothQuantModifier(smoothing_strength=0.8)` followed by `GPTQModifier(scheme="W8A8")`. Smooth first, then quantize. Naming them as separate stages is the difference between understanding the pipeline and reciting it.

### 9. Static vs dynamic activation scales — the calibration dependency, precisely

This is the axis that determines whether you own a calibration artifact at all.

| | **Static** | **Dynamic** |
|---|---|---|
| Where `s_x` comes from | A calibration pass, offline. Baked into the checkpoint. | `amax` of the actual tensor, computed inside the forward pass. |
| Calibration set needed | **Yes** | **No** |
| Runtime cost | Zero | A reduction over the activation tensor per quantized op |
| Failure mode | **Silent.** Production traffic whose activation distribution differs from calibration gets clipped or coarsely rounded. Nothing errors. | None of that class. Worst case is a slightly slower kernel. |
| vLLM invocation | pre-quantized checkpoint from llm-compressor | `--quantization fp8` (online, at load) |

vLLM's own documentation on online dynamic quantization states the mechanism plainly: **all Linear modules except `lm_head` have weights quantized to FP8-E4M3 with a per-tensor scale, and activation min/max are computed during each forward pass to give a dynamic per-tensor scale.** It also notes the honest caveat: *"latency improvements are limited in this mode."* Per-tensor dynamic scaling costs a full-tensor reduction and does not exploit per-token structure, so it is the safest configuration, not the fastest one. The performant middle ground is llm-compressor's `FP8_DYNAMIC` scheme: **static per-channel on weights, dynamic per-token on activations** — no calibration data required for either, because weights use RTN and activations are dynamic.

**The decision rule, stated once:**

> Use **dynamic** activation scales unless you have measured that static is meaningfully faster on your workload *and* you have a calibration set drawn from your own production traffic. Dynamic removes an entire class of silent failure for a small, measurable cost.

### 10. The KV cache is an independent lever — and its default is a trap

Weight precision and KV-cache precision are separate knobs. You can serve BF16 weights with an FP8 KV cache, or the reverse. In vLLM, `--kv-cache-dtype` accepts `auto`, `fp8`, `fp8_e4m3`, `fp8_e5m2`, `float16`, `bfloat16` — and it works on hardware that has **no FP8 compute at all**, because storing and converting FP8 is not a tensor-core operation. TensorRT-LLM's hardware matrix confirms the same asymmetry: Ampere gets FP8 KV cache, not FP8 weights.

Why it matters more than people expect: at large batch and long context, **KV traffic dominates weight traffic**. For Llama-3.1-8B at 2k context, 128 concurrent sequences:

```
  weights, bf16              = 16.1 GB   read once per step
  KV traffic per step        = 128 seqs × 2048 tok × 131,072 B/tok
                             = 34.4 GB   read once per step

  ⇒ 68% of the step's HBM traffic is KV, not weights.
  ⇒ Quantizing weights to FP8 cuts 16.1 → 8.0 GB: a 16% cut of the total.
  ⇒ Quantizing the KV cache too cuts 34.4 → 17.2 GB: another 34%.
```

**That is the arithmetic behind "FP8 gives 1.5–1.8×, not 2×."** Halving the weights only halves the smaller term at production batch sizes. This is also why vLLM's own FP8 documentation quotes *"a 2× reduction in model memory requirements and up to a 1.6× improvement in throughput"* — the memory number is exact and the throughput number is not, for exactly this reason.

**Now the trap.** vLLM's quantized-KV-cache documentation states that with no calibration, **all quantization scales are set to 1.0**. Read what that means: with `s = 1.0`, an E4M3 KV cache saturates at ±448 and its smallest normal is 1.56e-2, so any key or value above 448 clips outright and anything below ~0.016 falls into subnormals or zero. For most models attention K/V values sit comfortably inside that window, which is why `--kv-cache-dtype fp8` usually "just works" — but *"usually works because the default scale happens to suit the distribution"* is not the same as *"is calibrated,"* and it is not a claim you should make in a design review. vLLM's docs recommend calibrating the scales with llm-compressor for maximum accuracy, and per-attention-head scales are available (`q_scale = [num_heads]`, `k/v_scale = [num_kv_heads]`) on the Flash Attention backend. Also note: with the Flash Attention 3 backend and an FP8 KV cache, **queries are quantized to FP8 too** and the attention operation runs in the quantized domain — a bigger behavioural change than "the cache is smaller."

### 11. The hardware gate — which options exist on the GPUs you actually rent

Reproduced from vLLM's and TensorRT-LLM's own support matrices. This decides the *policy*, because a fleet with mixed generations cannot ship a single checkpoint that is fast everywhere.

| | Turing (7.5) | Ampere (8.0/8.6) | Ada (8.9) | Hopper (9.0) | Blackwell (10.0+) |
|---|---|---|---|---|---|
| **FP8 W8A8** | ✘ (runs as W8A16 via FP8 Marlin) | ✘ (same fallback) | ✔ | ✔ | ✔ |
| **FP8 block scaling** | ✘ | ✘ | ✘ | ✔ | ✔ (sm100/103; **not** sm120) |
| **INT8 W8A8** | ✔ | ✔ | ✔ | ✔ | **✘ — unsupported ≥ sm100** |
| **W4A16 AWQ** | ✔ | ✔ | ✔ | ✔ | ✔ |
| **W4A16 GPTQ** | ✔ | ✔ | ✔ | ✔ | ✔ |
| **NVFP4 / MXFP4** | ✘ | ✘ | ✘ | ✘ | ✔ |
| **FP8 KV cache** | ✔ | **✔** | ✔ | ✔ | ✔ |

Three non-obvious rows:

- **FP8 on Ampere does not error — it silently becomes W8A16.** vLLM runs FP8 checkpoints on compute capability ≥ 7.5 as weight-only via FP8 Marlin kernels. So "we deployed the FP8 checkpoint to the A100 pool and throughput didn't move" is the expected outcome, not a bug, and it is exactly the symptom 03.5 §Pitfalls warned about. The memory saving is real; the compute win is not there because the hardware path is not there.
- **INT8 is *removed* on Blackwell.** vLLM's INT8 documentation carries an explicit warning: INT8 is not supported on compute capability ≥ 10.0. If your cross-platform fallback policy is "INT8 everywhere," it breaks on your newest hardware, in the opposite direction from every other compatibility problem you have dealt with.
- **FP8 KV cache spans the whole table**, including Ampere. It is the one lever that is uniform across a heterogeneous fleet.

**The fleet-policy consequence:** there is no single checkpoint that is optimal everywhere. A realistic policy for a mixed fleet is *FP8 W8A8 on Hopper/Ada/Blackwell, W4A16 (AWQ or GPTQ) as the portable fallback for Ampere and older, FP8 KV cache everywhere.* W4A16 rather than INT8 as the fallback, precisely because INT8 dies on Blackwell while W4A16 spans Turing through Blackwell.

### 12. Where the win comes from — and the batch-size crossover, derived

This is the section that turns the taxonomy into a decision. Model the decode step with the roofline (module 03 lesson 2, module 05 §3):

```
  step_time  ≈  max( memory_time , compute_time )

  memory_time  = ( W_bytes + B · ctx · kv_bytes_per_token ) / HBM_BW
  compute_time = ( 2 · P · B ) / FLOPS_effective

  throughput   = B / step_time            [tokens/s across the batch]
```

Instantiate for **Llama-3.1-8B** (`P = 8.03e9`), 2k average context, on one **H100 SXM** (HBM 3.35 TB/s; dense BF16 989.5 TFLOP/s, dense FP8 1,979 TFLOP/s):

```
  BYTES PER FORMAT
    W_bf16   = 16.1 GB      W_fp8 = 8.03 GB      W_int4(g128) = 4.27 GB
    KV bf16  = 131,072 B/token          KV fp8 = 65,536 B/token
    KV per sequence per step at 2k ctx: bf16 268 MB · fp8 134 MB

  RIDGE POINT — the batch at which compute_time overtakes memory_time
  (weights only, to isolate the effect):

    BF16 :  2·8.03e9·B / 989.5e12  =  16.1e9 / 3.35e12   →  B ≈ 296
    FP8  :  2·8.03e9·B / 1979e12   =  8.03e9 / 3.35e12   →  B ≈ 295
    W4A16:  2·8.03e9·B / 989.5e12  =  4.27e9 / 3.35e12   →  B ≈  79
             ▲ compute is at BF16 rate — the matmul is still 16-bit
```

**Stop on that last line, because it is the whole deployment rule.** FP8 halves the bytes *and* doubles the FLOP ceiling, so its ridge point does not move: it is the same shape of workload, running twice as fast. W4A16 quarters the bytes but leaves the FLOP ceiling alone, so its ridge point falls by ~3.8×. Past batch ≈79, a W4A16 deployment is compute-bound at BF16 rate with dequantization overhead on top, and adding batch buys nothing.

Now the full model with KV traffic, at 2k context, effective W4A16 GEMM rate taken at 70% of BF16 peak (dequant in the inner loop is not free):

| Batch | BF16 + bf16 KV | FP8 + fp8 KV | W4A16 + bf16 KV | Winner |
|---|---|---|---|---|
| 1 | 205 tok/s | 410 tok/s | **738 tok/s** | W4A16 (3.6× BF16) |
| 8 | 1,469 | 2,943 | **4,176** | W4A16 |
| 16 | 2,505 | 5,266 | **6,257** | W4A16 |
| 32 | 4,342 | **8,698** | 8,336 | ≈ crossover |
| 64 | 6,258 | **12,468** | 10,143 | FP8 |
| 128 | 8,499 | **17,010** | 11,102 | FP8 |
| 256 | 10,111 | **20,237** | 11,754 | FP8 (1.72× W4A16) |

```
   tok/s
     ▲
20k  ┤                                              ╭──────── FP8 (W8A8)
     │                                        ╭─────╯          bytes ÷2
     │                                  ╭─────╯                FLOPs ×2
15k  ┤                            ╭─────╯
     │                      ╭─────╯
     │                ╭─────╯                    ╭──────────── W4A16
10k  ┤          ╭─────╯               ╭──────────╯   flattens at its
     │      ╭───╯          ╭──────────╯              own ridge (B≈79)
     │   ╭──╯      ╭───────╯                  ╭────────────── BF16
 5k  ┤ ╭─╯   ╭─────╯                ╭─────────╯
     │╭╯╭────╯          ╭───────────╯
     ├─────────────────────────────────────────────────────────▶ batch
     1    8   16   32   64        128           256
              ▲
              └── CROSSOVER ≈ B 30. Below it W4A16 wins (memory-bound,
                  and it moves the fewest bytes). Above it FP8 wins
                  (compute matters, and only FP8 raised the ceiling).
```

**That crossover is the "synchronous vs asynchronous" rule, derived rather than asserted.** Low batch — a synchronous, latency-bound, one-user-at-a-time deployment — is bytes-bound, so the format that moves the fewest bytes wins, and that is W4A16. High batch — asynchronous continuous batching, which is what this whole module is about — is drifting toward compute-bound, so the format that raised the FLOP ceiling wins, and that is W8A8. Neural Magic's deployment guidance says exactly this; now you can re-derive it for *your* model and *your* GPU instead of quoting it.

**Reality check on the absolute numbers.** These are roofline *ceilings*: they ignore attention-kernel inefficiency, sampling, the scheduler, Python overhead, and every layer that is not a quantized GEMM. Real vLLM runs land at roughly **35–50% of these figures**. Use the ratios (which are robust, because the un-modelled overheads affect all three columns similarly) and measure the absolutes. That is precisely what the deliverable asks you to do.

And the reason FP8's *measured* multiplier is 1.5–1.8× rather than the 2.0× in the table is 03.5 §12's Amdahl form, applied to serving:

```
    speedup = 1 / [ (1 − f) + f/2 ]      f = fraction of step time in
                                             FP8-accelerated work
    f = 0.90 → 1.82×
    f = 0.80 → 1.67×
    f = 0.70 → 1.54×      ← the shape of a real serving mix
    f = 0.50 → 1.33×
```

Inverting it is the useful move: **a measured 1.6× tells you about 75% of your step time was in the accelerated path**, which is a diagnosis, not just a number. If you measure 1.2×, `f ≈ 0.33` and something large is not being quantized — check whether the MoE layers, the embedding, or the LM head were excluded.

### 13. Cost per million tokens, worked for each choice

Convert throughput to money with lesson 05's equation:

```
  CPM ($ per 1M output tokens) =  hourly_rate
                                 ──────────────────────────  × 1e6
                                  tok/s × 3600 × utilisation
```

Take **one H100 at $2.99/hr**, the §12 table at batch 256, a **0.40 realization factor** on the roofline ceilings (the midpoint of the 35–50% band), and utilisation 1.0 (a saturated benchmark; §Perspectives explains why your fleet number will be worse).

```
  BF16 + bf16 KV
     tok/s   = 10,111 × 0.40 = 4,044
     CPM     = 2.99 / (4,044 × 3600 / 1e6) = 2.99 / 14.56 = $0.205 / M

  FP8 + fp8 KV
     tok/s   = 20,237 × 0.40 = 8,095
     CPM     = 2.99 / 29.14                              = $0.103 / M
     ⇒ 50% cheaper than BF16 at the roofline ratio

  FP8 + fp8 KV, Amdahl-derated to a MEASURED 1.6× over BF16
     tok/s   = 4,044 × 1.6 = 6,470
     CPM     = 2.99 / 23.29                              = $0.128 / M
     ⇒ 38% cheaper — the number to actually put in the report

  W4A16 + bf16 KV, batch 256
     tok/s   = 11,754 × 0.40 = 4,702
     CPM     = 2.99 / 16.93                              = $0.177 / M
     ⇒ only 14% cheaper than BF16 at high batch

  W4A16 + bf16 KV, batch 8  (a latency-bound, synchronous deployment)
     tok/s   = 4,176 × 0.40 = 1,670
     CPM     = 2.99 / 6.01                               = $0.497 / M
     vs FP8 at batch 8: 2,943 × 0.40 = 1,177 → $0.706 / M
     ⇒ at batch 8, W4A16 is 30% CHEAPER than FP8. The ranking INVERTS.
```

| Configuration | Batch | tok/s (realized) | **CPM** | vs BF16 |
|---|---|---|---|---|
| BF16 + bf16 KV | 256 | 4,044 | **$0.205** | — |
| BF16 + bf16 KV | 8 | 588 | **$1.413** | +590% |
| FP8 + fp8 KV (roofline) | 256 | 8,095 | **$0.103** | −50% |
| FP8 + fp8 KV (Amdahl 1.6×) | 256 | 6,470 | **$0.128** | **−38%** |
| W4A16 + bf16 KV | 256 | 4,702 | **$0.177** | −14% |
| W4A16 + bf16 KV | 8 | 1,670 | **$0.497** | −65% |
| FP8 + fp8 KV | 8 | 1,177 | **$0.706** | −50% |

Three readings:

1. **Batch size moves CPM more than format does.** BF16 at batch 256 ($0.205) beats *every* format at batch 8. Lesson 05's curve is still the biggest lever; quantization is the second.
2. **The format ranking inverts across the crossover**, exactly as §12 predicts. "W4A16 is cheaper" and "FP8 is cheaper" are both true statements about different operating points, and a report that does not state the batch is not a report.
3. **The honest FP8 number is the Amdahl-derated one.** Quoting −50% because the bytes halved is quoting a ceiling. Quote −38% and say it is measured, or quote −50% and label it a ceiling. Do not blur them; 03.5 §12 makes the same distinction for training and it is the same discipline.

### 14. The calibration-set trap — and the statistical-power trap behind it

**The calibration trap, mechanically.** Static activation quantization, GPTQ and AWQ all derive their scales from a small sample: 128 sequences for GPTQ's default, 512 for SmoothQuant's published scales, 512 sequences × 2048 tokens in llm-compressor's INT8 recipe. Those scales are fit to *that sample's distribution*.

Now feed a **generic prose** calibration set (Pile, C4, WikiText, ultrachat) to a model you serve for **code generation**. Code's activation distribution differs systematically: heavy indentation and bracket tokens, structured whitespace, long tails of rare identifiers, and — the part that actually matters — **different channels carrying the outliers**. §1's arithmetic then applies in the worst way: the scales are set for prose's outlier channels, so code's outlier channels either clip or force everything else toward zero. HumanEval pass@1 drops. MMLU, being prose-like, does not move. **A prose-only evaluation cannot see the regression it caused.**

llm-compressor's own best-practice list says the quiet part out loud: *"Employ the chat template or instruction template that the model was trained with"* and *"If you've fine-tuned a model, consider using a sample of your training data for calibration."* The tool's authors are telling you the calibration set is a workload-matching problem, not a hyperparameter.

Two escapes, in increasing order of cost:

- **Dynamic FP8** sidesteps the class entirely: no calibration file, scales from each forward pass. This is the strongest single argument for FP8 being the default on Hopper+ — not the throughput, the *absence of a silent failure mode*.
- **Native low-precision training** (Character.AI's int8 path, 03.5 §14) removes the post-hoc step altogether, eliminating train/serve mismatch by construction. Far beyond this module's scope — you are not retraining a model — but it is the correct answer to "how would you eliminate calibration risk entirely?", and having an answer beyond "use a better calibration set" is the Staff-level distinction.

**The statistical-power trap, arithmetically.** Here is the part most write-ups skip. vLLM's own FP8 documentation shows an evaluation result you should study for its *error bar*, not its value:

```
  |Tasks|Version|     Filter     |n-shot|  Metric   |   |Value|   |Stderr|
  |gsm8k|      3|flexible-extract|     5|exact_match|↑  |0.768|±  |0.0268|
                                                                  ▲
                              --limit 250 → the standard error is 2.68 POINTS
```

Verify it: for a proportion, `SE = sqrt(p(1−p)/n) = sqrt(0.768 × 0.232 / 250) = 0.0267`. ✓

Now the number you actually need — the error on the **difference between two runs**:

```
  SE(difference) = sqrt( SE_a² + SE_b² ) ≈ sqrt(2) × SE  = 1.414 × 0.0267 = 0.0378
  95% CI on the difference = ±1.96 × 0.0378 = ±0.074 = ±7.4 PERCENTAGE POINTS

  At --limit 500 with p ≈ 0.68:
     SE = sqrt(0.68 × 0.32 / 500) = 0.0209
     95% CI on the difference = ±1.96 × 1.414 × 0.0209 = ±5.8 points
```

**So a "+0.2 point" MMLU delta measured on a 500-sample slice is indistinguishable from zero, from −5.6, or from +6.0.** Reporting it as "no measurable accuracy loss" is not merely imprecise; it is a claim the experiment cannot support.

How many samples do you need to resolve a 1-point difference at 95% confidence?

```
  Require  1.96 × sqrt(2) × sqrt( p(1−p) / n )  ≤  0.01      with p ≈ 0.68

      2.772 × sqrt( 0.2176 / n )  ≤  0.01
      sqrt( 0.2176 / n )          ≤  0.003608
      0.2176 / n                  ≤  1.302e-5
      n                           ≥  16,700 samples

  For a 2-point resolution:  n ≥ 4,180
  For a 5-point resolution:  n ≥   670
```

**You cannot resolve a 1-point accuracy difference on an afternoon's eval budget.** That is not a reason to skip the measurement — it is a reason to *report the interval instead of the point*, and to state what your run could and could not have detected. The honest form of the claim is:

> *"FP8 vs BF16 on a 500-sample MMLU slice: 68.1 vs 68.3, difference +0.2 ± 5.8 points at 95%. The run rules out a degradation larger than ~6 points and says nothing below that. A 3,000-sample HumanEval slice resolves to ±2.4 points and moved −0.3."*

That sentence is worth more in a design review than any number in this lesson. It is also why an inference *provider* — serving thousands of customers' unknown workloads — reaches for **divergence metrics** (KL divergence between the quantized and unquantized next-token distributions) instead of task benchmarks: a divergence is measured per token, so a few hundred prompts yields hundreds of thousands of observations, and it catches a shift that flips a 0.51/0.49 token probability to 0.49/0.51 — real, but invisible to any accuracy metric at any feasible sample size.

### 15. What to actually ship — the decision procedure

```
  START: model, GPU fleet, workload
    │
    ├─▶ Is the whole fleet Hopper / Ada / Blackwell?
    │      YES ─▶ Is the operating batch above the §12 crossover (~30)?
    │              YES ─▶ ★ FP8 W8A8, dynamic scales, + FP8 KV cache
    │                        `--quantization fp8 --kv-cache-dtype fp8`
    │                        or an llm-compressor FP8_DYNAMIC checkpoint
    │              NO  ─▶ ★ W4A16 (AWQ or GPTQ, g128) + FP8 KV cache
    │                        latency-bound / synchronous / low-QPS
    │      NO (mixed or Ampere-and-older)
    │           ─▶ Does the model FIT at W8A16 / BF16 on the old cards?
    │                YES ─▶ ★ FP8 checkpoint everywhere; accept that Ampere
    │                          runs it as W8A16 (memory win only) — one
    │                          artifact, predictable behaviour
    │                NO  ─▶ ★ W4A16 as the portable artifact. NOT INT8:
    │                          INT8 is unsupported on Blackwell (§11)
    │
    ├─▶ ALWAYS, independent of the above:
    │      · FP8 KV cache — works on every generation from Turing up, and
    │        at production batch it cuts more HBM traffic than the weights do
    │      · If you must use STATIC activation scales or AWQ/GPTQ, the
    │        calibration set is drawn from YOUR traffic, versioned, and
    │        committed. Not from Pile.
    │
    └─▶ BEFORE ROLLOUT:
           · measure throughput and CPM at your real operating batch
           · measure accuracy on ≥2 slices: one general, one that looks like
             your production traffic
           · report the CONFIDENCE INTERVAL, not the point estimate (§14)
           · state which multiplier you are quoting: raw-kernel, roofline
             ceiling, or measured end-to-end
```

## Perspectives

**Developer / ML.** The calibration set is a dataset artifact with the same lifecycle problems as any training set: it drifts, it needs versioning, and its provenance determines what you can claim. llm-compressor's guidance to calibrate on your own fine-tuning data is really a statement that quantization is part of the model pipeline, not a deployment afterthought — which changes who owns it.

**Operator / observability.** Two checks turn "we shipped FP8" from a claim into evidence. First, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (03.5): if it did not move, the accelerated kernel path is not being taken — which is precisely what happens when an FP8 checkpoint lands on Ampere and silently becomes W8A16. Second, `vllm:kv_cache_usage_perc` and `vllm:num_requests_running` (module 05 §7): an FP8 KV cache should visibly raise the running-sequence count at the same usage percentage. If it did not, `--kv-cache-dtype` did not take effect. **A precision change that does not move a counter did not happen.**

**Hardware.** The FP8 multiplier is exactly 2 in silicon (8192 vs 4096 FLOP/SM/clock on Hopper, 03.5 §7) and never exactly 2 in your logs, because the rest of the pipeline is unchanged. W4A16's multiplier is exactly 4 on weight bytes and exactly 1 on FLOPs, which is why its advantage evaporates precisely where the workload stops being bytes-bound. Both statements are properties of the silicon that you can read off a datasheet and then predict a serving curve from — which is the whole point of module 03.

**Economics.** Quantization is the second-largest CPM lever after batch size, and §13's table shows the two interacting rather than adding: BF16 at batch 256 beats every format at batch 8. It also shows the ranking *inverting* across a computable crossover, which means "which quantization is cheapest" has no answer without an operating point. And the utilisation caveat from lesson 05 still dominates everything: a 38% CPM cut on a GPU running at 20% utilisation is a 38% cut on a number that is already 5× too high.

**Multi-tenant provider vs single-tenant company.** A provider makes the quantization decision *once* and amortises it across every customer's unknown traffic, so "MMLU didn't move" is not an adequate bar — a shift invisible to any accuracy benchmark at feasible sample sizes can still be real for some customer. Hence divergence metrics (§14). A single company quantizing its own model for its own product only has to validate against one traffic distribution, which it can actually sample. **Same technique, genuinely different evidentiary standard**, and knowing which situation you are in is the difference between over- and under-engineering the validation.

## Real-world use cases

- **vLLM — `docs/features/quantization/llm_compressor/fp8.md`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/llm_compressor/fp8.md>. Documents the `FP8_DYNAMIC` scheme as *static per-channel weights + dynamic per-token activations, requiring no calibration data*; states FP8 W8A8 needs compute capability ≥ 8.9 while ≥ 7.5 falls back to weight-only W8A16 via FP8 Marlin; quotes "2× reduction in model memory, up to 1.6× throughput"; and shows a gsm8k result of **0.768 ± 0.0268** at `--limit 250`. **What it shows:** the authoritative recipe *and* — in that stderr column — the statistical-power problem of §14, printed right next to the number everyone quotes and nobody reads.

- **vLLM — `docs/features/quantization/llm_compressor/int8_w8a8.md`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/llm_compressor/int8_w8a8.md>. The INT8 recipe is literally `[SmoothQuantModifier(smoothing_strength=0.8), GPTQModifier(scheme="W8A8")]` over 512 ultrachat samples at 2048 tokens, with best practices: start at 512 samples, use the model's own chat template, **and calibrate on your own training data if you fine-tuned**. Carries an explicit warning that **INT8 is unsupported on compute capability ≥ 10.0 (Blackwell)**. **What it shows:** SmoothQuant is a pre-processing stage rather than a quantizer; α=0.8 rather than the paper's 0.5 default is what production tooling actually ships; and the cross-platform-fallback story has an expiry date on new hardware.

- **IST-DASLab — GPTQ reference implementation** — <https://github.com/IST-DASLab/gptq>. `gptq.py` contains the mechanism in §6: `H = 2/n · Σ xxᵀ`, `percdamp=0.01` dampening, Cholesky inversion, `blocksize=128` column sweep with `W[:, i:] -= err ⊗ Hinv[i, i:]` error propagation, and the `--act-order` heuristic. The README's LLaMA table gives the numbers in §5: Wiki2 PPL 5.68 FP16 → 6.29 4-bit RTN → **6.09 4-bit GPTQ**; 25.54 3-bit RTN → **8.07 3-bit GPTQ**; and `--act-order` moving LLaMA-7B from 7.15 to 6.09. **What it shows:** the algorithm is 40 lines of readable PyTorch, and the RTN-vs-GPTQ gap widens dramatically as bits fall — which is why 8-bit needs no calibration and 3-bit needs a lot of it.

- **MIT HAN Lab — AWQ reference implementation** — <https://github.com/mit-han-lab/llm-awq>. `awq/quantize/auto_scale.py` contains the 20-point grid search over `α` minimising **block-output MSE** (not weight error), and `awq/utils/calib_data.py` shows the calibration source: the Pile validation split, 512-token blocks. MLSys 2024 Best Paper. **What it shows:** AWQ's "protect the salient 1%" is implemented as a search whose objective is end-to-end block output — and a search that touches the calibration data twenty times per block, which makes AWQ *more* calibration-sensitive than its "activation-aware" branding suggests, not less.

- **MIT HAN Lab — SmoothQuant reference implementation** — <https://github.com/mit-han-lab/smoothquant>. `smoothquant/smooth.py` implements `s_j = max|X_j|^α / max|W_j|^(1−α)` with `α=0.5` default, folded into the preceding LayerNorm/RMSNorm weight. The README's W8A8 perplexity table supplies §5's numbers (Llama-2-7B 5.474 → 5.515 at α=0.85; Llama-3-8B 6.138 → 6.258; Llama-3-70B 2.857 → 2.982) and shows α varying from 0.6 to 0.9 **per model**. Calibration = 512 random sentences from the Pile validation set. **What it shows:** α is a tuned per-model hyperparameter, not a constant — and Llama-3-70B degrading more than Llama-3-8B is a useful counterexample to "bigger models quantize better."

- **vLLM — `docs/features/quantization/quantized_kvcache.md`** (v0.27.1) — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/quantized_kvcache.md>. States that with no calibration, **all KV quantization scales are set to 1.0**; documents per-tensor vs per-attention-head scales (`q_scale=[num_heads]`, `k/v_scale=[num_kv_heads]`, Flash Attention backend only, via llm-compressor); notes that with the Flash Attention 3 backend **queries are also quantized to FP8** so attention runs in the quantized domain; and adds `--kv-cache-dtype-skip-layers` for sensitive layer types like sliding-window attention. **What it shows:** the default that everyone ships is the *uncalibrated* one, and "it works" is a statement about the model's K/V distribution happening to suit `s = 1.0`, not about the configuration being correct.

## Worked example

**Goal:** produce, for the deliverable, an **honest FP8-vs-BF16 CPM saving with a measured accuracy delta and its confidence interval**, on one rented H100. Pin **vLLM v0.27.1** everywhere.

### Step 1 — decide what you are even testing, before renting anything

```
  Model    Llama-3.1-8B-Instruct    P = 8.03e9
  GPU      1× H100 80GB @ $2.99/hr  HBM 3.35 TB/s, FP8 dense 1,979 TFLOP/s
  Target   asynchronous serving, continuous batching, --max-num-seqs 256

  §12 crossover check:
      operating batch 256  ≫  crossover ≈ 30      ⇒ W8A8 territory, not W4A16
  §11 hardware check:
      H100 = sm90 ⇒ FP8 W8A8 native, FP8 block scaling available,
                    INT8 available but pointless here
  §9 calibration check:
      dynamic FP8 needs no calibration set ⇒ zero calibration risk
                    ⇒ start here, and only reach for a static/AWQ variant
                      if the measurement says dynamic is leaving money behind
```

**That is the whole design, decided on paper.** Renting a GPU to discover that W4A16 loses at batch 256 is renting a GPU to re-derive §12.

### Step 2 — serve both variants

```bash
# BF16 baseline
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 --max-num-seqs 256 --max-model-len 4096

# FP8: online dynamic weight+activation quantization, plus an FP8 KV cache.
# --quantization fp8  → per-tensor FP8 weights at load, dynamic activation scales
# --kv-cache-dtype fp8 → E4M3 KV cache. NOTE: uncalibrated, scales = 1.0 (§10)
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8001 --max-num-seqs 256 --max-model-len 4096 \
  --quantization fp8 --kv-cache-dtype fp8

vllm --version   # record it. Every number below is version-specific.
```

Before benchmarking, **prove the change landed** — the §Perspectives check:

```bash
# KV blocks available should roughly DOUBLE on the fp8 endpoint.
curl -s localhost:8000/metrics | grep -E 'cache_config_info|kv_cache'
curl -s localhost:8001/metrics | grep -E 'cache_config_info|kv_cache'

# And tensor-pipe activity should rise under load on the fp8 endpoint.
dcgmi dmon -e 1004,1005 -d 100    # PIPE_TENSOR_ACTIVE, DRAM_ACTIVE
```

If the KV block count did not move, `--kv-cache-dtype` did not take effect and every number after this point is measuring the wrong thing.

### Step 3 — sweep, do not spot-check

The single most common mistake is benchmarking one batch size. §12 says the answer depends on batch, so sweep it:

```bash
for PORT in 8000 8001; do
  for C in 1 8 32 64 128 256; do
    vllm bench serve \
      --backend vllm --model meta-llama/Llama-3.1-8B-Instruct \
      --host localhost --port $PORT \
      --dataset-name random --random-input-len 1024 --random-output-len 256 \
      --request-rate inf --max-concurrency $C --num-prompts $((C * 20)) \
      --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
      --save-result --result-filename bench_${PORT}_c${C}.json
  done
done
```

Read `output_throughput`, `p99_ttft_ms` and `p99_tpot_ms` from each JSON, and convert to CPM with `$2.99/hr`. Representative shape (record *your* numbers — these are illustrative of the pattern §12 predicts, not captured output):

| Concurrency | BF16 tok/s | FP8 tok/s | Ratio | BF16 CPM | FP8 CPM |
|---|---|---|---|---|---|
| 1 | 96 | 168 | 1.75× | $8.65 | $4.94 |
| 8 | 640 | 1,090 | 1.70× | $1.30 | $0.76 |
| 32 | 1,890 | 3,120 | 1.65× | $0.44 | $0.27 |
| 128 | 3,410 | 5,460 | 1.60× | $0.244 | $0.152 |
| 256 | 4,020 | 6,310 | **1.57×** | **$0.207** | **$0.132** |

**CPM cut at the operating point: `1 − 0.132/0.207 = 36%`.** And invert the Amdahl form on the 1.57× to get a diagnosis for free:

```
  1.57 = 1 / [ (1 − f) + f/2 ]   ⇒   f = 2 × (1 − 1/1.57) = 0.726

  ⇒ about 73% of step time was in the FP8-accelerated path.
    Healthy. The other 27% is attention softmax, sampling, the scheduler,
    and the un-quantized lm_head/embeddings/norms.
    If you measured f ≈ 0.35, something large is not quantized — go look.
```

Note the ratio *falls* as concurrency rises (1.75× → 1.57×), because at high batch the un-quantized attention work grows with `B × ctx` while the weight-streaming saving is fixed. That is the KV-traffic term from §10 showing up in a measurement.

### Step 4 — measure the accuracy delta, and its interval

Two slices, chosen deliberately: one general, one resembling production traffic.

```bash
# General knowledge — larger n, because you need the resolution (§14)
lm_eval --model vllm \
  --model_args pretrained=meta-llama/Llama-3.1-8B-Instruct,add_bos_token=True \
  --tasks mmlu --num_fewshot 5 --limit 3000 --batch_size auto

lm_eval --model vllm \
  --model_args pretrained=meta-llama/Llama-3.1-8B-Instruct,quantization=fp8,kv_cache_dtype=fp8,add_bos_token=True \
  --tasks mmlu --num_fewshot 5 --limit 3000 --batch_size auto

# Code slice — the one the calibration caveat would show up on
lm_eval --model vllm \
  --model_args pretrained=meta-llama/Llama-3.1-8B-Instruct,add_bos_token=True \
  --tasks humaneval --limit 164 --confirm_run_unsafe_code
# repeat with quantization=fp8
```

*(`add_bos_token=True` matters: `lm_eval` does not add a BOS token by default, and vLLM's own docs warn that quantized models can be sensitive to its absence. Forgetting it can manufacture a difference that has nothing to do with quantization.)*

Now report it correctly. Suppose MMLU comes back 68.1 vs 68.3 at `n = 3000`:

```
  SE per run      = sqrt(0.681 × 0.319 / 3000) = 0.00851  = 0.85 points
  SE(difference)  = sqrt(2) × 0.00851          = 0.01204
  95% CI          = ±1.96 × 0.01204            = ±2.36 points

  REPORT: "MMLU 5-shot, n=3000: 68.1 → 68.3, Δ = +0.2 ± 2.4 points (95%).
           The run rules out degradation worse than ~2.2 points."

  HumanEval is only 164 problems TOTAL — there is no larger n available:
  SE per run     = sqrt(0.63 × 0.37 / 164) = 0.0377 = 3.8 points
  95% CI on Δ    = ±10.4 points
  REPORT: "HumanEval pass@1, n=164 (full set): 63.4 → 63.0, Δ = −0.4 ± 10.4
           points (95%). This slice cannot resolve anything under ~10 points;
           treat it as a smoke test, and use a larger internal code eval for
           a real gate."
```

**That HumanEval line is the most valuable sentence in the exercise.** Almost every quantization write-up you will read reports a HumanEval delta of a point or two as if it were a finding. It cannot be. Knowing — and saying — that the benchmark's own size caps your resolution at ±10 points is what makes the rest of your numbers credible.

### Step 5 — the claim you can defend

> *On 1× H100 at $2.99/hr, serving Llama-3.1-8B-Instruct at `--max-num-seqs 256`, FP8 weights + FP8 KV cache (vLLM v0.27.1, dynamic scales, no calibration set) delivered **1.57× output throughput and a 36% CPM reduction** ($0.207 → $0.132 per million output tokens). Inverting Amdahl on the 1.57× puts ~73% of step time in the accelerated path, which is consistent with a healthy serving mix. Accuracy: MMLU 5-shot at n=3000 moved +0.2 ± 2.4 points (95% CI) — degradation worse than ~2.2 points is ruled out; HumanEval at its full n=164 moved −0.4 ± 10.4 points and is a smoke test only. The KV cache is running **uncalibrated** at the default scale of 1.0; calibrating it with llm-compressor is the next step if we need a tighter bound.*

Every clause in that paragraph is checkable, and the two hedges are in it because the measurement genuinely does not support removing them.

## Practice

Feeds the **Cost-per-million-tokens** deliverable at [`../practice/cost-per-token/README.md`](../practice/cost-per-token/README.md). Pin vLLM v0.27.1 and record the version in every artifact.

1. **Decide on paper first.** Before renting: compute your model's §12 ridge points for BF16, FP8 and W4A16 on your target GPU, and state the crossover batch. Write down which format your operating batch implies. **Acceptance:** three ridge-point calculations with units carried, and a one-line prediction of which format wins at your batch — recorded *before* any measurement.

2. **Sweep, do not spot-check.** Run the Step-3 sweep at concurrency {1, 8, 32, 64, 128, 256} for BF16 and FP8. **Acceptance:** a CPM-vs-concurrency table for both formats with the throughput ratio per row, and a sentence explaining why the ratio changes across the sweep (§10's KV-traffic term).

3. **Prove the change landed.** Capture `cache_config_info` / KV-block counts on both endpoints and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` under load. **Acceptance:** evidence that KV capacity roughly doubled with `--kv-cache-dtype fp8` and that tensor-pipe activity moved. A CPM claim with no evidence the precision change took effect does not pass.

4. **Measure accuracy with an interval, not a point.** Run one general slice at the largest `n` you can afford and one production-shaped slice. **Acceptance:** for each slice, the two scores, the difference, **the 95% confidence interval computed from the binomial standard error**, and an explicit statement of what magnitude of regression the run could and could not have detected.

5. **Invert Amdahl on your measured multiplier** to estimate the accelerated fraction `f`. **Acceptance:** the arithmetic and a one-line interpretation ("f ≈ 0.73, consistent with a healthy mix" or "f ≈ 0.35 — investigate what is not quantized").

6. **(Stretch) Demonstrate the crossover.** Quantize the same model to W4A16 with llm-compressor's GPTQ recipe and add it as a third row at concurrency 1, 8 and 256. **Acceptance:** a table showing W4A16 winning at low concurrency and losing at high concurrency, with the measured crossover compared against your Step-1 prediction.

7. **(Stretch) Demonstrate the calibration trap.** Produce two W4A16 checkpoints — one calibrated on generic prose (ultrachat/Pile), one on a code-heavy set — and evaluate both on a code slice and a prose slice. **Acceptance:** the four numbers with intervals. If the intervals overlap, say so; a null result honestly reported is a pass, an over-claimed result is not.

**Acceptance overall:** the deliverable contains the **FP8-vs-BF16 CPM saving at a stated operating batch**, *and* a **measured accuracy delta with its confidence interval** on a general and a production-shaped slice, *and* evidence the precision change actually took effect. A CPM number with no paired accuracy interval does not pass; neither does an accuracy point estimate with no interval.

## Common pitfalls

1. **Quoting the 2× datasheet number as the expected result.** *Mechanism:* the FP8 tensor-core multiplier is exactly 2 in silicon (8192 vs 4096 FLOP/SM/clock, 03.5 §7), but only the accelerated fraction of step time benefits — Amdahl gives 1.5–1.8× for a realistic serving mix, and §10 shows why: at production batch, KV traffic exceeds weight traffic and only *KV* quantization touches it. Report the measured number and invert Amdahl to say what fraction was accelerated.

2. **Benchmarking one batch size.** *Mechanism:* §12 — W4A16's ridge point is ~3.8× lower than FP8's, so the format ranking **inverts** at a computable crossover (batch ≈30 for 8B on H100). A single-point benchmark can "prove" either format is better depending purely on which point you picked.

3. **Reporting an accuracy delta without its confidence interval.** *Mechanism:* §14 — at `--limit 500` the 95% CI on a difference is about ±6 points, and HumanEval's full 164 problems cap you at ±10. A "+0.2 point, no measurable loss" claim on 500 samples is noise reported as a finding. Report the interval and what it rules out.

4. **Deploying an FP8 checkpoint to an Ampere pool and expecting throughput.** *Mechanism:* §11 — vLLM runs FP8 on compute capability ≥ 7.5 as weight-only W8A16 via FP8 Marlin. It does not error; it silently gives you the memory saving without the compute saving. Symptom: memory drops, tokens/s does not move, `PIPE_TENSOR_ACTIVE` unchanged.

5. **Making INT8 the cross-platform fallback policy.** *Mechanism:* §11 — INT8 is **unsupported on compute capability ≥ 10.0**, so the policy breaks on your *newest* hardware. W4A16 spans Turing through Blackwell and is the correct portable artifact.

6. **Shipping SmoothQuant at α = 0.5 because that is the function default.** *Mechanism:* §8 — the published α varies 0.6–0.9 by model and llm-compressor's production recipe uses 0.8. You are leaving accuracy on the table for a parameter that costs nothing to set.

7. **Assuming `--kv-cache-dtype fp8` is calibrated.** *Mechanism:* §10 — with no calibration, vLLM sets all KV scales to **1.0**. It usually works because attention K/V magnitudes happen to fit E4M3's window at unit scale, which is luck-shaped rather than correctness-shaped. Say "uncalibrated" in the report, and calibrate with llm-compressor if you need a tighter bound.

8. **Calibrating on generic prose for a non-prose workload.** *Mechanism:* §14 — scales are fit to the calibration distribution's outlier channels; a different workload has different outlier channels, so its important activations clip or round to zero. MMLU (prose-like) will not show it. llm-compressor's own best practices say to calibrate on your own training data.

9. **Evaluating only on general benchmarks.** *Mechanism:* the corollary of (8). Always add a slice that resembles production traffic — and size it for the resolution you need (§14).

10. **Confusing granularity with algorithm.** *Mechanism:* §3 — moving weights from per-tensor to per-channel scales is close to free and recovers most of a naive scheme's loss. If your result looks worse than published, check granularity *before* you change methods.

11. **Following an `AutoAWQ` runbook.** *Mechanism:* the library is deprecated and vLLM's own AWQ documentation says so, directing you to `llm-compressor`'s AWQ examples. An old `pip install autoawq` line in a repo is a live trap.

12. **Forgetting `add_bos_token=True` in `lm_eval`.** *Mechanism:* `lm_eval` does not add a BOS token by default and vLLM's docs warn quantized models can be sensitive to its absence — so you can manufacture an accuracy "delta" that is entirely a tokenization artifact.

## Self-check

- **(a) What throughput multiplier should you expect from FP8 on an H100, roughly what CPM reduction, and why is it not 2×?**
  **Answer:** A *measured* **~1.5–1.8×** sustained throughput, giving a CPM cut of `1 − 1/1.6 ≈ 38%` at a fixed GPU price (roughly 40–50% if you quote the roofline ceiling — say which you mean). It is not 2× for two compounding reasons. First, Amdahl: `speedup = 1/[(1−f) + f/2]`, and only the FP8-accelerated fraction `f` of step time benefits — attention softmax, sampling, the scheduler and un-quantized layers do not. A measured 1.6× implies `f ≈ 0.75`, which is a healthy mix. Second, and more specific to serving: at production batch, **KV traffic exceeds weight traffic**. For 8B at 2k context and batch 128, weights are 16.1 GB per step while KV is 34.4 GB — so halving the weights cuts only ~16% of HBM traffic. Adding `--kv-cache-dtype fp8` is what attacks the other 68%, which is why the two flags belong together.

- **(b) Why can a generic calibration set hurt a code-generation workload specifically, and what escapes the problem?**
  **Answer:** Static activation quantization, AWQ and GPTQ all fit their scales to a calibration sample (128–512 sequences). Transformer activations contain **outlier channels** — specific feature dimensions, consistent across tokens, running 1–2 orders of magnitude above the rest — and which channels those are depends on the input distribution. Fit scales on prose and the scale `s` is sized for prose's outliers; code's outliers then either clip (unbounded error) or force `s` up so that ordinary code activations round toward zero (§1's worked example: one 10× outlier turned a 12%-error value into an exact zero). HumanEval pass@1 drops while MMLU, being prose-like, does not move — so a prose-only evaluation cannot see the damage it caused. Escapes, in cost order: **dynamic FP8** (scales computed per forward pass, no calibration file, no silent staleness — the strongest argument for FP8 as the Hopper+ default); calibrating on your own traffic (llm-compressor's own best-practice advice for fine-tuned models); and native low-precision training (Character.AI's int8 path), which removes the post-hoc step entirely.

- **(c) When do you reach for W4A16 (AWQ/GPTQ) over FP8 — and can you derive the boundary rather than quote it?**
  **Answer:** When you are **bytes-bound**, i.e. below the crossover batch, or when the model does not otherwise fit. Derive it from the ridge point — the batch at which compute time overtakes memory time, `2·P·B / FLOPS = W_bytes / HBM_BW`. For Llama-3.1-8B on an H100: BF16 → B≈296, FP8 → B≈295 (unchanged, because FP8 halved *both* bytes and the FLOP ceiling), **W4A16 → B≈79**, because it quartered the bytes but left the matmul at BF16 rate. So W4A16 saturates far earlier and, past its ridge, is compute-bound at BF16 rate with dequantization overhead on top. Including KV traffic, the practical crossover lands near **batch ≈30**: below it W4A16 wins (at batch 1 it is ~3.6× BF16 versus FP8's ~1.8×), above it FP8 wins (at batch 256, ~1.7× ahead of W4A16). That is the "W4A16 for synchronous/latency-bound, W8A8 for asynchronous continuous batching" rule, derived instead of memorised. Also note W4A16 is really **4.25 bits/weight** at `g=128` once scales and zero-points are counted.

- **(d) A colleague reports "FP8 gave us 1.2×." What do you tell them to check?**
  **Answer:** Invert Amdahl: `1.2 = 1/[(1−f) + f/2]` ⇒ `f = 2 × (1 − 1/1.2) = 0.33`. Only a third of step time was in the accelerated path, which is low for a serving workload — something large is not being quantized or not being accelerated. Check, in order: **(1) the GPU generation** — on Ampere or Turing, vLLM runs FP8 as weight-only W8A16 via FP8 Marlin, so there is no compute win at all by design (§11); confirm with `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`. **(2) whether the KV cache was quantized** — at production batch, KV traffic exceeds weight traffic, so weights-only FP8 caps the achievable gain (§10). **(3) which modules were excluded** — `lm_head` is always skipped, and MoE expert layers or the embedding may be too. **(4) the batch size** — at batch 1 nothing is compute-bound, so the FLOP-ceiling half of FP8's benefit is unavailable and you only get the byte halving.

- **(e) You measure MMLU 68.1 → 68.3 on a 500-sample slice and want to write "no accuracy loss." What is wrong with that?**
  **Answer:** The experiment cannot support the claim. For a proportion, `SE = sqrt(p(1−p)/n) = sqrt(0.68 × 0.32 / 500) = 0.0209`, i.e. 2.1 points per run; the standard error on the *difference* between two runs is `sqrt(2) × 0.0209 = 0.0295`, so the 95% CI on the difference is **±5.8 percentage points**. A +0.2 point observation is statistically indistinguishable from −5.6 or +6.0. To resolve a 1-point difference at 95% confidence you need `n ≥ ~16,700` samples; 2 points needs ~4,200; 5 points needs ~670. The correct report is the interval and what it excludes: *"Δ = +0.2 ± 5.8 points (95%); degradation worse than ~5.6 points is ruled out, nothing smaller is resolved."* This also explains why a multi-tenant provider uses **divergence metrics** on next-token distributions instead — a divergence is measured per token, so a few hundred prompts yields hundreds of thousands of observations and can detect a distribution shift that no accuracy benchmark could at any feasible sample size.

- **(f) Sketch a quantization policy for a fleet of A100s and H100s that will add Blackwell next year.**
  **Answer:** No single checkpoint is optimal everywhere, so the policy is per-tier with one universal element. **FP8 W8A8 (dynamic scales) on H100 and future Blackwell** — native tensor-core path, no calibration artifact, ~1.6× measured. **On A100, expect the same FP8 checkpoint to run as weight-only W8A16 via FP8 Marlin** — you keep the memory halving, you get no compute win, and that is by design rather than a misconfiguration; if the A100 tier needs more than that, ship a **W4A16 (AWQ or GPTQ, g128)** artifact for it. **Do not standardise on INT8 as the cross-platform fallback**, even though it is the historically obvious choice: vLLM's own docs mark INT8 unsupported on compute capability ≥ 10.0, so the policy would break precisely on the hardware you are about to buy. W4A16 spans Turing through Blackwell. **Universally, enable an FP8 KV cache** — it works on every generation from Turing up, is storage rather than a tensor-core operation, and at production batch cuts more HBM traffic than the weights do. Flag that the KV scales are uncalibrated (=1.0) unless you run llm-compressor, and validate each tier separately, because "the checkpoint is the same" does not mean "the numerics are the same."

## Connections & what's next

Quantization is the second explicit cost lever in this module (after 05's batching curve), and §13 shows the two are not independent: BF16 at batch 256 is cheaper than any format at batch 8, and the format ranking inverts across a crossover you can compute. It also compounds forward. A quantized checkpoint is **smaller**, so it reads off storage and crosses PCIe faster — lesson 09 turns that into cold-start seconds. It **frees VRAM**, so `--max-num-seqs` can rise, which is the headroom lesson 08's autoscaler is spending. And §11's fleet-policy problem — one artifact, several hardware tiers, different behaviour on each — is the first place in this module where a single decision has to be defended to more than one GPU generation at once, which is the shape of every capacity argument in Module 11.

It also closes a loop backwards. Lesson 06 §11 showed that an FP8 KV cache halves the bytes a disaggregated deployment must ship across the fabric, moving the transfer-to-prefill ratio from 0.37 to 0.19 on 400 Gb/s InfiniBand. **Quantization is not only a per-GPU numerics decision; it is an architecture enabler.** The two levers you were told were orthogonal turn out to multiply.

The discipline this lesson trains is the one the deliverable depends on everywhere: a claimed saving is only real once it is measured *at your operating point*, with an error bar, on data that looks like your traffic.

Next: [**07.8 · Autoscaling inference**](08-autoscaling-inference.md), where the question stops being "how cheap is one replica" and becomes "how many replicas should exist right now" — and where the VRAM this lesson just freed becomes the headroom the control loop is spending.

## References & further reading

**Primary sources — engine documentation (verified against the repository at v0.27.1)**

1. **vLLM — `docs/features/quantization/llm_compressor/fp8.md`** — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/llm_compressor/fp8.md> — the `FP8_DYNAMIC` scheme (static per-channel weights, dynamic per-token activations, **no calibration data required**), the compute-capability gate (≥ 8.9 for W8A8, ≥ 7.5 falls back to W8A16 via FP8 Marlin), the "2× memory, up to 1.6× throughput" framing, the online `--quantization fp8` description (per-tensor weights, dynamic per-tensor activations, "latency improvements are limited in this mode"), and the gsm8k result `0.768 ± 0.0268` at `--limit 250` used in §14.
2. **vLLM — `docs/features/quantization/llm_compressor/int8_w8a8.md`** — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/llm_compressor/int8_w8a8.md> — the production INT8 recipe `[SmoothQuantModifier(smoothing_strength=0.8), GPTQModifier(scheme="W8A8")]`, 512 ultrachat samples at 2048 tokens, the best-practice list (calibrate on your own training data if fine-tuned; use the model's chat template), and the **Blackwell (≥ sm100) INT8 unsupported** warning.
3. **vLLM — `docs/features/quantization/quantized_kvcache.md`** — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/quantized_kvcache.md> — per-tensor vs per-attention-head KV scales, the **default scales of 1.0 with no calibration**, the FA3-backend note that queries are also quantized, and `--kv-cache-dtype-skip-layers`.
4. **vLLM — `docs/features/quantization/README.md` and `online.md`** — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/README.md> — the hardware compatibility chart reproduced in §11, and the online-quantization shorthands (`fp8_per_tensor`, `fp8_per_block`, `fp8_per_channel`, `mxfp8`, `mxfp4`) with their weight/activation recipes and block sizes.
5. **vLLM — `docs/features/quantization/auto_awq.md`** — <https://github.com/vllm-project/vllm/blob/main/docs/features/quantization/auto_awq.md> — the deprecation warning on `AutoAWQ` and the redirect to `llm-compressor`'s AWQ examples. **Correction:** runbooks that install `autoawq` are out of date.
6. **vLLM — `vllm/config/cache.py`** — <https://github.com/vllm-project/vllm/blob/main/vllm/config/cache.py> — the `CacheDType` literal (`auto`, `float16`, `bfloat16`, `fp8`, `fp8_e4m3`, `fp8_e5m2`) behind `--kv-cache-dtype`.
7. **TensorRT-LLM — `docs/source/features/quantization.md`** (1.3.0-rc) — <https://github.com/NVIDIA/TensorRT-LLM/blob/main/docs/source/features/quantization.md> — the per-architecture hardware support matrix cross-checked in §11, including the Ampere row showing FP8 KV cache supported while FP8 weight/activation is not, and the note that sm100/103 FP8 block scaling uses an MXFP8 (UE8M0 scale) recipe distinct from SM90's FP32-scale recipe.

**Primary sources — method reference implementations**

8. **IST-DASLab — GPTQ** (`gptq.py`, `llama.py`), ICLR 2023 — <https://github.com/IST-DASLab/gptq> — the Hessian accumulation `H = 2/n·Σxxᵀ`, `percdamp=0.01`, Cholesky inversion, `blocksize=128` column sweep with error propagation `W[:, i:] -= err ⊗ Hinv[i, i:]`, `--nsamples 128` default, and the `--act-order` heuristic. README's LLaMA table supplies §5's PPL figures (4-bit RTN 6.29 vs GPTQ 6.09; 3-bit RTN 25.54 vs GPTQ 8.07; `--act-order` moving 7.15 → 6.09).
9. **MIT HAN Lab — AWQ / llm-awq** (`awq/quantize/auto_scale.py`, `awq/utils/calib_data.py`), MLSys 2024 Best Paper — <https://github.com/mit-han-lab/llm-awq> — the 20-point grid search over `α` minimising **block-output MSE**, the scale normalisation `s / sqrt(max(s)·min(s))`, and the calibration source (Pile validation split, 512-token blocks). Paper: arXiv 2306.00978. *(arxiv.org is unreachable from this environment's egress proxy; §7's mechanism is verified against the reference implementation, and the paper is cited for provenance.)*
10. **MIT HAN Lab — SmoothQuant** (`smoothquant/smooth.py`), ICML 2023 — <https://github.com/mit-han-lab/smoothquant> — the migration formula `s_j = max|X_j|^α / max|W_j|^(1−α)` with `α=0.5` default, folded into the preceding LayerNorm/RMSNorm; the W8A8 perplexity table used in §5 (Llama-2-7B 5.474→5.515 @ α=0.85, Llama-3-8B 6.138→6.258, Llama-3-70B 2.857→2.982, Falcon-7B @ α=0.6, Llama-2-70B @ α=0.9); and the calibration protocol (512 random Pile-validation sentences). Paper: arXiv 2211.10438. *(Same arxiv caveat as entry 9.)*
11. **Microsoft — LoRA reference (`loralib/layers.py`)** — <https://github.com/microsoft/LoRA> — not a quantization source, but the `scaling = lora_alpha / r` convention referenced when 07.10 stacks adapters on a quantized base.

**Studies and production practice**

12. **Kurtic, Marques, Pandit, Kurtz, Alistarh (Neural Magic / IST Austria) — "Give Me BF16 or Give Me Death? Accuracy-Performance Trade-Offs in LLM Quantization"** (ACL 2025, arXiv 2411.02355) — the >500,000-evaluation study behind §5's aggregate recovery figures (**~99.75%** for 8-bit W8A8, **~99.36%** for W4A16-INT across the Llama-3.1 family) and the sync-vs-async deployment rule that §12 re-derives from the roofline. **Provenance caveat:** arxiv.org is unreachable from this environment's egress proxy, so these two recovery percentages are carried forward from the previous version of this lesson rather than re-verified against the paper; re-check them before external citation. The §12 derivation does not depend on them.
13. **Fireworks AI — "How Fireworks evaluates quantization precisely and interpretably"** — <https://fireworks.ai/blog/fireworks-quantization> — the divergence-metrics methodology referenced in §14 and Perspectives: measuring KL divergence between quantized and unquantized next-token distributions rather than task-benchmark pass/fail, because a shift that flips a 0.51/0.49 probability is invisible to accuracy metrics and task benchmarks can even show noise-driven "improvement." Dated snapshot; treat the specific claims as of its publication date.
14. **Character.AI — "Optimizing AI Inference at Character.AI"** — <https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/> — native int8 training across weights, activations and KV cache, chosen explicitly to eliminate train/serve mismatch rather than to find a better calibration set; the architecture-level answer to §14's trap. Cost figures in that post are dated — verify before quoting externally.
