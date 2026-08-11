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
sources: 7
---

# Capstone synthesis: thread 11 artifacts into one platform

## Where this fits
Lesson 01 mapped the terrain: a per-company loop map, an artifact-to-signal matrix naming which
of your 11 artifacts is the killer proof for which employer, and a personal sequencing plan for
which loops to run first. That lesson answered *where* to point your portfolio. It deliberately
did not answer a harder question: what do you actually *say* when you're in the room and an
interviewer asks "walk me through your background"? Eleven artifacts pointed at eleven different
employers is still eleven disconnected answers. This lesson fills that gap — it takes the same 11
artifacts and threads them into **one coherent platform narrative**, a single story you tell the
same way whether you're at CoreWeave's system-design round or Nebius' domain deep-dive, that
happens to have different chapters emphasized for different audiences. Everything from lesson 03
(the written portfolio) through lesson 09 (the mock loop) assumes this narrative already exists;
this is the lesson where you build it.

## Why this matters
Eleven artifacts scattered across eleven modules read as eleven hobby projects. The same eleven
artifacts, threaded onto a single spine, read as **one engineer who built a reference GPU
platform** — and that is a staff-level signal no individual project can carry alone. Interviewers
at CoreWeave and Nebius are explicitly hiring "area owners" who deliver "measurable improvements
across services." A coherent platform narrative is the artifact that proves you think in *systems
and services*, not tasks.

The difference between senior and staff in a portfolio review is almost never more projects. It is
a through-line: a single thesis, tradeoffs you can name (including what you rejected), real
numbers, and honesty about failure. A candidate who says "here are my six GPU projects" is senior.
A candidate who says "I set out to make a multi-tenant GPU fleet observable, attributable,
schedulable, and survivable, and here are the five chapters that prove each" is staff — because
they've imposed an operating thesis on their own work and can defend the joints between chapters.

That act — imposing structure on a pile of your own past work rather than doing more work — is not
a packaging trick bolted onto real engineering. It is itself a rehearsal of a genuine staff
competency: staff engineers are routinely the people who look across scattered, seemingly
unrelated problems across an org and recognize the shared shape underneath them, then organize
work (and teams, and roadmaps) around that shape instead of chasing each problem individually.
Doing that to your own 11 artifacts before anyone asks you to is a low-stakes, fully-controlled
place to practice exactly the move you'll need to make on the job.

This lesson designs that narrative once, so every loop in lesson 01 draws from the same
well-rehearsed story instead of improvising a fresh pitch per company.

## What's new here (calibration)
- **Skip** (you own it): portfolio-101, how to write a resume bullet, generic "tell a story"
  advice, the brag-document mechanic itself.
- **New**: the *organizing thesis* that turns 11 artifacts into one platform, and the value-chain
  mapping where each artifact answers one specific fleet-operator question.
- **New**: the concrete staff-vs-senior delta for *this* portfolio — which 3-5 chapters to
  feature, which real numbers to lead with, which tradeoffs to name as rejected alternatives.
- **New**: a lens for checking your own narrative — which staff *archetype* (Tech Lead, Architect,
  Solver, Right Hand) your thesis is implicitly arguing you are, and whether your chapter
  selection actually supports that claim.
- **New**: the single highest-signal artifact to open with, and the storyboard you'll fill into
  the capstone README.

## Core concepts

### The organizing thesis
**"Making a multi-tenant GPU fleet observable, attributable, schedulable, and survivable."** Those
four verbs are the spine. Every chapter hangs off one of them. Say this sentence first in any
portfolio review; it frames all eleven artifacts as one deliberate program of work.

This isn't invented pedagogy — it's how real GPU-cloud vendors sell the exact same idea as a
product. CoreWeave's Mission Control, for instance, is explicitly positioned as a single
"operating standard" that unifies observability, automated fleet/node lifecycle management,
straggler/anomaly detection, and operational tooling into one coherent system rather than a pile
of standalone dashboards — the same observable → attributable/actionable → survivable shape this
lesson's thesis argues for, just sold as a product instead of told as a career narrative. If a
real vendor organizes an entire product line this way, imposing the same structure on your own
work is not artificial packaging — it's the correct way an experienced platform engineer actually
thinks about a fleet.

