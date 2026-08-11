---
lesson: 04
title: "Flagship blog + demo: 'Your GPU dashboard is lying'"
module: 12
concept: "contrarian proof + narrated demo"
status: not-started
est_time: "5 hrs"
prev: "03-portfolio-writeup.md"
next: "05-system-design-drills.md"
artifacts: ["a published flagship blog post ('Your GPU dashboard is lying'), two secondary post outlines (cost-per-1M-tokens, survive-a-failure), and a scripted 5-minute narrated demo of the value chain"]
sources: 7
---

# Flagship blog + demo: "Your GPU dashboard is lying"

[🎓 12 — Capstone & interview preparation](../README.md)

## Where this fits

Lesson 03 established the *written* portfolio: a repo README with a value-chain diagram, an RFC-style design doc that reasons in Goals/Non-goals/Alternatives/Risks, and a brag-doc of quantified outcomes. That's evidence for a reader who already clicked into your repo. What 04 adds is the **public contrarian proof and the narrated demo** — the two artifacts built for reach rather than depth, designed to pull a stranger in and hand them your framing in one sentence before they've read a single line of your design doc. The flagship post and the demo are what go at the very top of the README from lesson 03, and they are the first thing that leads every application from here forward.

## Why this matters

A design doc proves you can reason; a flagship blog post proves you *know a thing most people in the room don't*. "Your GPU dashboard is lying" (artifact 05) is the single strongest public proof in your portfolio because it is **contrarian, quotable, and load-bearing on the exact insight that separates GPU-infra people from generic platform engineers**: that `nvidia-smi` "GPU utilization" tells you a kernel was scheduled on the device, not that the device did useful FLOPs. A generic SRE reads 95% util and reports the fleet is busy. You read 95% util / 31% MFU and report the fleet is *idle inside the kernels* — burning money at full price for a third of the work. That gap is the whole thesis, and it is the thing a GPU-fleet hiring manager is scanning your writing for.

The reason a blog post beats a repo here is reach and framing. A repo requires the reader to reconstruct the insight; a post hands it to them in one sentence they can repeat to their own team. "Your GPU dashboard is lying" is repeatable — a hiring manager who reads it can say it in their next standup, and now your framing is in their head with your name attached. That is how a post does distribution work a resume cannot: a repo is discovered only by someone already evaluating you, while a post can be discovered by someone who has never heard of you and decides, on the strength of one argument, to go find your repo. The title is contrarian on purpose: it makes a claim strong enough to be wrong, then earns it with the SM_ACTIVE-vs-GPU_UTIL measurement. Honesty and reproducibility are what keep it from reading as clickbait — the reader must be able to run your measurement and see 95/31 for themselves.

This is not a novel angle you're inventing from scratch, and that's a feature, not a weakness — it means the genre is proven. GPU-infra vendors have published essentially this same argument: Trainy's engineering blog makes the case that 100% GPU utilization is trivially achievable through memory-bound operations that do zero useful compute, and MFU as a metric exists precisely because the industry needed a number that couldn't be gamed that way — the PaLM paper (Chowdhery et al., 2022) is where the term "Model FLOPs Utilization" was coined, and neoclouds like Lambda publish their own MFU reasoning as a matter of course. Writing your version well means you're not inventing a genre, you're executing a known-effective one with your own fleet's numbers and your own voice.

The demo does the other half. Some managers read; some watch. A five-minute narrated demo that walks the *value chain* — attribution to scheduling to observability to economics — and ends on the dashboard telling the truth is a portable proof you can drop into any application or send after a call. Together the post and the demo are your lead public surface: the two links that go at the very top of the README from lesson 03.

## What's new here

- **You already know** how to write clearly and how to record your screen. **Skip** the mechanics of blogging and screencasting. The new thing is *structuring for a hiring-manager reader* and *narrating the value chain instead of the code*.
- **You already know** the SM_ACTIVE vs GPU_UTIL vs MFU distinction cold (module 05). **Skip** re-deriving it. Here it becomes the *spine of a public argument* — measurement → reveal → fix → takeaway.
- **New angle 1:** the **flagship-post arc** — hook (the lie) → the measurement → the reveal (95% util / 31% MFU) → the fix (what to actually alert on) → the takeaway.
- **New angle 2:** the **flagship + two secondaries** shape — one contrarian lead, two supporting posts (cost-per-1M-tokens from 07, survive-a-failure from 08) that show range without diluting the lead.
- **New angle 3:** the **narrate-the-chain-not-the-code** demo script, and **publishable-by-default distribution** with employer specifics scrubbed.
- **New angle 4:** knowing this is a *proven public genre* — real GPU-infra vendors and the paper that coined MFU already made versions of this argument — and using that as calibration for how sharp and reproducible your own version needs to be to hold up next to theirs.

