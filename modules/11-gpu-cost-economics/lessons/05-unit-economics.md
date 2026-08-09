---
lesson: 05
title: "Unit economics: joining infra dollars to application counters"
module: 11
concept: "infra-$ to business-unit join"
status: not-started
est_time: "4 hrs"
artifacts: ["a $/1M-token calculator that joins attributed GPU-hours × blended rate ÷ app-emitted token counts, emitting both direct and fully-loaded numbers"]
---

# Unit economics: joining infra dollars to application counters

[💰 11 — GPU cost and unit economics](../README.md)

## Why this matters

Every prior cost lesson gave you one half of a fraction. Lesson 07 taught cost-per-token as a serving metric; lesson 08 taught cost-per-run and MFU as a training metric. This lesson is the general operator that produces *both* of those and any other unit your business cares about, from the same two ledgers: an infra ledger denominated in **$/GPU-hour** and an application ledger denominated in **business units** (tokens, requests, runs, experiments, customers). Unit economics is the join between them. If you cannot state your product's cost in $/1M tokens or $/training-run *and defend the denominator and the loading*, you do not actually know your gross margin — you know your cloud bill, which is a different and less useful number.

This is the single most-probed competency in a GPU-platform or inference-infra interview, because it is where FinOps meets the product P&L. When an inference-provider (Together, Fireworks, Baseten, Anyscale) or a frontier lab's serving team asks "what does a million tokens cost us," the naive answer — divide the monthly GPU bill by monthly tokens — fails three follow-ups: it ignores per-namespace attribution, it uses list rate instead of blended committed rate, and it silently drops the idle/fragmentation tax onto the floor. The senior answer walks the join key explicitly and shows where the money leaks.

The reason this matters beyond interviews: unit cost is the number that decides pricing, that decides build-vs-rent (lesson 06, module 10), and that decides which workloads get killed in a capacity crunch. A unit cost computed on the *direct* basis makes a service look profitable while the fleet as a whole bleeds; a unit cost computed *fully-loaded* tells you the truth but can make a healthy service look underwater if you misallocate the waste. Knowing which number is honest in which decision is the actual skill.

## What's new here

- **You already know** cost-per-token (07) and cost-per-run + MFU (08) as standalone metrics. **Skip** re-deriving them. Here they are two instances of one formula, and the point is the *general join*, not either endpoint.
- **You already know** per-pod GPU attribution (04/05) and the SM_ACTIVE-vs-GPU_UTIL util-lie (05). **Skip** the mechanism. Here it becomes the *numerator sourcing* — the attributed GPU-hours are the input, and the util-lie is why a "utilized" fleet still has a fully-loaded cost far above its direct cost.
- **New angle 1:** treating $/token and $/run as the *same formula* with different denominators, so you can synthesize a *new* unit on demand — including **$/successful-experiment**, which divides by successful runs and thereby prices your failure rate.
- **New angle 2:** **blended vs marginal rate** as a deliberate choice in the numerator's rate term, and how commitment coverage (lesson 06) moves the blended number.
- **New angle 3:** **direct vs fully-loaded** unit cost — whether the idle/fragmentation/control-plane/storage waste ledger (03/04) gets allocated across units, and when each basis is the honest one.

## Core notes

### The general formula

$$
\text{unit cost} = \frac{H_{\text{attr}} \times r_{\text{blend}}}{U}
$$

