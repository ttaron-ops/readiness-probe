---
lesson: 02
title: "Capstone synthesis: thread 11 artifacts into one platform"
module: 12
concept: "one coherent platform narrative"
status: not-started
est_time: "5 hrs"
prev: "01-hiring-landscape.md"
next: "03-portfolio-writeup.md"
artifacts: ["capstone storyboard: chapter list + whole-platform architecture diagram + through-line thesis"]
sources: 9
---

# Capstone synthesis: thread 11 artifacts into one platform

## Where this fits

Lesson 01 mapped the terrain: the screening-intent model, a procedure for decoding a live posting
into a rubric, the artifact-to-signal matrix naming which of your eleven artifacts converts in
which round, and a sequencing plan for which loops to run first. That lesson answered *where* to
point your portfolio. It deliberately did not answer a harder question: what do you actually *say*
when you are in the room and someone asks "walk me through your background"? Eleven artifacts
pointed at eleven different rounds is still eleven disconnected answers, and lesson 01's own
debrief argument says why that fails — six interviewers writing six different impressions reads as
scattered, not broad. This lesson fills the gap. It takes the same eleven artifacts and threads
them into **one coherent platform narrative**: a single story with a spine, told the same way in
every loop, with different chapters emphasised for different audiences. Everything from lesson 03
(the written portfolio) through lesson 09 (the mock loop) assumes this narrative already exists.

## Why this matters

Eleven artifacts scattered across eleven modules read as eleven hobby projects. The same eleven
artifacts threaded onto a single spine read as **one engineer who built a reference GPU platform**
— and that is a signal no individual project can carry, because the signal is not in any project.
It is in the joints between them.

The difference between a senior portfolio and a staff one is almost never *more projects*. It is a
through-line: a single thesis, tradeoffs you can name including the ones you rejected, real
numbers, and honesty about where the design is worse than the incumbent. A candidate who says
"here are my six GPU projects" is describing work. A candidate who says "I set out to make a
multi-tenant GPU fleet observable, attributable, schedulable and survivable, and here are the four
chapters that prove each" has imposed an operating thesis on their own work and can defend the
joints. The second is a different kind of claim about the person, not a better-formatted version of
the first.

That act — imposing structure on a pile of past work rather than doing more work — is not
packaging bolted onto real engineering. It *is* a rehearsal of a genuine staff competency. The
recurring description of what senior staff engineers do is that they notice the same problem
recurring in different disguises across an organisation, name the shared shape, and organise effort
around the shape instead of chasing the instances. Doing that to your own eleven artifacts, where
the stakes are zero and you control every input, is direct practice for the version you will have
to do inside an organisation with messy inputs and other people's opinions.

There is also a purely mechanical payoff. A rehearsed spine means you are never improvising. Every
loop draws from the same well, and the same four transitions get smoother every time you say them.
By the third loop the narrative costs you no working memory at all, which frees the whole of your
attention for the interviewer's actual follow-ups — which is where the round is really won.

## What's new here (calibration)

- **Skip** (you own it): portfolio-101, resume-bullet craft, generic "tell a story" advice, the
  brag-document mechanic itself.
- **New**: the **derivation** of the organising thesis — why these four verbs and not four others,
  and the test each verb imposes on a candidate chapter.
- **New**: the **stack view** — your narrative should be isomorphic to the system it describes, and
  the whole-platform diagram is the proof that it is.
- **New**: three **fully worked chapter arcs** (the util lie, the attribution controller, the
  tooling teardown) written out in the form you will actually speak them, with the real numbers
  from this course's own lessons rather than placeholder figures.
- **New**: the **rejected-alternatives bank** — six tradeoffs with a defensible verdict, the
  arithmetic behind each verdict, and the follow-up an interviewer will push.
- **New**: the **joints** — the four transition sentences that make eleven chapters read as one
  argument, and the branch points where an interviewer will interrupt.
- **New**: the **staff-archetype check** as a selection filter on chapters, not a personality quiz.

## Core concepts

### 1. What the narrative is actually for

Three consumers, three different failure modes, one artifact that must serve all three.

**The four-minute skim.** A reviewer opening your repo is not reading; they are scanning for three
things — one sentence that says what kind of engineer you are, one number that proves you operated
at real scale, and one diagram that shows you can hold a whole system in your head. If those three
are not above the fold, the synthesis work falls on the reviewer, and reviewers do not do synthesis
work. They remember whichever project loaded first.

**The spoken opener.** "Walk me through your background" arrives in every loop, usually in the
first four minutes, when you are least warmed up. If the answer is chronological, you spend ninety
seconds on module 01 and the interviewer's attention is gone before you reach the interesting part.
The spine fixes the order so the strongest chapter is *first*, not third.

**The debrief union.** Six interviewers each write a paragraph, and the paragraphs are compared for
consistency (lesson 01 §2). A spine makes six people write approximately the same sentence about
you, which is the strongest possible debrief shape. Without one, you get six different sentences
and the summary line becomes "did a lot of interesting things."

```
   ONE NARRATIVE, THREE CONSUMERS — and what each one drops on the floor
  ══════════════════════════════════════════════════════════════════════════════

                        ┌───────────────────────────────┐
                        │   THE CAPSTONE NARRATIVE      │
                        │   thesis · chapters · numbers │
                        └───────┬───────┬───────┬───────┘
                                │       │       │
              ┌─────────────────┘       │       └──────────────────┐
              ▼                         ▼                          ▼
     ┌────────────────┐        ┌────────────────┐        ┌──────────────────┐
     │ 4-MIN SKIM     │        │ SPOKEN OPENER  │        │ DEBRIEF UNION    │
     │ (repo README)  │        │ (30 s / 3 min) │        │ (6 paragraphs)   │
     ├────────────────┤        ├────────────────┤        ├──────────────────┤
     │ NEEDS:         │        │ NEEDS:         │        │ NEEDS:           │
     │  · 1 sentence  │        │  · fixed order │        │  · one repeated  │
     │  · 1 number    │        │  · hook first  │        │    sentence      │
     │  · 1 diagram   │        │  · 4 joints    │        │  · consistency   │
     ├────────────────┤        ├────────────────┤        ├──────────────────┤
     │ DROPS:         │        │ DROPS:         │        │ DROPS:           │
     │  everything    │        │  chronology,   │        │  peak            │
     │  below the     │        │  build logs,   │        │  performance in  │
     │  fold          │        │  tool lists    │        │  any one round   │
     └────────────────┘        └────────────────┘        └──────────────────┘
              │                         │                          │
              └─────────────────────────┴──────────────────────────┘
                                        ▼
                    ALL THREE ARE SERVED BY THE SAME SPINE.
                    If you have to write three different stories,
                    you do not have a thesis — you have three pitches.
```

### 2. Deriving the thesis, rather than asserting it

The organising thesis is:

> **"Making a multi-tenant GPU fleet observable, attributable, schedulable, and survivable."**

Do not memorise it as a slogan. Derive it, because the derivation is what lets you defend it when
someone asks "why those four?"