### The value chain — each artifact answers one fleet-operator question

| Fleet-operator question | Chapter artifacts |
|---|---|
| Who's using my GPUs & what's it costing? | 01 gpu-cost-exporter, 02 gpu-cost-operator, 03 efficiency/cost report, 04 per-pod attribution, 11 FOCUS synthesis |
| Are my dashboards telling the truth? | 05 "your GPU dashboard is lying" — util vs MFU/goodput (the sharpest angle) |
| How do I share scarce GPUs fairly? | 06 Kueue showback |
| What does a unit of work cost? | 07 cost-per-million-tokens, 10 capex-vs-cloud |
| What happens when hardware fails? | 08 survive-a-failure lab, 04 failure-mode log |
| What's the machine underneath? | 01b container anatomy, 02b topology teardown, 09 network read, 10 KTHW/etcd |

Note the four verbs of the thesis map cleanly: observable (row 1-2), attributable (row 1, 3),
schedulable (row 3-4), survivable (row 5) — with "the machine underneath" (row 6) as the substrate
proving depth.

### What makes it read STAFF, not senior
Do **not** present all eleven exhaustively — that reads as a list. Feature **3-5 standout
chapters**, at least one at production/fleet scale, and for each cover the full arc: **problem →
your ownership → architecture → stack → metrics → post-launch evolution.**
- **Real numbers, not adjectives.** "The dashboard read 95% GPU utilization while MFU was 31%"
  beats "improved observability." Numbers are the staff tell.
- **Tradeoff narration — name what you rejected.** GPU sharing: MIG vs time-slice vs full
  passthrough, and why you chose what you chose. Cost model: chargeback vs showback. Scheduling:
  Kueue vs Volcano vs Slurm. Staff engineers are defined by the alternatives they can articulate
  rejecting.
- **Failure honesty.** The failure-mode log (04) and survive-a-failure lab (08) are not weaknesses
  to hide — they are operational-maturity signals. Leading with "here's how the fleet breaks and
  how I designed for it" is a staff move.
- **A single through-line thesis** stated once and returned to. It's the connective tissue between
  chapters.