## Core concepts

### Why this is the flagship

Three properties, all required:

- **Contrarian** — "your dashboard is lying" contradicts the number everyone trusts. A strong claim earns attention a "how I built X" post never will.
- **Quotable** — the thesis compresses to one sentence: *`nvidia-smi` util means "a kernel was running," not "useful FLOPs were produced."* A reader can repeat it.
- **Load-bearing** — it proves the util-vs-MFU/goodput insight, which is the specific competency a GPU-fleet operator screens for and a generic platform engineer lacks.

### The measurement (the honest core)

The post lives or dies on being reproducible. Lay the three signals side by side (ref module 05):

| Signal | Source | What it actually means | Failure it hides |
|---|---|---|---|
| `GPU_UTIL` (DCGM_FI_DEV_GPU_UTIL) | nvidia-smi / DCGM | % of time ≥1 kernel was resident on the device | A tiny kernel pinning the GPU at 100% while doing almost no work |
| `SM_ACTIVE` (DCGM_FI_PROF_SM_ACTIVE) | DCGM profiling | fraction of SMs actually issuing over the window | Low occupancy — kernels running but most of the chip idle |
| **MFU / goodput** | achieved FLOPs ÷ peak FLOPs (derived) | fraction of the GPU's theoretical throughput you actually got | Everything above — the real efficiency number |

The reveal number: a fleet reading **95% GPU_UTIL** can be running at **~31% MFU**. Util is green; two-thirds of the money is on the floor. That contrast is the post's climax.

MFU itself is not a metric you're inventing — it's the term coined in the PaLM paper (Chowdhery et al., 2022) specifically to give the field a throughput number that can't be gamed by a memory-bound kernel pinning the utilization reading at 100%. Citing that origin in the post (even in a single footnote) does two things: it signals you know the metric's provenance rather than treating it as a buzzword, and it borrows credibility from a primary source the reader may already trust.

### The flagship-post arc

1. **Hook — the lie.** Open on the dashboard everyone trusts: 95% GPU utilization, all green. Claim it's lying.
2. **The measurement.** Introduce SM_ACTIVE vs GPU_UTIL vs MFU. Show how to measure each (DCGM profiling metrics; MFU from achieved-vs-peak FLOPs). Keep it reproducible — the reader can run this.
3. **The reveal.** 95% util / 31% MFU on the same fleet. Name the gap: you are paying full $/GPU-hr for a third of the work.
4. **The fix.** What to *actually* alert on — SM_ACTIVE and goodput/MFU thresholds, not GPU_UTIL. What a truthful dashboard shows. Where batching, occupancy, and data-loading stalls hide.
5. **The takeaway.** One repeatable line the reader leaves with: util is a liveness check, not an efficiency metric; measure goodput or you're flying blind on the most expensive hardware you own.

Keep it **honest and reproducible** throughout — the credibility of a contrarian post is entirely in whether the reader can verify the reveal. That includes resisting the urge to make the reveal numbers *too* clean: real measurements have noise, and a suspiciously round 95%/31% with no caveat reads to an experienced reader as smoothed-over or fabricated rather than measured. State the sample window, the workload, and any variance you saw.

### The two secondary posts

One flagship, two supporting posts that demonstrate range without competing for the lead:

- **Cost-per-million-tokens (artifact 07)** — the economics proof. How you join attributed GPU-hours × blended rate ÷ token counts, and why direct vs fully-loaded is the honest distinction. Shows you connect the metal to the P&L.
- **Survive-a-failure (artifact 08)** — the operations proof. What happens when a GPU/node/job dies mid-run, and how the platform degrades gracefully. Shows you've run fleets in anger, not just measured them.

Sequence them *after* the flagship in the README so the contrarian lead pulls readers in and the secondaries hold the ones who want depth.

### The 5-minute narrated demo

**Narrate the value chain, not the code.** Nobody hires off watching you scroll a file. They hire off watching you *reason about a system*. Script structure:

| Minute | Beat | What's on screen |
|---|---|---|
| 0:00-0:45 | The frame | The whole-fleet architecture diagram (lesson 03). Name the value chain in one breath. |
| 0:45-2:00 | The lie | The dashboard at 95% util, all green. State the thesis. |
| 2:00-3:30 | The truth | Switch to SM_ACTIVE / MFU panels. Show the 31%. This is the payoff shot — *the dashboard telling the truth.* |
| 3:30-4:30 | The chain | Trace one GPU-hour through attribution → cost-per-token → what you'd alert on. |
| 4:30-5:00 | The takeaway | Restate the quotable line, point to the repo/post. |

Record once you can narrate it without reading. The demo is a proof of *how you think*, so the voiceover carries it — the screen is just evidence. The single most common failure mode here is narrating code or repo structure instead of the value chain: a reviewer watching you scroll through YAML or explain a file layout learns nothing about how you reason, because file layout isn't a decision, it's plumbing. Every minute of the five should be advancing an argument, not a tour.

### Distribution

Where GPU-infra hiring managers actually read: a **personal blog** (owned surface, canonical link), **GitHub** (repo README links to the post), and the **relevant technical communities** where fleet operators hang out. Make everything **publishable-by-default**: write it so it can go public, and **scrub employer specifics** — real fleet numbers become ratios or normalized figures, internal service names become generic tiers, no confidential $/GPU-hr or real fleet-size numbers. The reveal (95/31) works fine on a synthetic-but-realistic fleet; you never need real employer data to make the point, and using actual employer $/GPU-hr or fleet-size figures is a confidentiality risk you don't need to take for a post that works just as well scrubbed.

## Perspectives

**The reader-as-hiring-manager view.** The reader you're optimizing for is scanning for one thing: a line worth repeating to their own team. A hiring manager doesn't remember your prose, your formatting, or even most of your argument — they remember the one sentence they said out loud in their next standup. That means the whole post is in service of manufacturing that one line, and every paragraph that doesn't sharpen it is a paragraph diluting the thing that actually gets you routed to a loop.

**The reader-as-skeptic view.** A contrarian claim invites scrutiny by design — you are telling an experienced reader that a number they trust is wrong, and their first instinct is to look for the seam. What keeps the post credible under that scrutiny is not cleverness, it's reproducibility: a stated measurement window, a real (if scrubbed) workload, a query the reader could run themselves. Cleverness earns a skim; reproducibility earns trust, and trust is what turns a reader into someone forwarding your post to a colleague.

**The finance/leadership view.** Whether to frame a metric flatteringly or truthfully is not a purely technical choice — it's a political one, and this post is a public demonstration of which side you land on. A dashboard reading 95% util looks great in a leadership review; a dashboard reading 31% MFU invites uncomfortable questions about spend. Publicly choosing to surface the harder, truer number — and explaining why it matters to the P&L, not just to engineers — is itself part of the signal: it shows you'll tell leadership the number they need, not the one that's easiest to present.

**The distribution-economics view.** A repo and a post do fundamentally different economic work. A repo has to be found by someone already looking for you; a post can be found by someone who has never heard of you and decides, on the strength of one argument, to go looking. That asymmetry is why the flagship post — not the repo — is what belongs at the top of every cold application: it's the artifact capable of doing reach work, while the repo and design doc do depth work once the reader has already arrived.

## Real-world use cases

- **Trainy, "GPU Utilization is a Misleading Metric"** — https://www.trainy.ai/blog/gpu-utilization-misleading-metric — what it shows: a real GPU-infra vendor publishing essentially this flagship post's argument already — 100% utilization achievable via memory-only operations doing zero useful compute — proof the genre works and a calibration bar for how sharp your own version needs to be.
- **MarkTechPost, "Why GPU Utilization Falls Short: Understanding SM Efficiency for Better LLM Performance"** — https://www.marktechpost.com/2024/09/03/why-gpu-utilization-falls-short-understanding-streaming-multiprocessor-sm-efficiency-for-better-llm-performance/ — what it shows: an independent technical outlet covering the same SM-efficiency argument, evidence that this angle has enough general interest to travel beyond a single vendor's blog.
- **The PaLM paper (Chowdhery et al., 2022)** — https://arxiv.org/pdf/2204.02311 — what it shows: the primary source that coined "Model FLOPs Utilization" — citing it grounds your post's central metric in the literature instead of treating MFU as an invented buzzword.
- **Lambda's MFU whitepaper** — https://lambda.ai/hubfs/4.%20Resources/White%20Papers/Lambda%20MFU.pdf — what it shows: a neocloud on the learner's own target-employer list publishing exactly this reasoning to customers, evidence that the argument you're making is the same one GPU-fleet operators use to sell and to hire.