Start from the operator's position. You have a fleet of depreciating accelerators, several tenants
with conflicting demands, and a finance function that wants to know where the money went. There are
exactly four questions you must be able to answer before you can run that fleet as a business, and
each one is answerable only if the previous one is:

1. **Can I see what the hardware is doing?** — *observable.* Without an honest measurement of work
   done, everything downstream is built on a number that does not mean what it says. This is why
   the util-lie chapter is first in the narrative: it is not the flashiest chapter, it is the
   *load-bearing* one. Every other claim depends on the measurement being honest.
2. **Can I say who did it?** — *attributable.* An honest device-level measurement with no tenant
   map is a fact nobody can act on. Attribution is the join that turns a hardware fact into an
   organisational one, and it is the join that neither the billing layer nor the telemetry layer
   ships, because it sits at the seam between them.
3. **Can I decide who gets it next?** — *schedulable.* Once you know who is using what, the
   question becomes allocation policy: quota, fair share, gang scheduling, preemption, borrow and
   reclaim. Scheduling without attribution is guesswork dressed as policy.
4. **Does it survive contact with reality?** — *survivable.* All three of the above assume the
   hardware works. At fleet scale it does not: Meta's measured mean-time-to-failure for 1,024-GPU
   jobs is 7.9 hours (arXiv:2410.21680), with shorter figures at larger scales given as
   projections. A platform that is observable, attributable and schedulable but falls over on a
   dead node is a demo.

**The four verbs are a dependency chain, not a list**, and that is why the order is fixed. Say the
chain out loud once and the thesis stops sounding like a slogan: *"You cannot attribute what you
cannot honestly measure, you cannot schedule fairly what you cannot attribute, and none of it
matters if a dead GPU takes the whole job down."*

**Each verb is also a test.** A candidate chapter earns a featured slot only if it clearly answers
one of the four *and* moves a number. "Interesting" is not a criterion. Apply the test honestly and
most portfolios lose three or four chapters, which is the point — the cut list is what makes the
remaining chapters legible.

### 3. The value chain, drawn as the stack it actually is

The eleven artifacts are not a list of projects. They are layers, and the narrative should be
isomorphic to the system. Here is the whole-platform view — this is the diagram lesson 03 renders
as the front door of your capstone repo, and the thing you draw on a whiteboard when someone says
"show me how it fits together."

```
   THE REFERENCE GPU PLATFORM — signal flow, and which artifact owns each edge
  ══════════════════════════════════════════════════════════════════════════════

   ┌───────────────────────────────────────────────────────────────────────┐
   │ SURVIVABLE   health · XID · lemon-node detect · drain · checkpoint    │
   │              ── artifacts 08 (survive-a-failure), 04 (failure log)    │
   │                 05.5 (XID/health layer)                              │
   └────────────▲──────────────────────────────────────────────────────────┘
                │ protects everything below it; consumes health signals
   ┌────────────┴──────────────────────────────────────────────────────────┐
   │ SCHEDULABLE  quota · gang · borrow/reclaim · topology-aware placement │
   │              ── artifacts 06 (Kueue showback), 02b (topology),        │
   │                 09 (fabric read)                                      │
   └────────────▲──────────────────────────────────────────────────────────┘
                │ decides allocation using the attributed ledger
   ┌────────────┴──────────────────────────────────────────────────────────┐
   │ ATTRIBUTABLE  UUID → pod → namespace → share → dollars                │
   │               ── artifacts 04 (per-pod attribution controller),       │
   │                  02 (the operator + CRDs), 11 (FOCUS schema,          │
   │                  tooling teardown), 07 (cost per 1M tokens),          │
   │                  10 (capex vs cloud)                                  │
   │                                                                       │
   │   ┌──────────────────────────────────────────────────────────────┐    │
   │   │  THE JOIN NOBODY SHIPS                                       │    │
   │   │  allocation record (kubelet pod-resources)                   │    │
   │   │            ╳  meets at the GPU UUID  ╳                       │    │
   │   │  hardware measurement (DCGM / NVML)                          │    │
   │   │  → Σ allocated_share + unallocated ≡ 1.000  (identity A)     │    │
   │   └──────────────────────────────────────────────────────────────┘    │
   └────────────▲──────────────────────────────────────────────────────────┘
                │ consumes an HONEST ratio, not a duty cycle
   ┌────────────┴──────────────────────────────────────────────────────────┐
   │ OBSERVABLE   SM_ACTIVE / TENSOR_ACTIVE / DRAM_ACTIVE, not GPU_UTIL    │
   │              ── artifacts 05 (the util-lie exhibit + query pack),     │
   │                 01 (the exporter), 03 (hardware report)               │
   └────────────▲──────────────────────────────────────────────────────────┘
                │ reads counters off the silicon
   ┌────────────┴──────────────────────────────────────────────────────────┐
   │ THE MACHINE  namespaces/cgroups · NUMA · PCIe/NVLink · driver · etcd  │
   │              ── artifacts 01b (container anatomy), 02b (topology),    │
   │                 09 (network), 10 (bare metal / KTHW)                  │
   │              ── the substrate. Proves depth; never the headline.      │
   └───────────────────────────────────────────────────────────────────────┘

   THE ONE-SENTENCE READ OF THIS DIAGRAM:
     "Signals come off the silicon, get made honest, get attributed to a
      tenant, drive an allocation decision, and the whole stack is wrapped
      in a survivability layer — because the hardware fails on a schedule
      you can compute."
```

Two properties of this diagram are worth defending out loud, because both are follow-up magnets.