### Which staff archetype is your thesis arguing you are?
Will Larson's staff-archetypes framework names four distinct shapes the "staff engineer" title
covers: the **Tech Lead** (guides one team's approach and execution), the **Architect**
(responsible for direction, quality, and approach across a critical area), the **Solver** (digs
into a gnarly cross-cutting problem, resolves it, moves to the next), and the **Right Hand**
(extends a leader's attention and judgment across a large, complex org). Your four-verb thesis is
implicitly arguing you're mostly an **Architect** — you defined the direction and approach for an
entire fleet-management domain, end to end — with a strong secondary claim to **Solver** in the
chapters where you dug into one gnarly problem (the util-vs-MFU dashboard lie, a live failure
scenario) and resolved it cleanly. That's a useful check on your chapter selection: if a candidate
chapter doesn't support "defined the approach for a critical area" or "solved a hard, bounded
problem," ask whether it belongs in your featured 3-5 at all, or whether it's better left as
supporting depth.

### The artifact to LEAD with
**"Your GPU dashboard is lying" (05).** It is contrarian, memorable, and instantly proves
GPU-fleet thinking — the util-vs-MFU/goodput gap is a distinction only someone who has actually
operated accelerators makes. It hooks the interviewer in one sentence. Back it immediately with
**gpu-cost-operator (02)** — "and I ship custom operators against a real fleet, not just
dashboards" — which pairs the contrarian insight with production-shipping credibility. Insight +
shipping = staff.

### The whole-platform view
The value chain isn't a list of projects; it's a stack. Exporter/operator emit signals →
attribution and cost models consume them → showback/scheduling act on them → the survivability
layer protects the whole thing → all of it sits on the machine substrate (topology, network,
etcd). Your one architecture diagram should show that flow, not eleven disconnected boxes.

## Perspectives

**Candidate's view — why one thesis beats 11 disconnected projects.** From your side of the table,
the temptation is to show range: eleven projects prove you touched eleven different corners of GPU
infrastructure. But range without organization reads as scattered, not broad — an interviewer
skimming eleven bullet points has no way to tell whether you understand how they relate or just
happened to build a lot of things. A single thesis converts the same eleven artifacts from "a lot
of projects" into "evidence of one deliberate, multi-year program" — the same facts, a
fundamentally different read.

**Reviewer's view — what a hiring manager's four-minute skim actually optimizes for.** A resume
and portfolio review is not a careful read; it's a fast scan for a small number of signals: one
sentence that tells them what kind of engineer you are, one number that proves you operated at
real scale, and one diagram that shows you can hold a whole system in your head at once. A
reviewer skimming eleven separate READMEs has to do the synthesis work themselves — and most
won't; they'll remember whichever project happened to load first. A reviewer handed a thesis
sentence, a real number, and one architecture diagram does your synthesis work for them, which is
exactly what gets you past the four-minute skim into an actual conversation.

**Systems-architecture view — the narrative should be isomorphic to the system.** Notice that this
lesson's four-verb thesis — observable, attributable, schedulable, survivable — is not just a
mnemonic device, it's a real layered-systems shape: signals flow up from raw telemetry, get
consumed by attribution/cost logic, drive scheduling/allocation action, and sit underneath a
survivability layer that protects the whole stack. That's the same "layers, not a junk drawer"
principle you'd apply to any well-designed distributed system. Telling your story with the same
structure the system actually has isn't a coincidence — a narrative shaped like the system it
describes is easier to believe, and easier to defend under follow-up questions, than one that
imposes an arbitrary chronological order onto the same facts.

**Staff-vs-senior signal view — imposing structure IS the competency, not a proxy for it.** It's
tempting to treat capstone synthesis as marketing distinct from "real engineering," but the skill
this lesson drills — scanning a body of scattered-looking work and recognizing (or building) the
shared thesis underneath it — is close to verbatim what senior staff engineers describe doing on
the job: watching for the same problem recurring in different disguises across teams, and only
then organizing effort around the pattern instead of the individual instances. Practicing that
discipline on your own eleven artifacts, where the stakes are low and you control all the
evidence, is direct rehearsal for the harder version of the same skill you'll need once you're
inside an organization with far messier, far less controllable inputs.

## Real-world use cases
- **Lalit Maganti (Perfetto/Chromium engineer), "How I Find Problems to Solve as a Staff
  Engineer"** — https://lalitm.com/post/find-problems-staff-engineer/ — what it shows: a real
  staff engineer's first-person account of imposing an organizing thesis on scattered
  observations and work over time, rather than treating each problem as an isolated task — the
  direct real-world parallel to this lesson's four-verb-thesis move.
- **CoreWeave Mission Control product page** —
  https://www.coreweave.com/mission-control — what it shows: a real GPU-cloud vendor organizing
  observability, automated fleet/node lifecycle, and anomaly detection into one narrative product
  rather than separate tools — proof the "value chain, not a list of features" framing used in
  this lesson isn't invented pedagogy.
- **Sean Goedecke, "How I got promoted to staff engineer twice"** —
  https://www.seangoedecke.com/staff-engineer-promotions/ — what it shows: a real engineer's
  account of building a coherent promotion narrative out of scattered project work, and the
  observation that being known and valued for a clear body of work matters more than raw technical
  strength alone — directly analogous to building a portfolio narrative for external hiring.
- **Will Larson, "Staff archetypes"** — https://staffeng.com/guides/staff-archetypes/ — what it
  shows: the canonical Tech Lead / Architect / Solver / Right Hand framework used above to check
  which archetype your thesis is implicitly arguing you are.

## Worked example
A 30-second staff-signal opener, rehearsed: *"I set out to make a multi-tenant GPU fleet
observable, attributable, schedulable, and survivable — one reference platform, not scattered
scripts. The chapter I'll open with is counterintuitive: your GPU dashboard is lying to you. On a
real workload the dashboard read 95% utilization while model-FLOPs-utilization was 31% — the GPU
was busy spinning, not computing. I built an exporter and a custom Kubernetes operator to surface
goodput and MFU, not just util, then hung per-pod cost attribution off it — FOCUS-aligned so
finance could actually read it. I chose showback over hard chargeback because in a scarce-GPU
multi-tenant setting chargeback creates hoarding; I rejected Volcano and raw Slurm for Kueue
because it's Kubernetes-native and I was already operating an operator-driven control plane. And I
designed the whole thing assuming hardware fails — I keep a failure-mode log and ran a
survive-a-failure training lab, because on a 250k-GPU fleet a node dies every few minutes."* That
paragraph names the thesis, leads with the contrarian chapter, cites a real number, narrates two
rejected alternatives, and closes on failure maturity — the full staff delta in one breath.

If pressed on "so what kind of engineer are you, exactly," the archetype check gives you a crisp
follow-up: *"Mostly I'd say Architect — I defined the approach for this whole fleet-management
area end to end — with some Solver mixed in on the specific chapters, like the dashboard
investigation, where I dug into one gnarly, bounded problem and resolved it."* That answer only
works because the chapters you chose actually support it — which is exactly what the archetype
check in Core concepts is for.

## Practice
Build the storyboard in
[GPU platform capstone](../practice/gpu-platform-capstone/README.md):
1. Write the **through-line sentence** (start from the four-verb thesis; make it yours) at the top
   of the capstone README.
2. List your **chapters** in value-chain order, each tagged with the fleet-operator question it
   answers and the artifact IDs behind it.
3. Pick your **3-5 featured chapters** (≥1 production-scale) and for each write the
   problem→ownership→architecture→stack→metrics→evolution arc, with at least one real number.
4. For each featured chapter, write the **rejected alternative(s)** you'll name (MIG/time-slice/
   passthrough, chargeback/showback, Kueue/Volcano/Slurm).
5. Draw **one architecture diagram of the whole platform** showing signals flowing
   exporter/operator → attribution/cost → showback/scheduling → survivability → machine
   substrate.
6. Write and rehearse your **30-second opener** leading with "dashboard is lying" (05) backed by
   gpu-cost-operator (02).
7. Name which **staff archetype** (Tech Lead / Architect / Solver / Right Hand) your thesis and
   chapter selection argue you are, and check that every featured chapter actually supports that
   claim — cut or reorder any that don't.

## Common pitfalls
1. **Writing the thesis after picking your favorite projects, instead of letting the thesis pick
   the chapters.** Reviewers can tell when a narrative is retrofitted onto pre-chosen favorites —
   the joints don't quite line up, and follow-up questions expose that the "thesis" doesn't
   actually constrain anything. Write the four-verb thesis first, then let it decide which
   chapters earn a featured slot and which get cut.
2. **Treating the four verbs as decorative section headers instead of a real test.** "Observable,
   attributable, schedulable, survivable" isn't a formatting scheme — every artifact you feature
   should have to earn its place by clearly answering one of the four, and an artifact that only
   loosely fits should be cut or moved to supporting material, not kept because it's a favorite.
3. **Opening with the least contrarian chapter out of instinct to "warm up."** Leading with a safe,
   unsurprising chapter and saving the strongest material for later loses the hook that makes an
   interviewer lean in. "Your GPU dashboard is lying" works precisely because it's the *first*
   thing said, not the third.
4. **Confusing "one architecture diagram" with "one diagram per artifact, stitched together."**
   Eleven small diagrams taped into a sequence still reads as eleven projects. The whole-platform
   diagram has to be drawn as a single system with one signal flow, not a collage of the
   individual artifacts' own diagrams.
5. **Naming a tradeoff without naming what you actually rejected.** "I used Kueue for scheduling"
   is a fact; "I chose Kueue over Volcano and raw Slurm because it's Kubernetes-native and I was
   already running an operator-driven control plane" is a staff signal. The rejected alternative is
   the part that proves judgment, not just familiarity with one tool.

## Self-check
- What is the organizing thesis that turns 11 artifacts into one platform, and what are its four
  verbs?
  **Answer:** "Making a multi-tenant GPU fleet observable, attributable, schedulable, and
  survivable" — observable, attributable, schedulable, survivable, each verb anchoring a chapter
  cluster.
- Which single artifact should you lead a portfolio review with and why, and what backs it?
  **Answer:** "Your GPU dashboard is lying" (05) — the contrarian util-vs-MFU/goodput angle
  instantly proves GPU-fleet thinking; back it with gpu-cost-operator (02) to pair the insight with
  production-shipping credibility.
- Name three tradeoff pairs you should narrate as rejected alternatives to read staff, not senior.
  **Answer:** GPU sharing (MIG vs time-slice vs full passthrough), cost model (chargeback vs
  showback), and scheduling (Kueue vs Volcano vs Slurm).
- Which staff archetype does the four-verb thesis mainly argue you are, and what's the secondary
  claim?
  **Answer:** Mainly Architect — you defined the direction and approach across an entire critical
  area (fleet management) — with a secondary claim to Solver in chapters where you resolved one
  specific, gnarly, bounded problem (the dashboard investigation, a live failure scenario).
- Why is "one thesis" a stronger signal to a reviewer than "eleven well-written project READMEs,"
  even if the underlying work is identical?
  **Answer:** A reviewer's skim is fast and optimizes for a single organizing sentence, one real
  number, and one diagram; eleven separate READMEs force the reviewer to do the synthesis work
  themselves (most won't), while one thesis does that synthesis for them and gets you past the
  skim into an actual conversation.

## Connections & what's next
This lesson depends directly on lesson 01: the artifact-to-signal matrix built there tells you
which chapters to emphasize when — the same core thesis stays constant, but which 3-5 chapters you
feature can shift slightly toward whichever employer's killer-artifact mapping is most relevant
for that loop. It also sets up the rest of the module: lesson 03 turns this storyboard into a
written repo README and RFC-style design doc; lesson 04 builds the flagship blog post directly out
of the "dashboard is lying" opening chapter; lesson 07 (narrate your artifacts) drills telling each
chapter's problem→ownership→architecture→stack→metrics→evolution arc out loud in under three
minutes; and lesson 08 (staff behavioral) reuses this same thesis as the backbone for STAR
stories. Next: **lesson 03** takes the through-line thesis, the chapter list, and the
whole-platform diagram this lesson produced and turns them into the actual repo README and design
doc a hiring manager will read cold.

## References & further reading

**Primary sources**
- CoreWeave Mission Control product page: https://www.coreweave.com/mission-control
- Will Larson, "Staff archetypes" (Tech Lead / Architect / Solver / Right Hand framework): https://staffeng.com/guides/staff-archetypes/

**Real-world engineering blogs**
- Lalit Maganti, "How I Find Problems to Solve as a Staff Engineer": https://lalitm.com/post/find-problems-staff-engineer/
- Sean Goedecke, "How I got promoted to staff engineer twice": https://www.seangoedecke.com/staff-engineer-promotions/
- CoreWeave Kueue engineering blog (showback/scheduling chapter): https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads

**Deeper dives**
- Julia Evans, brag documents: https://jvns.ca/blog/brag-documents
- fonzi.ai, "How to Build an Engineering Portfolio That Gets You Hired": https://fonzi.ai/blog/portfolio-for-engineer

[🎓 12 — Capstone & interview preparation](../README.md)