## Worked example

Post opens: *"Our GPU dashboard said 95% utilization. It was lying — we were running at 31% MFU. Here's how to catch it."* Two sentences, the contrarian claim staked and quantified. Section two puts GPU_UTIL, SM_ACTIVE, and MFU in a table, notes MFU's origin in the PaLM paper's need for a metric that can't be gamed by memory-bound kernels, and shows the DCGM profiling query for SM_ACTIVE, so the reader can run it. Section three is the reveal: same fleet, over a stated one-week window, `GPU_UTIL` flat around 95%, `SM_ACTIVE` averaging ~40%, derived MFU landing at 29-33% depending on the workload mix — with the one-line cause (low occupancy + data-loading stalls between kernels) and an honest note that the number moves with batch size rather than presenting one suspiciously tidy figure. Section four flips it to action: alert on `DCGM_FI_PROF_SM_ACTIVE < 0.5` and track goodput, *not* `GPU_UTIL`; here's what the truthful Grafana panel looks like. Takeaway: *"GPU_UTIL is a liveness check. If it's your efficiency metric, you're paying full price for a third of the work and your dashboard is hiding it."*

The demo: the reviewer watches five minutes. Opens on the architecture diagram, one sentence of value chain. Cuts to the green 95% dashboard — "this is what everyone trusts." Cuts to the SM_ACTIVE/MFU panel showing ~31% — the truth shot. Traces one H100-hour: attributed to a namespace, priced at cost-per-1M-tokens, and the alert that *should* have fired. Closes on the quotable line and the repo link. Not one frame is spent scrolling code or explaining repo layout — every beat advances the same argument the post makes. The reviewer now knows, in five minutes and one post, that you understand GPUs at a level a generic platform engineer does not — which is the entire hiring thesis for a GPU-fleet role.

## Practice

Feeds `../practice/gpu-platform-capstone/README.md`:

1. **Draft the flagship post outline** end to end using the five-beat arc (hook → measurement → reveal → fix → takeaway). Write the measurement table and the actual reveal numbers from your artifact-05 data, with a stated sample window rather than a single suspiciously round figure. Make the SM_ACTIVE measurement reproducible — include the DCGM query. Add one line citing MFU's origin (PaLM paper).
2. **Outline the two secondaries** (cost-per-1M-tokens from 07, survive-a-failure from 08) to one-paragraph-per-section depth. Sequence all three in the README after the flagship.
3. **Write the 5-minute demo script** on the minute-by-minute grid: the frame, the lie, the truth, the chain trace, the takeaway. Record a first pass; iterate until you can narrate without reading, and check that no beat drifts into narrating code or file structure.
4. **Scrub for publish:** replace any real employer numbers with ratios/normalized figures and generic tier names. Confirm the reveal still lands on synthetic-but-realistic data.
5. **Read one comparable published piece** (Trainy's post or the Lambda MFU whitepaper) before your final pass, and note one thing they do that your draft doesn't yet.

## Common pitfalls

- **Making the reveal look too clean.** A round 95%/31% presented with no variance, no sample window, and no workload description reads as fabricated to an experienced reader — real measurements have noise. State the window, the workload, and the range you actually observed.
- **Burying the fix beneath the reveal.** If the post ends on "look how bad the number is" without a clear, actionable "here's what to alert on instead," it reads as complaining rather than engineering. The fix section is not optional padding — it's the part that proves you solved the problem, not just found it.
- **Using real employer $/GPU-hr or fleet-size numbers.** Beyond the confidentiality risk, it's unnecessary — the argument holds identically on scrubbed, normalized figures, so there's no upside to the exposure.
- **Narrating code or repo structure in the demo instead of the value chain.** This is the single most common failure mode: a reviewer watching you explain file layout or scroll YAML learns nothing about how you reason about the system. Every minute of the demo should advance the argument, not tour the codebase.
- **Treating MFU as a buzzword instead of a real metric with a real origin.** Using "MFU" without knowing (or citing) that it comes from the PaLM paper's need for a gaming-resistant efficiency number is a tell that you learned the term secondhand rather than understanding why it exists.

## Self-check

- Why is "Your GPU dashboard is lying" a stronger flagship than a "how I built my GPU cost exporter" post? **Answer:** It's contrarian (contradicts the trusted 95% util number), quotable (compresses to one sentence a reader can repeat), and load-bearing on the util-vs-MFU/goodput insight that specifically separates GPU-infra people from generic platform engineers. A "how I built X" post proves you can build; the flagship proves you know something the reader's own team probably doesn't — which is what a GPU-fleet manager screens for.
- What exactly does `nvidia-smi` GPU utilization measure, and what should you alert on instead? **Answer:** GPU_UTIL measures the fraction of time at least one kernel was resident on the device — "a kernel was running," a liveness signal — not whether useful FLOPs were produced. A fleet can read 95% util at ~31% MFU. Alert on SM_ACTIVE (are the SMs actually issuing) and goodput/MFU (achieved vs peak FLOPs) instead, because those catch low-occupancy and stall waste that GPU_UTIL hides.
- The demo is five minutes. Should you spend them walking the code, and what's the payoff shot? **Answer:** No — narrate the value chain (attribution → scheduling → observability → economics), not the code; managers hire off how you reason about a system, not scrolling files. The payoff shot is the cut from the green 95%-util dashboard to the SM_ACTIVE/MFU panel showing 31% — the dashboard telling the truth — which makes the flagship thesis visible in one frame.
- Where does the term "MFU" come from, and why does citing that origin strengthen the post? **Answer:** MFU (Model FLOPs Utilization) was coined in the PaLM paper (Chowdhery et al., 2022) specifically as a throughput metric that can't be gamed by a memory-bound kernel pinning GPU_UTIL at 100%. Citing it shows you understand the metric's provenance rather than using it as a buzzword, and borrows credibility from a primary source the reader may already know.
- Why should the reveal numbers (95%/31%) include a stated sample window and variance rather than being presented as one clean figure? **Answer:** Real measurements have noise; an implausibly tidy figure with no window or workload context reads as smoothed-over or fabricated to an experienced reader. Stating the sample window, workload, and observed range is what keeps the reveal reproducible and credible rather than merely dramatic.

## Connections & what's next

This lesson is the public-facing counterpart to lesson 03's written portfolio: the same Risks-section thinking that produced the time-slicing attribution hole in the design doc is what produces the util-vs-MFU reveal here, just aimed at a reach-oriented audience instead of a design-review one, and the same architecture diagram from 03 opens both the README and the demo's first 45 seconds. It also arms the rest of the module directly — the flagship post's five-beat arc (hook → measurement → reveal → fix → takeaway) is the same shape lesson 06's debugging drills and lesson 07's artifact narration ask you to reproduce out loud, under time pressure, without slides. Next: **lesson 05** moves from published artifacts to live system-design drills — the same volunteered numbers, tradeoffs, and failure honesty this post demonstrates in writing now have to hold up in real time, opening a design prompt cold.

## References & further reading

**Primary sources**
- Chowdhery et al., "PaLM: Scaling Language Modeling with Pathways" (origin of the MFU metric): https://arxiv.org/pdf/2204.02311
- NVIDIA DCGM — profiling metrics (SM_ACTIVE and friends): https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html

**Real-world engineering blogs**
- Trainy — "GPU Utilization is a Misleading Metric": https://www.trainy.ai/blog/gpu-utilization-misleading-metric
- Lambda — MFU whitepaper: https://lambda.ai/hubfs/4.%20Resources/White%20Papers/Lambda%20MFU.pdf
- GMI Cloud — GPU utilization myths (why util ≠ useful work): https://www.gmicloud.ai/en/blog/gpu-utilization-myths
- NVIDIA — Introducing NVIDIA Fleet Intelligence (fleet-level GPU observability): https://developer.nvidia.com/blog/introducing-nvidia-fleet-intelligence-for-optimized-ai-factories/

**Deeper dives**
- MarkTechPost — "Why GPU Utilization Falls Short: Understanding SM Efficiency for Better LLM Performance": https://www.marktechpost.com/2024/09/03/why-gpu-utilization-falls-short-understanding-streaming-multiprocessor-sm-efficiency-for-better-llm-performance/
- Your module artifacts: `dashboard-is-lying` (05, the flagship source), `cost-per-million-tokens` (07, secondary), `survive-a-failure lab` (08, secondary).