**Why "the machine" is a substrate layer and not a chapter.** Container anatomy, NUMA topology and
fabric reads are how you *know* things, not what you *built*. They belong under the stack as
evidence of depth, surfaced when an interviewer probes a mechanism ("how does the container even
see the device?"), never as a headline. A portfolio that leads with "I read the Linux namespace
API" is describing study, not a platform.

**Why attribution is the middle layer and not the top.** It is tempting to put cost at the top
because it is the business-facing outcome. Resist it. Cost is *derived* from attribution, and
attribution is derived from honest measurement; putting cost on top hides the dependency that makes
your argument work. The layering is the argument.

### 4. Three worked chapter arcs

Every featured chapter uses the same six-beat arc: **problem → ownership → design → rejected
alternative → number → evolution.** Here are the three load-bearing chapters written out in full,
in the register you would actually speak them. These are not templates to fill in later; they are
the material, drawn from this course's own findings.

#### Chapter A — "Your GPU dashboard is lying" (artifact 05)

> **Problem.** Every GPU dashboard in existence leads with a utilisation number, and that number
> does not mean what the whole industry assumes. `DCGM_FI_DEV_GPU_UTIL` is field 203, an unmodified
> passthrough of NVML's `nvmlUtilization_t.gpu`, which the header defines as *the percentage of a
> short driver-chosen sample window during which at least one kernel was resident on the GPU.* It
> is a threshold at one. One kernel and ten thousand concurrent kernels evaluate the same
> predicate. It has no arity and it has no notion of how many SMs exist.
>
> **Ownership.** I derived the failure rather than reading about it. For an 8B-parameter model in
> BF16 on an H100 SXM5 at batch size 1: every weight must be read from HBM once per token, so 8e9
> params × 2 bytes = 16 GB at 3.35 TB/s ≈ 4.78 ms of memory time, against 2 × 8e9 = 16 GFLOP at
> 989 TFLOP/s ≈ 16.2 µs of compute time. The ratio is 0.0034. The workload sits about 295× to the
> left of the H100's roofline ridge point. So the tensor pipes *must* be idle roughly 99.7% of the
> time — and simultaneously a kernel is resident for essentially the whole NVML sample window,
> because the inter-kernel gap is microseconds against a window of at least ~167 ms.
>
> **Design.** The honest metric is on a completely different collection path.
> `DCGM_FI_PROF_SM_ACTIVE` (field 1002) is not a driver counter at all — it is the ratio of summed
> active cycles to summed elapsed cycles across every SM, produced by differencing two hardware
> performance-monitor snapshots. On a 132-SM H100, a kernel occupying 32 SMs continuously reads
> 32/132 = 0.24, even though those 32 SMs are fully busy. I built the side-by-side panel, the
> detector query, and the alerting policy that follows from it.
>
> **Rejected alternative.** `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (1001) is the tempting fix, because it
> carries the `PROF` prefix and looks like the honest family. It is not: its own definition is
> *context bound and a pipe busy*, which is a presence duty cycle reimplemented on the hardware
> path. On a batch-1 decode server it reads ~1.0 alongside `SM_ACTIVE ≈ 0.16`. I rejected it, and
> the reason it matters is that 1001 ships **enabled** in dcgm-exporter's default counter CSV while
> 1002 and 1003 ship **commented out** — so the default configuration gives you two presence
> metrics and zero breadth metrics.
>
> **Number.** Same GPU, same second: `GPU_UTIL` 99–100, `SM_ACTIVE` 0.16, `SM_OCCUPANCY` 0.09,
> `PIPE_TENSOR_ACTIVE` 0.011, `DRAM_ACTIVE` 0.71. Eighty-six percent of the compute die dark while
> the dashboard reads solid green. Multiply `DRAM_ACTIVE` by peak: 0.71 × 3.35 TB/s ≈ 2.4 TB/s
> achieved — HBM is the actual wall.
>
> **Evolution.** The fix moves the honest metrics and not the lie. Enabling continuous batching
> took `SM_ACTIVE` from 0.16 to 0.55 and `PIPE_TENSOR_ACTIVE` from 0.011 to 0.19, for about 2.9×
> serving throughput on the same eight cards — and `GPU_UTIL` read 99 before, during and after.
> That is the strongest form of the argument: **the default metric is uninformative about the
> problem and uninformative about the solution.**

#### Chapter B — the per-pod attribution controller (artifact 04, with 02)

> **Problem.** Finance asks what a team's training run cost last night, and every off-the-shelf
> answer is wrong for a structural reason. Cloud billing stops at the node — an 8×H100 box is one
> line item. `nvidia.com/gpu` counts allocations, not use. DCGM measures devices, and under any
> software sharing mode there are fewer devices than tenants. The only honest per-pod number is a
> join across all three, and nobody ships it in a form that matches an arbitrary mix of sharing
> modes and an arbitrary rate card.
>
> **Ownership.** I built the join. Two independent loops, because the two sources fail differently:
> an ownership loop that lists the kubelet's pod-resources API and resolves each device ID into a
> join key, and a utilisation loop that reads DCGM and NVML. They meet at exactly one place — the
> GPU UUID — and that single meeting point is the whole design.
>
> **Design.** The load-bearing type decision is one character: the owner map is
> `map[string][]Owner`, not `map[string]Owner`. Under time-slicing the device plugin fabricates
> annotated IDs of the form `GPU-<uuid>::<replica>`, so pod-resources hands you a *distinct string
> per pod* — the ownership map survives sharing even though the metric does not. A
> `map[string]Owner` silently keeps whichever holder was written last, which is exactly the bug
> that charges one pod for the whole card and the others for nothing.
>
> **Rejected alternative.** I could have divided a shared device's utilisation by the number of
> pods currently holding a replica. I divide by the configured `replicas` count instead. With
> `replicas: 4` and three co-resident pods, each pod's share is 0.25 and the fourth replica's 0.25
> is unallocated — because dividing by three would socialise idle capacity onto the tenants,
> converting a platform problem (you over-provisioned the replica count) into a tenant cost they
> cannot act on.
>
> **Number.** Two reconciliation identities, both continuously asserted in PromQL. Identity A:
> per physical GPU, the sum of holders' allocated shares plus the unallocated remainder is exactly
> 1.000 — a sum above 1 means double-counted holders, a sum below 1 means an unresolved device ID
> or unbooked MIG stranding. Identity B: per measurement identity, the sum of per-pod utilised
> shares approximates the device's own utilisation, and a ratio of exactly N means you forgot to
> deduplicate a fanned-out series. And the honesty number: on a 64-GPU fleet with 24 GPUs
> time-sliced, 46.8% of every utilisation-based chargeback dollar rests on an estimate — about
> $63.8k/month at $4.00/GPU-hour — which is the business case for the per-PID probe, stated in
> dollars instead of in engineering preference.
>
> **Evolution.** The exposure fraction is the metric that drives the roadmap. Getting per-PID
> attribution working on those 24 shared GPUs drops it to near zero and relabels the same dollars
> from `shared-estimate` to `per-pid`. That is a prioritisation argument a finance partner can
> read.

#### Chapter C — why I built it instead of adopting the incumbent (artifact 11)

> **Problem.** The first question any engineering director asks is "doesn't a tool already do
> this?" They have usually already installed one. An opinion loses that conversation; a location in
> the source wins it.
>
> **Ownership.** I read OpenCost's GPU cost path end to end, at a named commit, and traced a
> dollar from the node's price metric to the pod's `gpuCost` field. Three questions, applied to any
> cost engine: what is the numerator, where does the price come from, and where does the
> interesting telemetry actually land?
>
> **Design of the finding.** The numerator is a Kubernetes resource *request* count, emitted as
> `container_gpu_allocation` from `costs.GPUReq[0].Value`, turned into hours by `applyGPUsAllocated`
> as `GPUHours = value × hrs`, and multiplied in `applyNodesToPod`: `alloc.GPUCost = alloc.GPUHours
> × node.CostPerGPUHr`, where the price is one scalar per *physical* GPU. DCGM utilisation *is*
> queried — correctly `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, not the util lie — and stored in
> `GPUAllocation.GPUUsageAverage`, where in the allocation path it is read by exactly one thing: a
> display ratio. It is never multiplied into cost.
>
> **Rejected alternative — and this is the fair part.** I did not conclude the tool is bad. It is
> an allocated-ledger engine, and on a fleet of exclusively held whole GPUs it is *correct*. Five
> of the six gaps I found are one correct design assumption meeting hardware that changed
> underneath it. The sixth is physics.
>
> **Number.** Same H100, same $2.10/GPU-hour price, same one-hour window, four tenancy
> configurations: **$2.10 exclusive (correct), $14.70 under 7-way MIG (7× high), $0.00 under
> time-slicing with GPU Feature Discovery's `renameByDefault=true` (the shared resource key is
> absent from both numerator paths), and $8.40 with `renameByDefault=false` (N× high).** The spread
> is not noise — it is four different code paths, each traceable to a named line, and which one you
> get is decided by a GPU-operator flag nobody in the cost conversation is usually aware of.
>
> **Evolution.** The story has moved, and saying so is the credential. As of 2026 OpenCost ships a
> feature-gated `pkg/inferencecost` module that genuinely joins allocation cost to vLLM token
> counters and emits `llm_cost_per_million_tokens` under two named cost bases. So the honest 2026
> claim is not "nobody joins dollars, utilisation and app counters" — it is "one tool does, off by
> default, for vLLM inference only, on top of a numerator that is still whole-GPU. The remaining
> gap is the numerator and the sharing regimes, not the join."