- $H_{\text{attr}}$ — GPU-hours attributed to the workload in a time window, from the attribution regime of lessons 01/04/05 (namespace/pod-level, time-aligned).
- $r_{\text{blend}}$ — blended $/GPU-hour for that window (on-prem TCO/GPU-hr from module 10, or the rental/committed rate, blended across the fleet's commitment mix — see lesson 06).
- $U$ — units produced in the *same window*, from an application-level counter emitted by the workload.

The whole discipline is: **make the window and the workload identity align on all three terms.** A mismatch — GPU-hours from Kubecost billing-day boundaries, tokens from an app dashboard on a different timezone — silently corrupts the ratio. Pin one window (hour or day), one workload key (namespace, or namespace × model), and pull all three from that.

### The join, term by term

| Term | Source | Gotcha |
|---|---|---|
| $H_{\text{attr}}$ | DCGM / attribution exporter → GPU-seconds per pod, summed to namespace, ÷ 3600 (lessons 01/04) | Must be *allocated* GPU-hours (what you're paying for), not *active* GPU-hours. A 30%-SM_ACTIVE GPU still bills 1.0 GPU-h/h. |
| $r_{\text{blend}}$ | On-prem: TCO/GPU-hr (mod 10). Cloud: weighted mean of committed + on-demand + spot $/GPU-hr for that window (mod 06). | Do not use list/on-demand rate if you're 80% committed — that overstates cost 30-60%. |
| $U$ | Serving: tokens from vLLM/TGI `/metrics` (`*_prompt_tokens_total`, `*_generation_tokens_total`) — ref 07. Training: completed runs / samples / steps — ref 08. | Counters are cumulative; take the window *delta*, and decide prefill vs decode inclusion up front. |

### The three canonical units — one formula

**$/1M tokens (inference).** $U$ = generation (or prompt+generation) tokens, ×$10^6$ normalization. The prefill/decode asymmetry and batching amortization (07) live entirely inside how $U$ moves for a given $H_{\text{attr}}$: bigger effective batch → more tokens per GPU-hour → lower unit cost, no change to the formula. Decide once whether you price on **output tokens only** (matches most API price sheets) or prompt+output (matches true compute); state it.

**$/training-run or $/experiment (training).** $U$ = 1 run (or 1 epoch, or 1B samples). MFU (08) is the efficiency multiplier: at fixed $H_{\text{attr}}$, doubling MFU halves the GPU-hours needed per run and thus the unit cost. Same formula, denominator is runs.

**$/successful-experiment (research fleet) — new.** $U$ = *successful* runs only, but $H_{\text{attr}}$ still counts hours burned by crashed, OOM'd, diverged, and pre-empted jobs. This deliberately loads failure into the unit:

$$
\frac{\$}{\text{successful run}} = \frac{H_{\text{all runs}} \times r_{\text{blend}}}{N_{\text{success}}} = \frac{\$}{\text{run}}\times\frac{1}{\text{success rate}}
$$

A 60% success rate makes each *usable* result 1.67× the naive per-run cost. Neoclouds and research platforms (e.g. internal-lab experiment trackers, CoreWeave/Lambda customer dashboards) surface this because it prices reliability and checkpoint hygiene, not just FLOPs. It is the cleanest single number for "how much does our iteration inefficiency cost."

### Blended vs marginal rate

- **Blended** $r$ — bill units at the fleet's average $/GPU-hr across the commitment mix. Correct for **steady-state margin and pricing**: it reflects what capacity actually costs you on average.
- **Marginal** $r$ — bill the *next* unit at the rate of the capacity that serves it: spot or on-demand top-up above your committed baseline. Correct for **incremental decisions**: "should we accept this extra batch job," "what does one more customer's traffic cost tonight." A unit served entirely off already-paid-for reserved capacity has near-zero *marginal* cash cost but a real *blended* cost.
- Commitment coverage (lesson 06) is the dial that sets the blended number: raising committed coverage from 50%→80% lowers $r_{\text{blend}}$ toward the committed rate but raises exposure to commitment waste at low utilization. Unit economics consumes that blended rate as an input.

### Direct vs fully-loaded loading

The waste ledger (idle GPUs, fragmentation, control-plane nodes, networking, storage, the util-lie gap from 05) is real spend that produces zero units directly. Two honest bases:

- **Direct unit cost** — $H_{\text{attr}}$ = only the GPU-hours the workload's pods actually held; $r$ = pure GPU rate. Excludes idle/fragmentation/overhead. Honest for **relative comparisons** ("is model A cheaper to serve than model B") and for engineering optimization targets, because it isolates the workload's own efficiency.
- **Fully-loaded unit cost** — allocate the entire fleet's cost (including idle, fragmentation, control plane, storage, networking) across the units actually produced. Mechanically: inflate either $H_{\text{attr}}$ by an overhead factor, or $r$ by a loading multiplier $L = \frac{\text{total fleet \$}}{\text{directly-attributed GPU \$}}$. Honest for **P&L, pricing, and gross margin**, because customers do not pay for the idle 30% — your other units must. If the fleet is 65% utilized, $L\approx 1/0.65 = 1.54$; a $2.00 direct $/1M tokens becomes ~$3.08 fully-loaded, and *that* is the number your pricing must clear.

Rule of thumb: **optimize on direct, price on fully-loaded, and never quote one as the other.** The gap between them *is* the idle/fragmentation tax, expressed per business unit — which is the most legible way to show an exec why utilization matters.

## Worked example

A serving namespace `llm-chat-prod` on an H100 fleet, one billing day.

**Infra side.** Attribution exporter reports 8 pods, each holding 1 H100 for the full 24 h → $H_{\text{attr}} = 8 \times 24 = 192$ GPU-h. Fleet commitment mix: 70% committed at \$1.90/GPU-h, 30% on-demand at \$3.20/GPU-h → $r_{\text{blend}} = 0.7(1.90)+0.3(3.20) = \$2.29$/GPU-h.

**App side.** vLLM `/metrics` delta over the day: 41.0B generation tokens (pricing on output only, stated). $U = 41{,}000$ (in millions).

**Direct $/1M tokens.**
$$
\frac{192 \times 2.29}{41{,}000} = \frac{439.7}{41{,}000} = \$0.01072/\text{1M} \approx \$10.72/\text{1B tokens}
$$

**Fully-loaded.** The cluster this namespace runs on is 62% utilized: the other 38% of GPU-hours are idle headroom, fragmentation (bin-packing 8×1 GPU pods onto nodes), plus a control-plane + fast-storage overhead measured at 9% of GPU spend. Loading multiplier $L = \frac{1}{0.62} \times 1.09 = 1.613 \times 1.09 = 1.758$.

$$
\$0.01072 \times 1.758 = \$0.01885/\text{1M} \approx \$18.85/\text{1B tokens}
$$

**The gap.** Direct says \$10.72/1B; fully-loaded says \$18.85/1B — a **76% premium**, and that delta is the idle/fragmentation/overhead tax expressed per token. If the product prices at \$15/1B output tokens, it looks fat on the direct basis and is **underwater on the loaded basis** — the fleet is subsidizing this service out of its idle capacity, which is not free. The fix is either raise price, raise utilization (shrink $L$ toward 1), or both. Note the util-lie (05) hides inside this: if those 8 GPUs run at 45% SM_ACTIVE, the *engineering* headroom is even larger than the 62% cluster-level figure suggests — you're paying loaded cost for tokens you could produce at a fraction of the GPU-hours with better batching.

## Practice

Feeds `../practice/gpu-cost-synthesis/README.md`:

1. Build a small calculator (script or notebook) that ingests three CSVs — attributed GPU-hours per namespace-day, a commitment-mix rate table, and app-emitted token deltas — and emits **direct** and **fully-loaded** $/1M tokens per namespace, plus the loading multiplier $L$ and the per-token idle tax (loaded − direct).
2. Extend it to a training dataset: compute $/run and $/successful-run from a runs table with a `status` column; report the success-rate multiplier and the dollars burned on failed jobs.
3. Add a marginal-vs-blended toggle: recompute the incremental unit cost of the top-decile-traffic hour served off spot top-up, and contrast with the blended number.
4. Write a 5-line memo stating which basis (direct / loaded, blended / marginal) you'd quote for: (a) a pricing decision, (b) a model-A-vs-B efficiency review, (c) accepting an overnight batch job.

## Self-check

- Why can you not compute a defensible $/1M-token number by dividing the monthly cloud GPU bill by monthly tokens? **Answer:** It fails the join on all three terms: no per-namespace attribution (numerator is the whole fleet, not the workload), it uses list/blended-wrong rate instead of the actual commitment-mix blended rate, and it silently either includes or excludes the idle/fragmentation/overhead tax without a stated basis — so you can't tell if it's direct or fully-loaded, and it's neither cleanly.
- A research fleet's $/run is \$4,000 and its job success rate is 55%. What is $/successful-run, and what does the gap represent? **Answer:** $4{,}000 / 0.55 = \$7{,}273$ per successful run. The \$3,273 gap is the cost of failed/crashed/diverged jobs amortized onto usable results — it prices iteration inefficiency and checkpoint/reliability hygiene, which naive $/run hides.
- Your fleet is 62% utilized with 9% non-GPU overhead. When do you quote the direct $/1M tokens vs the fully-loaded one, and what does their ratio equal? **Answer:** Quote direct (excludes idle/overhead) for relative model-vs-model efficiency comparisons and engineering optimization targets; quote fully-loaded (loading multiplier $L = (1/0.62)\times1.09 \approx 1.76$) for pricing, P&L, and gross-margin decisions, because customers don't pay for idle capacity so the produced units must absorb it. Their ratio *is* $L$ — the idle/fragmentation/overhead tax per unit.

## Resources

- vLLM production metrics reference (Prometheus counters incl. prompt/generation token totals): https://docs.vllm.ai/en/latest/serving/metrics.html
- FinOps Foundation — Unit Economics capability: https://www.finops.org/framework/capabilities/unit-economics/
- NVIDIA DCGM-Exporter (source of per-GPU utilization/attribution metrics): https://github.com/NVIDIA/dcgm-exporter
- OpenCost / Kubecost GPU allocation model: https://www.opencost.io/docs/specification
- Baseten — how they think about $/token and inference unit economics: https://www.baseten.co/blog/
