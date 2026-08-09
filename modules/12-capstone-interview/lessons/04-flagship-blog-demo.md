---
lesson: 04
title: "Flagship blog + demo: 'Your GPU dashboard is lying'"
module: 12
concept: "contrarian proof + narrated demo"
status: not-started
est_time: "3 hrs"
artifacts: ["a published flagship blog post ('Your GPU dashboard is lying'), two secondary post outlines (cost-per-1M-tokens, survive-a-failure), and a scripted 5-minute narrated demo of the value chain"]
---

# Flagship blog + demo: "Your GPU dashboard is lying"

[🎓 12 — Capstone & interview preparation](../README.md)

## Why this matters

A design doc proves you can reason; a flagship blog post proves you *know a thing most people in the room don't*. "Your GPU dashboard is lying" (artifact 05) is the single strongest public proof in your portfolio because it is **contrarian, quotable, and load-bearing on the exact insight that separates GPU-infra people from generic platform engineers**: that `nvidia-smi` "GPU utilization" tells you a kernel was scheduled on the device, not that the device did useful FLOPs. A generic SRE reads 95% util and reports the fleet is busy. You read 95% util / 31% MFU and report the fleet is *idle inside the kernels* — burning money at full price for a third of the work. That gap is the whole thesis, and it is the thing a GPU-fleet hiring manager is scanning your writing for.

The reason a blog post beats a repo here is reach and framing. A repo requires the reader to reconstruct the insight; a post hands it to them in one sentence they can repeat to their own team. "Your GPU dashboard is lying" is repeatable — a hiring manager who reads it can say it in their next standup, and now your framing is in their head with your name attached. That is how a post does distribution work a resume cannot. The title is contrarian on purpose: it makes a claim strong enough to be wrong, then earns it with the SM_ACTIVE-vs-GPU_UTIL measurement. Honesty and reproducibility are what keep it from reading as clickbait — the reader must be able to run your measurement and see 95/31 for themselves.

The demo does the other half. Some managers read; some watch. A five-minute narrated demo that walks the *value chain* — attribution to scheduling to observability to economics — and ends on the dashboard telling the truth is a portable proof you can drop into any application or send after a call. Together the post and the demo are your lead public surface: the two links that go at the very top of the README from lesson 03.

## What's new here

- **You already know** how to write clearly and how to record your screen. **Skip** the mechanics of blogging and screencasting. The new thing is *structuring for a hiring-manager reader* and *narrating the value chain instead of the code*.
- **You already know** the SM_ACTIVE vs GPU_UTIL vs MFU distinction cold (module 05). **Skip** re-deriving it. Here it becomes the *spine of a public argument* — measurement → reveal → fix → takeaway.
- **New angle 1:** the **flagship-post arc** — hook (the lie) → the measurement → the reveal (95% util / 31% MFU) → the fix (what to actually alert on) → the takeaway.
- **New angle 2:** the **flagship + two secondaries** shape — one contrarian lead, two supporting posts (cost-per-1M-tokens from 07, survive-a-failure from 08) that show range without diluting the lead.
- **New angle 3:** the **narrate-the-chain-not-the-code** demo script, and **publishable-by-default distribution** with employer specifics scrubbed.

## Core notes

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

### The flagship-post arc

1. **Hook — the lie.** Open on the dashboard everyone trusts: 95% GPU utilization, all green. Claim it's lying.
2. **The measurement.** Introduce SM_ACTIVE vs GPU_UTIL vs MFU. Show how to measure each (DCGM profiling metrics; MFU from achieved-vs-peak FLOPs). Keep it reproducible — the reader can run this.
3. **The reveal.** 95% util / 31% MFU on the same fleet. Name the gap: you are paying full $/GPU-hr for a third of the work.
4. **The fix.** What to *actually* alert on — SM_ACTIVE and goodput/MFU thresholds, not GPU_UTIL. What a truthful dashboard shows. Where batching, occupancy, and data-loading stalls hide.
5. **The takeaway.** One repeatable line the reader leaves with: util is a liveness check, not an efficiency metric; measure goodput or you're flying blind on the most expensive hardware you own.

Keep it **honest and reproducible** throughout — the credibility of a contrarian post is entirely in whether the reader can verify the reveal.

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

Record once you can narrate it without reading. The demo is a proof of *how you think*, so the voiceover carries it — the screen is just evidence.

### Distribution