**Notice what the three chapters do together.** A is a measurement finding. B is an engineering
response to it. C is the justification for having built B rather than installed something. That is
a complete argument with no missing step, and it is why these three are the load-bearing chapters
and the other eight are supporting.

### 5. The rejected-alternatives bank

Naming what you rejected is the single highest-density staff signal available in a portfolio
review, because it is the one thing that cannot be faked by having read about a technology. Here is
the bank, with the verdict, the reason, and the follow-up you will get.

| Decision | Options | Verdict and the actual reason | The follow-up you will get |
|---|---|---|---|
| **Sharing model** | passthrough · MIG · time-slicing · MPS | Depends on tenancy, and say so. MIG gives hardware isolation and clean attribution — a MIG instance belongs to exactly one container — at the cost of static partitions that do not rebalance and stranded capacity (7×`1g.10gb` sums to 0.858 of the card on a memory basis; the missing 14.2% is real capacity that cannot be allocated). Time-slicing gives density and zero isolation. MPS gives a *bounded* error because the control daemon caps thread percentage. | *"Why not always MIG?"* → because it cannot rebalance under shifting traffic, and because the partition geometry is a decision someone has to revisit; on a fleet with many small, bursty tenants the stranding plus the reconfiguration cost dominates the isolation benefit. |
| **Cost model** | chargeback · showback | Showback first. In a scarce-GPU multi-tenant setting, hard chargeback creates hoarding: teams hold capacity defensively because releasing it means losing it. Showback surfaces the same number without the incentive. Move to chargeback only once the attribution passes conservation and the disputes have a protocol. | *"So the numbers don't matter?"* → they matter more, not less; showback is what earns the right to bill later, because the first disputed invoice on an unvalidated number ends the programme. |
| **Utilisation integration** | `avg_over_time × window` · `sum_over_time × Δ` | `sum_over_time`. `avg_over_time` averages only over samples that exist, so a pod that ran 9 hours of a 24-hour window gets its mean extrapolated across the whole window — an overstatement of exactly `window ÷ time-present`, which was 2.67× in my own worked case. And it errs in the direction that makes the fleet look *more* utilised, i.e. it understates the problem on exactly the bursty workloads where the problem is worst. | *"How do you know your integral is right?"* → a synthetic ground-truth run: saturate one GPU for exactly 600 s with a burn tool, expect 0.1667 GPU-hours, accept 0.150–0.184 (±2 scrape intervals). 0.30 means double counting, ~0.08 means Δ is 15 s not 30 s, ~4.0 means you reproduced the `avg_over_time` bug. |
| **Build vs adopt** | extend OpenCost · build the controller | Build the correction layer, but be precise about why: the incumbent's cost expression has exactly two factors, an integer request count and a per-physical-device price, and a sub-device fraction is neither. It needs a third term. That is an input-model change, not a configuration. | *"Why not upstream a patch?"* → two of the six gaps genuinely are a few lines (add the shared resource key; repair an inverted guard) and are worth upstreaming. The fractional numerator needs a decision on the fractional basis — SM slice vs memory vs blended — that the standard deliberately does not prescribe. |
| **Scheduling** | Kueue · Volcano · Slurm | Kueue, because the control plane was already operator-driven and Kueue's quota model (nominal quota, borrowing limit, cohort reclaim) is Kubernetes-native and composes with the CRDs already in the cluster. Volcano and Slurm are both defensible; the reason to reject them here is integration surface, not capability. | *"What would make you pick Slurm?"* → an HPC-native tenant population that already writes batch scripts, and topology-aware placement requirements that the Kubernetes scheduler framework makes awkward. |
| **Reclaim policy** | on `GPU_UTIL` · on `SM_ACTIVE` · on `SM_ACTIVE` gated by `FB_USED` | Gated. Reclaiming on activity alone kills a warm, loaded serving replica: `SM_ACTIVE ≈ 0` with 58 GB of resident weights and KV-cache arena is a *paused* replica, not an idle GPU. One such reclaim and the whole reclaim system gets switched off politically. | *"What threshold?"* → below ~2 GB framebuffer the card is not holding a served model; that is a floor to tune per fleet, not a universal constant, and the honest answer names it as a fleet-specific tuning parameter. |

**The discipline that makes this bank work:** for every verdict, you must be able to state the
condition under which you would reverse it. A verdict with no reversal condition is a preference,
and interviewers can hear the difference.

### 6. Selecting chapters: the cut is the signal

Feature **three to five** chapters. Everything else becomes a chapter index. The selection test is
mechanical:

```
   CHAPTER SELECTION — run every artifact through this
  ══════════════════════════════════════════════════════════════════════════════

   artifact
      │
      ▼
   ┌─────────────────────────────────────────┐
   │ Q1 · Does it answer ONE of the four     │──── no ───▶ SUBSTRATE
   │      verbs, cleanly and primarily?      │             (depth evidence,
   └───────────────┬─────────────────────────┘              surfaced on probe)
                  yes
                   ▼
   ┌─────────────────────────────────────────┐
   │ Q2 · Can I state a NUMBER I measured?   │──── no ───▶ NOT READY
   └───────────────┬─────────────────────────┘             (go measure, or
                  yes                                       demote to index)
                   ▼
   ┌─────────────────────────────────────────┐
   │ Q3 · Can I name what I REJECTED and     │──── no ───▶ INDEX
   │      the condition that reverses it?    │             (a build, not a
   └───────────────┬─────────────────────────┘              decision)
                  yes
                   ▼
   ┌─────────────────────────────────────────┐
   │ Q4 · Does it survive "how do you know   │──── no ───▶ INDEX
   │      that?" asked three times?          │             (borrowed depth)
   └───────────────┬─────────────────────────┘
                  yes
                   ▼
              ★ FEATURED CHAPTER ★
              (cap at five; if six pass, the sixth is
               the one whose number is softest)
```

