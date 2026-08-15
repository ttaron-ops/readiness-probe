---
lesson: 05
title: "Unit economics: joining infra dollars to application counters"
module: 11
concept: "infra-$ to business-unit join"
status: not-started
est_time: "6 hrs"
prev: "04-fragmentation-cost.md"
next: "06-commitment-strategy.md"
artifacts: ["a $/1M-token calculator that joins attributed GPU-hours × blended rate ÷ app-emitted token counts, emitting both direct and fully-loaded numbers"]
sources: 11
---

# Unit economics: joining infra dollars to application counters

[💰 11 — GPU cost and unit economics](../README.md) · ← [04 Fragmentation: unschedulable GPUs](04-fragmentation-cost.md) · → [06 Commitment & procurement strategy](06-commitment-strategy.md)

## Where this fits

Lessons 01-04 built the machinery this lesson consumes. 01 established the four attribution regimes and what's provably unattributable per GPU-hour; 02 split the output into two ledgers — allocated (what's billed) vs utilized (what's actually used); 03 taught you to detect idle GPUs without false-positives; 04 priced the fragmentation tax, capacity that is free but unschedulable. Together those four lessons hand you one number, $H_{\text{attr}}$ (attributed GPU-hours), plus a waste ledger sitting beside it. This lesson is where that infra number stops being an infra artifact and becomes a business number: the general join that turns $H_{\text{attr}} \times r$ into $/token, $/run, $/experiment, or any other unit your product cares about. You already know two special cases of this join — cost-per-token (07) and cost-per-run/MFU (08) — as standalone metrics; here they collapse into one formula, and the new work is the join itself, not either endpoint. It also sets up lesson 06, which is where the blended rate $r_{\text{blend}}$ this lesson treats as an input actually comes from.

## Why this matters

Every prior cost lesson gave you one half of a fraction. Lesson 07 taught cost-per-token as a serving metric; lesson 08 taught cost-per-run and MFU as a training metric. This lesson is the general operator that produces *both* of those and any other unit your business cares about, from the same two ledgers: an infra ledger denominated in **$/GPU-hour** and an application ledger denominated in **business units** (tokens, requests, runs, experiments, customers). Unit economics is the join between them. If you cannot state your product's cost in $/1M tokens or $/training-run *and defend the denominator and the loading*, you do not actually know your gross margin — you know your cloud bill, which is a different and less useful number.

This is the single most-probed competency in a GPU-platform or inference-infra interview, because it is where FinOps meets the product P&L. When an inference-provider (Together, Fireworks, Baseten, Anyscale) or a frontier lab's serving team asks "what does a million tokens cost us," the naive answer — divide the monthly GPU bill by monthly tokens — fails three follow-ups: it ignores per-namespace attribution, it uses list rate instead of blended committed rate, and it silently drops the idle/fragmentation tax onto the floor. The senior answer walks the join key explicitly and shows where the money leaks.

The reason this matters beyond interviews: unit cost is the number that decides pricing, that decides build-vs-rent (lesson 06, module 10), and that decides which workloads get killed in a capacity crunch. A unit cost computed on the *direct* basis makes a service look profitable while the fleet as a whole bleeds; a unit cost computed *fully-loaded* tells you the truth but can make a healthy service look underwater if you misallocate the waste. Knowing which number is honest in which decision is the actual skill.

## What's new here (calibration)

- **You already know** cost-per-token (07) and cost-per-run + MFU (08) as standalone metrics. **Skip** re-deriving them. Here they are two instances of one formula, and the point is the *general join*, not either endpoint.
- **You already know** per-pod GPU attribution (04/05) and the SM_ACTIVE-vs-GPU_UTIL util-lie (05). **Skip** the mechanism. Here it becomes the *numerator sourcing* — the attributed GPU-hours are the input, and the util-lie is why a "utilized" fleet still has a fully-loaded cost far above its direct cost.
- **New angle 1:** treating $/token and $/run as the *same formula* with different denominators, so you can synthesize a *new* unit on demand — including **$/successful-experiment**, which divides by successful runs and thereby prices your failure rate.
- **New angle 2:** **blended vs marginal rate** as a deliberate choice in the numerator's rate term, and how commitment coverage (lesson 06) moves the blended number.
- **New angle 3:** **direct vs fully-loaded** unit cost — whether the idle/fragmentation/control-plane/storage waste ledger (03/04) gets allocated across units, and when each basis is the honest one.

## Core concepts

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

## Perspectives

**Product/pricing perspective.** A price sheet is a promise made in advance, against demand you don't fully control yet. The number that promise must clear is the fully-loaded unit cost — because every customer's traffic implicitly shares the fleet's idle capacity, fragmentation tax, and control-plane overhead, whether or not that customer ever sees a GPU sit idle. Price below fully-loaded and you are subsidizing that customer out of margin on every other customer; price it and you have a defensible floor. This is why pricing teams push back hard when an engineering team quotes them a direct number — it looks better, but it's the wrong basis for a commitment made across an entire fleet's economics.

**Engineering-optimization perspective.** An efficiency team should never be judged against the fully-loaded number, because most of what inflates it — fleet-wide idle time, scheduler fragmentation, someone else's abandoned namespace — is outside that team's control. The direct number isolates what the team actually owns: how many tokens or samples it extracts per GPU-hour its own pods hold. Judge a serving team on direct $/1M tokens and a genuine batching or kernel win shows up immediately; judge them on fully-loaded and a win can be swamped by unrelated fleet-wide idle noise, which teaches the wrong lesson about what to fix.

**Finance/FP&A perspective.** Blended vs marginal is not a rounding choice, it's a real accounting decision with consequences that show up in different statements. Blended rate feeds gross margin and steady-state pricing — it's what the income statement should reflect. Marginal rate feeds accept/reject decisions on incremental demand — "does this new customer's traffic clear its own cost," independent of sunk commitment. Using marginal rate for pricing understates true cost (you'll price as if excess reserved capacity is free forever); using blended rate for an accept/reject call on spare capacity overstates the cost of saying yes to work you'd otherwise eat as waste. FP&A's job is knowing which one belongs in which model.

**Research-velocity perspective.** $/successful-experiment is unusual among unit-cost metrics because it prices something that looks like an engineering-quality problem — job success rate — as a hard dollar figure. A research org with sloppy checkpointing, no preemption handling, or noisy hardware looks fine on raw $/run and terrible on $/successful-run; the gap between the two numbers is the exact dollar value of investing in reliability. It turns "we should improve our training infrastructure" from a vague aspiration into a line item with a specific ROI, which is why it's the number research-platform teams increasingly report internally.

## Real-world use cases

- **Character.AI — "Optimizing AI Inference at Character.AI"** ([blog.character.ai/optimizing-ai-inference-at-character-ai-2](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/)) and its follow-up, **"...Part Deux"** ([blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/)) — reduced serving cost per query by a factor of **33× versus their late-2022 baseline**, and state it would cost **13.5× more** to serve the same traffic on leading commercial APIs. Achieved via native int8 training plus custom int8 inference kernels, and KV-cache size reduction of **more than 20×** through Multi-Query Attention, inter-turn host-memory caching (95% cache hit rate), and cross-layer cache sharing. This is the best lead use case in the module: a concrete, quantified, multi-year unit-economics story that shows the direct $/token number *should* move an order of magnitude as engineering matures — exactly the trajectory the naive "just divide the bill" number can't show you.
- **Baseten — "Cost-efficient high-performance Qwen3-TTS"** ([baseten.co/blog/cost-efficient-high-performance-qwen3-tts](https://www.baseten.co/blog/cost-efficient-high-performance-qwen3-tts/)) — reports up to **2.5× better throughput per dollar** running on NVIDIA Blackwell versus Hopper for this workload, part of an industry-wide move that has cut serving costs for some deployments by roughly 90%. Shows the numerator side of the unit-cost formula ($r_{\text{blend}}$, effectively — cost per useful GPU-hour) moving with a hardware generation, independent of any application-level batching work.
- **Together AI — "Foundational research powering efficient inference at scale"** ([together.ai/blog/foundational-research-powering-efficient-inference-at-scale](https://www.together.ai/blog/foundational-research-powering-efficient-inference-at-scale)) — describes production-tested inference optimizations (adaptive speculative decoding, custom kernels, NVFP4 quantization across 72-GPU Blackwell meshes) that increase tokens produced per GPU-hour, i.e. move $U$ in the unit-cost formula without touching $H_{\text{attr}}$ or $r$. A clean illustration that "cut $/token" and "increase throughput per GPU-hour" are the same lever seen from two sides of the formula.
- **NVIDIA — "Leading Inference Providers Cut AI Costs by up to 10x With Open Source Models on NVIDIA Blackwell"** ([blogs.nvidia.com/blog/inference-open-source-models-blackwell-reduce-cost-per-token](https://blogs.nvidia.com/blog/inference-open-source-models-blackwell-reduce-cost-per-token)) — names Baseten, DeepInfra, Fireworks AI, and Together AI as providers who cut cost-per-token by up to 10× moving from Hopper to Blackwell, and cites Sully.ai's healthcare-inference deployment cutting costs 90% by switching to open-source models on Baseten's Blackwell stack. An industry-wide, multi-vendor datapoint that the direct unit-cost number is not static — it is actively being driven down by a hardware-generation transition on top of the software optimizations above.

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

## Common pitfalls

1. **Dividing the monthly bill by monthly tokens.** The single most common mistake, and the one that fails every follow-up question. It skips per-namespace attribution, uses whatever blended-or-not rate happens to be on the invoice, and mixes direct and loaded costs with no stated basis. The Character.AI story above is the antidote to internalize: a well-run inference team's real $/token *should* move by an order of magnitude as they optimize — a single monthly-bill ratio can't show you that trajectory or tell you which lever moved it.
2. **Using list/on-demand rate in $r_{\text{blend}}$ when the fleet runs mostly on commitment.** This overstates cost by roughly 30-60% for a heavily-committed fleet. It isn't just an estimation error either — Kubecost's own accuracy research finds that for organizations running committed-use discounts or Reserved Instances, list pricing misrepresents actual spend by **30-50%** ([Apptio/Kubecost, "Enhancing Cost Accuracy"](https://www.apptio.com/blog/enhancing-cost-accuracy/)), which independently corroborates the size of this error in a widely-used cost tool's default behavior.
3. **Quoting the wrong basis for the audience.** Quote *direct* in a pricing conversation and you'll set a price that's underwater once idle/overhead lands on the P&L — finance gets burned six months later reconciling margin. Quote *fully-loaded* in an engineering efficiency review and a genuine batching win gets buried under fleet-wide idle noise the engineer never controlled — the team gets burned by being judged on someone else's waste. State the basis every time you say a number out loud.
4. **Ignoring failure rate in training unit economics.** $/run without $/successful-run understates true iteration cost by exactly $1/\text{success rate}$. A team quoting only $/run to justify a training budget is hiding the dollars spent on crashed, OOM'd, and diverged jobs — often the single largest lever available for cutting real spend.
5. **Mismatched windows or timezones between the infra-hours source and the app-counter source.** GPU-hours pulled from a Kubecost billing-day boundary (UTC) joined against tokens from an app dashboard on local time silently corrupts the ratio — a few hours of skew can shift $U$ or $H_{\text{attr}}$ by double digits of percent on a bursty workload. This is the easiest mistake to make when actually building the calculator in Practice #1, and the hardest to notice after the fact because the resulting number still looks plausible.

## Self-check

- Why can you not compute a defensible $/1M-token number by dividing the monthly cloud GPU bill by monthly tokens? **Answer:** It fails the join on all three terms: no per-namespace attribution (numerator is the whole fleet, not the workload), it uses list/blended-wrong rate instead of the actual commitment-mix blended rate, and it silently either includes or excludes the idle/fragmentation/overhead tax without a stated basis — so you can't tell if it's direct or fully-loaded, and it's neither cleanly.
- A research fleet's $/run is \$4,000 and its job success rate is 55%. What is $/successful-run, and what does the gap represent? **Answer:** $4{,}000 / 0.55 = \$7{,}273$ per successful run. The \$3,273 gap is the cost of failed/crashed/diverged jobs amortized onto usable results — it prices iteration inefficiency and checkpoint/reliability hygiene, which naive $/run hides.
- Your fleet is 62% utilized with 9% non-GPU overhead. When do you quote the direct $/1M tokens vs the fully-loaded one, and what does their ratio equal? **Answer:** Quote direct (excludes idle/overhead) for relative model-vs-model efficiency comparisons and engineering optimization targets; quote fully-loaded (loading multiplier $L = (1/0.62)\times1.09 \approx 1.76$) for pricing, P&L, and gross-margin decisions, because customers don't pay for idle capacity so the produced units must absorb it. Their ratio *is* $L$ — the idle/fragmentation/overhead tax per unit.
- A customer's traffic runs entirely on already-paid-for reserved capacity that would otherwise sit idle. Should you evaluate whether to accept it using the blended rate or the marginal rate, and why might using the wrong one mislead you? **Answer:** Use the marginal rate for the accept/reject decision — the true incremental cash cost of that traffic is close to zero since the capacity is sunk and otherwise wasted. Using the blended rate would overstate the cost of saying yes and could cause you to reject revenue-positive traffic. Conversely, using marginal rate to set a *steady-state price* would understate true cost, since it ignores that the reserved capacity isn't free in aggregate — it must be paid for by someone's blended rate eventually.

## Connections & what's next

This lesson is the hinge of the module: lessons 01-04 feed it $H_{\text{attr}}$ and the waste ledger (attribution, the two ledgers, idle detection, fragmentation cost), and it generalizes the two unit metrics you already knew as standalone tools — cost-per-token (07) and cost-per-run/MFU (08) — into one join. The one input this lesson treats as given, $r_{\text{blend}}$, is not free-floating: it's a function of how the fleet is procured. Lesson 06 opens that box — commitment coverage, spot eligibility, and the build-vs-buy-vs-rent decision that actually sets the blended rate this lesson consumes.

## References & further reading

- **Primary sources**
  - vLLM — Production Metrics reference (Prometheus counters incl. prompt/generation token totals): [docs.vllm.ai/en/stable/design/metrics](https://docs.vllm.ai/en/stable/design/metrics/) — read for the exact counter names and semantics behind the $U$ term.
  - FinOps Foundation — Unit Economics capability: [finops.org/framework/capabilities/unit-economics](https://www.finops.org/framework/capabilities/unit-economics/) — read for the industry-standard framing of unit cost as a FinOps capability, not just a spreadsheet formula.
  - NVIDIA DCGM-Exporter (source of per-GPU utilization/attribution metrics): [github.com/NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — read for where $H_{\text{attr}}$'s underlying signal actually comes from.
  - OpenCost Specification — GPU allocation model: [opencost.io/docs/specification](https://opencost.io/docs/specification/) — read for how a widely-deployed OSS tool defines allocated cost, and where it diverges from utilization.
  - Apptio/Kubecost — "Enhancing Cost Accuracy": [apptio.com/blog/enhancing-cost-accuracy](https://www.apptio.com/blog/enhancing-cost-accuracy/) — read for the vendor's own data on how far list pricing drifts from actual committed-use spend (grounds pitfall #2).
- **Real-world engineering blogs**
  - Character.AI — "Optimizing AI Inference at Character.AI": [blog.character.ai/optimizing-ai-inference-at-character-ai-2](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) — 33× cost reduction since 2022; int8 kernels and MQA-driven KV-cache cuts, what it shows: direct unit cost as a moving, engineerable target.
  - Character.AI — "Optimizing AI Inference at Character.AI (Part Deux)": [blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — follow-up optimizations continuing the same cost trajectory.
  - Baseten — "Cost-efficient high-performance Qwen3-TTS": [baseten.co/blog/cost-efficient-high-performance-qwen3-tts](https://www.baseten.co/blog/cost-efficient-high-performance-qwen3-tts/) — 2.5× throughput-per-dollar on Blackwell vs Hopper, what it shows: the rate term moving with hardware generation.
  - Together AI — "Foundational research powering efficient inference at scale": [together.ai/blog/foundational-research-powering-efficient-inference-at-scale](https://www.together.ai/blog/foundational-research-powering-efficient-inference-at-scale) — production speculative decoding/quantization, what it shows: the $U$ term moving via software.
  - NVIDIA — "Leading Inference Providers Cut AI Costs by up to 10x With Open Source Models on NVIDIA Blackwell": [blogs.nvidia.com/blog/inference-open-source-models-blackwell-reduce-cost-per-token](https://blogs.nvidia.com/blog/inference-open-source-models-blackwell-reduce-cost-per-token) — industry-wide cost-per-token datapoint across four inference providers.
- **Deeper dives**
  - a16z — "Welcome to LLMflation: LLM inference cost is going down fast": [a16z.com/llmflation-llm-inference-cost](https://a16z.com/llmflation-llm-inference-cost/) — a longer-horizon look at the ~10×/year decline in cost for equivalent-quality inference, useful context for why any unit-cost snapshot ages fast.