Where GPU-infra hiring managers actually read: a **personal blog** (owned surface, canonical link), **GitHub** (repo README links to the post), and the **relevant technical communities** where fleet operators hang out. Make everything **publishable-by-default**: write it so it can go public, and **scrub employer specifics** — real fleet numbers become ratios or normalized figures, internal service names become generic tiers, no confidential $/GPU-hr. The reveal (95/31) works fine on a synthetic-but-realistic fleet; you never need real employer data to make the point.

## Worked example

Post opens: *"Our GPU dashboard said 95% utilization. It was lying — we were running at 31% MFU. Here's how to catch it."* Two sentences, the contrarian claim staked and quantified. Section two puts GPU_UTIL, SM_ACTIVE, and MFU in a table and shows the DCGM profiling query for SM_ACTIVE, so the reader can run it. Section three is the reveal: same fleet, `GPU_UTIL` flat at 95%, `SM_ACTIVE` at ~40%, derived MFU ~31% — with the one-line cause (low occupancy + data-loading stalls between kernels). Section four flips it to action: alert on `DCGM_FI_PROF_SM_ACTIVE < 0.5` and track goodput, *not* `GPU_UTIL`; here's what the truthful Grafana panel looks like. Takeaway: *"GPU_UTIL is a liveness check. If it's your efficiency metric, you're paying full price for a third of the work and your dashboard is hiding it."*

The demo: the reviewer watches five minutes. Opens on the architecture diagram, one sentence of value chain. Cuts to the green 95% dashboard — "this is what everyone trusts." Cuts to the SM_ACTIVE/MFU panel showing 31% — the truth shot. Traces one H100-hour: attributed to a namespace, priced at cost-per-1M-tokens, and the alert that *should* have fired. Closes on the quotable line and the repo link. The reviewer now knows, in five minutes and one post, that you understand GPUs at a level a generic platform engineer does not — which is the entire hiring thesis for a GPU-fleet role.

## Practice

Feeds `../practice/gpu-platform-capstone/README.md`:

1. **Draft the flagship post outline** end to end using the five-beat arc (hook → measurement → reveal → fix → takeaway). Write the measurement table and the actual reveal numbers from your artifact-05 data. Make the SM_ACTIVE measurement reproducible — include the DCGM query.
2. **Outline the two secondaries** (cost-per-1M-tokens from 07, survive-a-failure from 08) to one-paragraph-per-section depth. Sequence all three in the README after the flagship.
3. **Write the 5-minute demo script** on the minute-by-minute grid: the frame, the lie, the truth, the chain trace, the takeaway. Record a first pass; iterate until you can narrate without reading.
4. **Scrub for publish:** replace any real employer numbers with ratios/normalized figures and generic tier names. Confirm the reveal still lands on synthetic-but-realistic data.

## Self-check

- Why is "Your GPU dashboard is lying" a stronger flagship than a "how I built my GPU cost exporter" post? **Answer:** It's contrarian (contradicts the trusted 95% util number), quotable (compresses to one sentence a reader can repeat), and load-bearing on the util-vs-MFU/goodput insight that specifically separates GPU-infra people from generic platform engineers. A "how I built X" post proves you can build; the flagship proves you know something the reader's own team probably doesn't — which is what a GPU-fleet manager screens for.
- What exactly does `nvidia-smi` GPU utilization measure, and what should you alert on instead? **Answer:** GPU_UTIL measures the fraction of time at least one kernel was resident on the device — "a kernel was running," a liveness signal — not whether useful FLOPs were produced. A fleet can read 95% util at ~31% MFU. Alert on SM_ACTIVE (are the SMs actually issuing) and goodput/MFU (achieved vs peak FLOPs) instead, because those catch low-occupancy and stall waste that GPU_UTIL hides.
- The demo is five minutes. Should you spend them walking the code, and what's the payoff shot? **Answer:** No — narrate the value chain (attribution → scheduling → observability → economics), not the code; managers hire off how you reason about a system, not scrolling files. The payoff shot is the cut from the green 95%-util dashboard to the SM_ACTIVE/MFU panel showing 31% — the dashboard telling the truth — which makes the flagship thesis visible in one frame.

## Resources

- GMI Cloud — GPU utilization myths (why util ≠ useful work): https://www.gmicloud.ai/en/blog/gpu-utilization-myths
- NVIDIA — Introducing NVIDIA Fleet Intelligence (fleet-level GPU observability): https://developer.nvidia.com/blog/introducing-nvidia-fleet-intelligence-for-optimized-ai-factories/
- NVIDIA DCGM — profiling metrics (SM_ACTIVE and friends): https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html
- Your module artifacts: `dashboard-is-lying` (05, the flagship source), `cost-per-million-tokens` (07, secondary), `survive-a-failure lab` (08, secondary).