Running this course's eleven artifacts through the filter gives a stable answer: **05 (observable),
04 (attributable), 11 (attributable — the justification), 06 (schedulable), 08 (survivable)** are
featured; 01, 01b, 02b, 03, 07, 09, 10 become the index and the substrate. Artifact 02 (the
operator) is a special case — it is not its own chapter, it is the *vehicle* for chapter B, and
narrating it separately splits one strong story into two weak ones.

**The cut list is itself a talking point.** "I built eleven things; five carry the argument and six
are evidence I understand the substrate" is a sentence that demonstrates judgement about your own
work. Volunteering the cut is stronger than being caught with an unfeatured project you cannot
defend.

### 7. The staff-archetype check

Will Larson's staff-archetypes framework names four shapes the title covers: the **Tech Lead**
(guides one team's approach and execution), the **Architect** (owns direction, quality and approach
across a critical area), the **Solver** (digs into a gnarly cross-cutting problem, resolves it,
moves on), and the **Right Hand** (extends a leader's attention across a large organisation).

Your four-verb thesis is implicitly arguing **Architect**, with a strong secondary claim to
**Solver**. Architect, because you defined the direction and approach for an entire fleet-management
domain end to end rather than executing within someone else's. Solver, because two chapters — the
util-lie investigation and the OpenCost teardown — are exactly the shape of digging into one
gnarly, bounded problem and resolving it cleanly.

**Use it as a filter, not a personality quiz.** If a candidate featured chapter supports neither
"defined the approach for a critical area" nor "solved a hard bounded problem," it does not belong
in the featured five. Run it: chapter 06 (Kueue showback) supports Architect. Chapter 08
(survive-a-failure) supports Solver. Chapter A supports Solver strongly and Architect weakly.
Chapters B and C support Architect strongly. The set is coherent, which is the outcome you want —
a portfolio that argues two adjacent archetypes reads as one person; a portfolio that argues all
four reads as unfocused.

The follow-up this arms you for is real and common: *"so what kind of engineer are you?"* Answer in
one sentence with the archetype and the evidence, not with adjectives.

### 8. The joints — what makes eleven chapters one argument

The chapters are not the hard part. The *transitions* are, because a transition is where an
interviewer decides whether they are hearing one argument or a sequence of anecdotes. There are
exactly four joints, each of which is one sentence, and each of which should be rehearsed until it
is automatic.

```
   THE NARRATIVE SPINE — fixed order, four joints, and the branch points
  ══════════════════════════════════════════════════════════════════════════════

   [THESIS]  "observable → attributable → schedulable → survivable"
       │
       │  ◀── if interrupted here: "why those four?" → the dependency chain (§2)
       ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ CH.A  THE LIE   GPU_UTIL 99 / SM_ACTIVE 0.16 / TENSOR 0.011          │
   │                 86% of the die dark on a solid-green dashboard       │
   └──────────────────────────┬───────────────────────────────────────────┘
        JOINT 1 ▶ "Once the measurement is honest, the next question is
                   whose work it was — and that's the join nobody ships."
                          │
       ◀── branch: "why is nobody shipping it?" → jump straight to CH.C, then
           come back. The spine tolerates this reorder; a chronological story
           does not.
                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ CH.B  THE JOIN  UUID meets pod-resources; Σshare + unalloc ≡ 1.000   │
   │                 46.8% of chargeback $ labelled as an estimate        │
   └──────────────────────────┬───────────────────────────────────────────┘
        JOINT 2 ▶ "The obvious objection is 'doesn't OpenCost do this?' —
                   so I read the source before I wrote a line of mine."
                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ CH.C  THE GAP   $2.10 / $14.70 / $0.00 / $8.40, same card, same tool │
   └──────────────────────────┬───────────────────────────────────────────┘
        JOINT 3 ▶ "With an attributed ledger you can finally make an
                   allocation decision instead of a guess."
                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ CH.D  THE POLICY  Kueue quota, borrow/reclaim, gang scheduling       │
   └──────────────────────────┬───────────────────────────────────────────┘
        JOINT 4 ▶ "And all of that assumes the hardware works, which at
                   fleet scale it measurably does not."
                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ CH.E  THE FAILURE  MTTF 7.9 h at 1,024 GPUs (measured, arXiv:        │
   │                    2410.21680); failure-mode log; checkpoint cost    │
   └──────────────────────────┬───────────────────────────────────────────┘
                              ▼
   [CLOSE]  "Most of it ran on a 200-node SIMULATED fleet with the traces
             published — and here's where my design is worse than the
             incumbent's."   ◀── the honesty close. Do not skip it.
```

**Joint 2 is the most valuable sentence in the whole narrative**, because it converts the
interviewer's strongest objection into your strongest chapter *before they raise it*. Volunteering
an objection you have already answered is a different signal from answering it when cornered, and
the difference is legible.

**The honesty close is not modesty.** "Here is where my design is worse" is the sentence that makes
the preceding six minutes believable. Concretely, you have three to offer: per-tenant attribution
under time-slicing is an approximation with an explicit uncertainty flag and no amount of code
fixes it, because the device exposes one counter and not one per CUDA context; the fleet numbers
come from a simulation and the real-hardware validation was one rented afternoon; and the dollar
figures are a dated snapshot on a stated basis, not a rate card you negotiated.

### 9. Compression: the same spine at four lengths

You will need this narrative at four durations, and they are not the same story truncated. They are
four different selections from the same spine.

| Length | Where it is used | What survives | What is cut |
|---|---|---|---|
| **One line** (10 s) | recruiter screen, LinkedIn headline, the first sentence of your README | the specialisation + the one artifact | everything else |
| **30 seconds** | "tell me about yourself"; the opener in every loop | thesis + chapter A + one number + the honesty caveat | joints 2–4, all other chapters |
| **3 minutes** | "walk me through a project you're proud of" | one chapter at full six-beat arc | the thesis becomes one clause, not a paragraph |
| **6–8 minutes** | portfolio review; the narrated demo | thesis + A → B → C → D → E with all four joints | the substrate layer, unless probed |

The one-liner, written out:

> *"Platform engineer specialised into GPU fleet cost and observability — I built the per-pod GPU
> attribution controller that the default tooling structurally can't."*

The 30-second version:

> *"I set out to make a multi-tenant GPU fleet observable, attributable, schedulable and
> survivable — one reference platform, not scattered scripts. The chapter I lead with is
> counterintuitive: your GPU dashboard is lying to you. On a batch-1 decode server the dashboard
> read 99% utilisation while only 16% of the SMs were lit and the tensor pipes were at 1% —
> because the field everyone trusts measures whether a kernel was resident, not whether the
> silicon did work. I built the exporter and the Kubernetes controller that join the honest
> counters to the kubelet's allocation record, per pod, with a conservation identity that proves
> the shares sum to exactly one physical GPU. Then I read OpenCost's source and found the same card
> bills $2.10, $14.70, $0.00 or $8.40 depending only on how it's shared. Most of it ran on a
> 200-node simulated fleet with the traces published, and the time-slicing split is an explicit
> approximation, not a measurement."*

That is roughly 150 words: thesis, contrarian chapter, three real numbers, the justification for
building, and two honesty caveats. **Rehearse it until the caveats come out as confidently as the
numbers**, because a caveat delivered apologetically reads as a weakness and the same caveat
delivered flatly reads as rigour.

## Perspectives

**Candidate's view — range without organisation reads as scattered.** The instinct is to show
eleven projects because eleven proves breadth. But an interviewer skimming eleven bullets has no
way to tell whether you understand how they relate or merely built a lot of things, and the
cheapest interpretation is the second. One thesis converts identical facts from "a lot of projects"
into "one deliberate multi-year programme." Same evidence, different claim about the person.

**Reviewer's view — the four-minute skim is a search, not a read.** They are looking for one
sentence, one number, and one diagram. Anything that requires them to synthesise is dropped. This
is also why the whole-platform diagram must be a *single system with one signal flow*, not eleven
project diagrams pasted in sequence — a collage still reads as eleven projects, no matter how neat
the collage is.

**Systems-architecture view — the narrative should be isomorphic to the system.** The four verbs
are not a mnemonic; they are a real layered shape. Signals flow up from the silicon, get made
honest, get attributed, drive allocation, and sit under a survivability layer. Telling the story
with the same structure the system has is easier to believe and much easier to defend under
follow-up, because every "why is that there?" question has a structural answer rather than a
chronological one.

**Staff-signal view — imposing structure is the competency, not a proxy for it.** The move this
lesson drills — scan scattered work, find or build the shared thesis, organise around the shape
instead of the instances — is close to verbatim what staff engineers describe doing on the job.
Practising it on your own eleven artifacts, where you control every input, is direct rehearsal for
the version with messy inputs and other people's opinions.

**Vendor's view — the industry organises the same way.** GPU-cloud vendors sell exactly this shape
as a product: a single operating layer that unifies observability, fleet and node lifecycle
management, anomaly and straggler detection, and operational tooling, rather than a pile of
standalone dashboards. If commercial products converge on observable → attributable/actionable →
survivable, imposing that structure on your own work is not artificial packaging; it is the shape
an experienced fleet operator already thinks in.

## Real-world use cases

- **NVIDIA `dcgm-exporter`'s default counter CSV and shipped Grafana dashboard.**
  `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_PROF_GR_ENGINE_ACTIVE` ship enabled; `SM_ACTIVE` and
  `SM_OCCUPANCY` ship commented out, and the vendor's own eight-panel dashboard (grafana.com
  #12239) has no SM-breadth panel at all. **What it shows:** chapter A's claim is not a
  provocation — the misleading metric is the shipped default and the honest one is opt-in. This is
  the exhibit that makes the whole "observable" layer of your thesis land in one sentence.

- **OpenCost issue #3900** (filed 2026-07-05, closed 2026-07-09) — `costBasis=usage` and
  `costBasis=allocation` returning figures within ~1% of each other for a vLLM server measured at
  roughly 11% GPU utilisation; expected ≈$0.275 against actual ≈$2.50 on a 1-GPU, 1-hour, $2.50/hr
  workload. **What it shows:** chapter C's central argument stated by the reporter and accepted by
  the maintainers, then fixed in four days. Citing a project's own agreed issue is a different
  register from criticising the project, and it is the register that survives a room containing
  someone who contributes to it.

- **OpenCost issue #3828** (filed 2026-06-02, open at last reading) — a HAMi user whose pod is
  allocated a quarter of a card and billed for a whole one, whose stated workaround is to look up
  the fraction externally and multiply `gpuCost` by it. **What it shows:** an unaffiliated stranger
  independently specifying chapter B's correction layer as a feature request. That is the strongest
  available evidence that the gap you built for is real and not self-generated.

- **Meta, *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025).** Eleven months on two large A100 research clusters; measured MTTF
  of 7.9 hours for 1,024-GPU jobs, with 1.8 h at 16,384 GPUs and 0.23 h at 131,072 given as the
  paper's projections. **What it shows:** the fourth verb is not rhetorical. It also shows why
  citation precision matters — quoting the projections as measurements is the kind of slip a
  careful interviewer catches, and it costs more credibility than the number was worth.

- **Will Larson's staff-archetypes framework.** Tech Lead / Architect / Solver / Right Hand.
  **What it shows:** the vocabulary a panel is often already using internally, and a filter you can
  apply to your own chapter selection before someone else applies it to you.

## Worked example

**Scenario.** A portfolio review, forty-five minutes, one hiring manager and one senior engineer.
They have your repo open. The opening line is "walk us through it."

**Minute 0:00–0:30 — the thesis and the hook.** Deliver the 30-second version from §9 verbatim.
Stop talking. Do not fill the silence.

**Minute 0:30–1:00 — the senior engineer interrupts.** *"Hang on — 99% utilisation and 16% SM
active? That sounds like your exporter was broken."* This is the single most likely interruption,
and it is a gift, because the answer is a derivation rather than an assertion:

> *"It's not an exporter bug, it's two independent collection paths. Field 203 is a straight
> passthrough of `nvmlUtilization_t.gpu` — the driver integrates a one-bit predicate, 'is at least
> one kernel resident', over a sample window of one second to a sixth of a second depending on the
> product. `SM_ACTIVE` is field 1002 and comes from differencing two hardware performance-monitor
> snapshots: summed active cycles over summed elapsed cycles across all 132 SMs. Neither can be
> derived from the other. And you can predict the gap before measuring it — an 8B model in BF16 at
> batch 1 moves 16 GB of weights per token at 3.35 TB/s, which is 4.78 ms, wrapped around 16 GFLOP
> of arithmetic at 989 TFLOP/s, which is 16 microseconds. The tensor pipes have to be idle about
> 99.7% of the time, and a kernel has to be resident essentially the whole window, because the
> inter-kernel gap is microseconds against a window of at least 167 milliseconds."*

**Minute 1:00–1:30 — the hiring manager pivots to money.** *"So what's it worth?"*

> *"On a 500-GPU H100 fleet at $2.50 per GPU-hour — on-demand basis, mid-2026 snapshot, and quotes
> in that window ranged from about $2 to $11, so treat it as a band — capacity costs about $10.9M a
> year. In my own 24-GPU worked case, 57% of allocated GPU-hours were doing no SM work. A
> ten-point fleet-wide utilisation move is about $1.1M a year. But I'd be careful with that number:
> some of that gap is memory-bandwidth-bound inference that physics will not give back, which is
> exactly why my dashboard decomposes the gap into six buckets with six different owners instead of
> reporting one waste percentage."*

That last clause is the staff move. You volunteered the limit of your own headline number before
anyone tested it.

**Minute 1:30–2:30 — joint 2, delivered on schedule.** *"The obvious objection is 'doesn't OpenCost
do this?' so I read the source before writing a line of mine."* Then chapter C, compressed to the
four-number table. Expect: *"Couldn't you just configure it?"* Answer: *"No — and that's the
interesting part. The cost expression has exactly two factors, an integer request count and a price
per physical device. A sub-device fraction is neither, so it needs a third term. That's an
input-model change, not a setting. Two of the six gaps genuinely are a few lines and are worth
upstreaming; the fractional numerator isn't one of them."*

**Minute 2:30–4:00 — chapters D and E, then the honesty close.** Kueue quota and reclaim, the
failure-mode log, then: *"Most of this ran on a 200-node simulated fleet with the traces published.
I rented one real GPU for an afternoon to validate the simulation rather than to develop against
it. And the per-tenant split under time-slicing is an approximation with an explicit uncertainty
flag — the device exposes one counter, not one per CUDA context, so no amount of code recovers N
per-tenant values from one device value."*

**Minute 4:00 onward — you are now in their questions, which is where you want to be.** The whole
purpose of the spine is to get here in four minutes with all three consumers served: they have the
sentence, the numbers, and the diagram, and every subsequent question lands on a chapter you have
already rehearsed at three lengths.

**Self-scoring afterwards.** Did you land the thesis in one sentence? Did every chapter carry a
number? Did you name at least two rejected alternatives with their reversal conditions? Did the
honesty close come out flat rather than apologetic? Did joint 2 arrive before the objection? Five
yeses is a pass; anything less is a specific thing to rehearse, not a vague feeling.

## Practice

Build the storyboard in
[GPU platform capstone](../practice/gpu-platform-capstone/README.md).

1. **Write the through-line sentence.** Start from the four verbs, make it yours, and write the
   *dependency chain* underneath it in one sentence — you will be asked why those four.

2. **Run all eleven artifacts through §6's selection filter.** Record the verdict per artifact
   (featured / index / substrate / not-ready) and, for every "not ready," the specific measurement
   that would fix it. Cap featured at five.

3. **Write the six-beat arc for each featured chapter**, in speaking register, using §4's three as
   the model. Every arc must contain a number you measured and a rejected alternative with its
   reversal condition. Do not write these as bullet points — write them as prose you would say,
   because the failure mode is a chapter that reads well and speaks badly.

4. **Fill the rejected-alternatives bank.** Take §5's six rows, replace the verdicts with your own
   where they differ, and add the reversal condition for each. Then have someone push the follow-up
   column at you cold.

5. **Draw the whole-platform diagram.** One system, one signal flow, five layers, artifacts named
   on the layers they belong to. It must be a single diagram, not a collage. This is the front-door
   image for lesson 03.

6. **Write and rehearse all four compressions** (one line, 30 s, 3 min, 6–8 min). Record yourself
   at 30 seconds and listen for one thing only: does the honesty caveat sound like rigour or like
   an apology?

7. **Rehearse the four joints in isolation.** Say each transition sentence ten times. These are the
   sentences you will fumble under pressure, and they are the ones that make the argument cohere.

8. **Run the archetype check.** Name the archetype your chapter selection argues for, then verify
   every featured chapter supports it. Cut or reorder anything that does not.

**Acceptance:** a thesis sentence with its dependency chain · a selection verdict for all eleven
artifacts with a capped featured set · five six-beat arcs in speaking prose, each with a measured
number and a reversal condition · one whole-platform diagram · four rehearsed compressions · four
rehearsed joints · a named archetype whose evidence you can point at.

## Common pitfalls

1. **Writing the thesis after picking your favourite projects.** **Mechanism:** a retrofitted
   thesis does not constrain anything, so the joints do not line up and follow-up questions expose
   it — the tell is that no chapter was ever cut. **Symptom:** you cannot answer "what did you
   leave out, and why?" **Fix:** write the four verbs first, then let §6's filter decide.

2. **Treating the four verbs as section headers.** **Mechanism:** if every artifact "sort of fits"
   a verb, the verbs are doing no work and the narrative is a list with nicer labels. **Symptom:**
   an artifact is featured because it was hard, not because it answers a verb and moves a number.

3. **Opening with the safe chapter to warm up.** **Mechanism:** attention is highest in the first
   thirty seconds and declines; spending it on the exporter and saving the util lie for third
   inverts the attention curve. **Symptom:** the interviewer's follow-ups are all about your
   least interesting chapter, because that is the one they were listening hardest to.

4. **A collage instead of a diagram.** **Mechanism:** eleven project diagrams in sequence encode
   eleven systems, so the reader still has to do the synthesis. **Symptom:** you find yourself
   saying "and then this connects to that" while pointing between boxes that share no edge.

5. **Naming a tradeoff without naming the rejected option.** **Mechanism:** "I used Kueue" is a
   fact about tooling; "I chose Kueue over Volcano and Slurm because the control plane was already
   operator-driven, and I'd reverse it for an HPC-native tenant population" is a decision with a
   boundary. Only the second is evidence of judgement. **Symptom:** every technology in your story
   sounds like the only option you considered.

6. **Delivering the honesty caveat apologetically.** **Mechanism:** the caveat's job is to make the
   preceding claims credible; delivered as an apology it does the opposite and invites the
   interviewer to probe for more weakness. **Symptom:** your voice drops on "it was simulated."
   **Fix:** say the caveat *first* in the sentence — "on a 200-node simulated fleet, traces
   published, I measured…" — so it reads as a specification rather than a confession.

7. **Quoting a number without its basis and date.** **Mechanism:** GPU rates vary by more than 2×
   depending on basis, and utilisation figures vary by workload; an unqualified figure is your most
   attackable sentence. **Symptom:** "GPUs cost about $2.50 an hour" followed by "well, it
   depends…" when pushed. **Fix:** basis and date inline, every time.

8. **Letting the thesis drift between loops.** **Mechanism:** if you re-word the thesis per company
   you lose the compounding benefit of rehearsal and the debrief union fragments again.
   **Symptom:** the fourth loop feels no easier than the first. **Fix:** the thesis is fixed; only
   the *chapter emphasis* changes per employer, per lesson 01's matrix.

## Self-check

- **State the organising thesis and derive it, rather than reciting it.** *Answer:* "Making a
  multi-tenant GPU fleet observable, attributable, schedulable and survivable." The derivation is a
  dependency chain from the operator's position: you cannot attribute what you cannot honestly
  measure, because a device counter with no tenant map is a fact nobody can act on; you cannot
  schedule fairly what you cannot attribute, because allocation policy without a usage ledger is
  guesswork; and none of it matters if hardware failure takes the job down, which at fleet scale it
  measurably does — Meta reports a measured 7.9-hour MTTF for 1,024-GPU jobs (arXiv:2410.21680).
  The four verbs are ordered, not enumerated, and the order is the argument.

- **Why are chapters A, B and C the load-bearing three?** *Answer:* because together they form a
  complete argument with no missing step. A is a measurement finding (the flagship metric reports
  kernel residency, not work — 99% `GPU_UTIL` against 0.16 `SM_ACTIVE` and 0.011 tensor activity on
  one GPU in one second). B is the engineering response (join the honest counters to the kubelet's
  allocation record per pod, with a conservation identity that proves shares sum to exactly 1.000
  per physical GPU). C is the justification for building B rather than installing something (the
  incumbent's numerator is a whole-GPU request count, so the same card bills $2.10, $14.70, $0.00
  or $8.40 depending only on the sharing mode and one GPU-operator flag). Remove any one and the
  argument has a hole a single follow-up finds.

- **Which chapter do you lead with, and what backs it immediately?** *Answer:* the util lie
  (artifact 05), because it is contrarian, one sentence long, reproducible by the listener, and it
  proves a distinction only someone who has operated accelerators makes. Back it immediately with
  the attribution controller (04, shipped via the operator from 02) so the insight is paired with
  shipping credibility: insight alone reads as a blogger, shipping alone reads as an implementer,
  and the pair is what reads as a platform engineer.

- **Give three rejected alternatives with their reversal conditions.** *Answer:* (i) *Reclaim on
  `SM_ACTIVE` alone* — rejected because a paused-but-loaded serving replica reads `SM_ACTIVE ≈ 0`
  with tens of gigabytes resident, so I gate on `DCGM_FI_DEV_FB_USED`; I would revisit the ~2 GB
  floor per fleet since it depends on the smallest served model. (ii) *`avg_over_time × window` for
  GPU-hours* — rejected because it extrapolates over samples that do not exist and overstates by
  exactly `window ÷ time-present` (2.67× in my case), so I use `sum_over_time × Δ/3600`; I would
  reconsider the Δ-independent form if I could not pin the rule-group interval. (iii) *MIG
  everywhere* — rejected because static partitions do not rebalance and 7×`1g.10gb` strands about
  14% of the framebuffer; I would choose MIG wherever hard isolation between tenants is a
  requirement rather than a preference.

- **Why must the narrative be isomorphic to the system?** *Answer:* because every "why is that
  there?" question then has a structural answer instead of a chronological one. A story ordered by
  when you built things forces the listener to hold an arbitrary sequence; a story ordered by the
  system's own layering means each chapter's position *is* its justification — measurement under
  attribution under scheduling, all wrapped in survivability, on a substrate. It also makes the
  narrative reorderable under interruption: if someone jumps from chapter A to chapter C, the spine
  still holds, because the chapters are related by dependency rather than by date.

- **What does the honesty close contain, and why is it not modesty?** *Answer:* three specific
  limits — the time-slicing per-tenant split is an approximation with an explicit uncertainty flag,
  because the device exposes one counter rather than one per CUDA context and no code recovers N
  values from one; the fleet results come from a 200-node simulation with published traces, with
  one rented GPU-afternoon used to validate rather than develop; and every dollar figure is a dated
  snapshot on a named basis. It is not modesty because each limit is stated *with its mechanism*,
  which demonstrates that you know exactly where the boundary is. An interviewer's job is to find
  the boundary; arriving with it already mapped converts their strongest probe into your evidence.

- **How does the same spine serve a recruiter screen and a forty-five-minute portfolio review?**
  *Answer:* by selection, not truncation. The one-liner keeps only the specialisation and the one
  artifact; the 30-second version adds the thesis, chapter A, three numbers and the honesty caveat;
  the 3-minute version drops the thesis to a clause and runs one chapter through the full six-beat
  arc; the 6–8-minute version runs A → B → C → D → E with all four joints and keeps the substrate
  in reserve for probes. Because all four are selections from one fixed spine, rehearsal compounds
  across them instead of fragmenting into four separate pitches.

## Connections & what's next

This lesson consumes lesson 01's artifact-to-signal matrix and produces the spine everything
downstream runs on. Lesson 03 turns the storyboard, the chapter list and the whole-platform diagram
into a published repo README, an RFC-style design doc, and a brag-doc. Lesson 04 builds the
flagship blog post directly out of chapter A. Lesson 05's design drills reuse the rejected
alternatives bank as the tradeoff vocabulary you volunteer under a clock. Lesson 07 drills each
chapter's six-beat arc at three compressions. Lesson 08 reuses the same thesis as the backbone for
staff behavioural stories — the chapters become the Situations, and the rejected alternatives
become the Actions worth narrating.

Next: **lesson 03** takes the thesis, the chapters and the diagram and turns them into evidence a
hiring manager reads cold, with nobody in the room to explain it.

## References & further reading

**Primary sources**

- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: which fields ship enabled and which ship commented out; the one-file proof behind chapter A.
- NVIDIA `dcgm-exporter`, `grafana/dcgm-exporter-dashboard.json` — https://github.com/NVIDIA/dcgm-exporter/blob/main/grafana/dcgm-exporter-dashboard.json — read for: the vendor's eight-panel default dashboard with no SM-breadth panel (grafana.com #12239).
- OpenCost issue #3900 — https://github.com/opencost/opencost/issues/3900 — read for: the maintainers' own statement of the request-vs-usage gap, with the $2.50-vs-$0.275 expected/actual. Filed 2026-07-05, closed 2026-07-09.
- OpenCost issue #3828 — https://github.com/opencost/opencost/issues/3828 — read for: an unaffiliated user specifying chapter B's correction layer as a feature request.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: the measured 7.9-hour MTTF at 1,024 GPUs behind the fourth verb, and the fact that the larger-scale figures are projections.
- Will Larson, "Staff archetypes" — https://staffeng.com/guides/staff-archetypes/ — read for: the Tech Lead / Architect / Solver / Right Hand framework used as a chapter-selection filter in §7.

**Course-internal sources — where every number in this lesson comes from**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — field 203's exact semantics, the two collection paths, the batch-1 derivation (16 GB / 3.35 TB/s vs 16 GFLOP / 989 TFLOP/s), and why `GR_ENGINE_ACTIVE` is not the honest metric.
- `platform-eng/modules/05-gpu-observability/lessons/08-capstone-allocated-vs-utilised.md` — the `sum_over_time` integration, the six-bucket decomposition, the 57.3% idle share, and the synthetic ground-truth validation.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — the two-loop architecture, `map[string][]Owner`, the share model, identities A and B, and the 46.8% exposure fraction.
- `platform-eng/modules/11-gpu-cost-economics/lessons/09-existing-tooling-limits.md` — the OpenCost source trace and the $2.10 / $14.70 / $0.00 / $8.40 four-regime table.

**Deeper dives**

- Julia Evans, "Brag documents" — https://jvns.ca/blog/brag-documents — read for: the mechanic behind the one-page quantified brag-doc that lesson 03 produces; the relevant discipline is one metric per line.

**Not relied upon**

- Vendor product pages and marketing material describing unified fleet-management platforms were
  consulted for the "vendors organise the same way" observation in Perspectives. They are marketing
  artifacts, not technical documentation, so the point is stated as a general industry pattern
  rather than as a claim about any named product's internals.

[🎓 12 — Capstone & interview preparation](../README.md)
