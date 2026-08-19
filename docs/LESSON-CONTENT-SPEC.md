# Lesson content specification

> **What changed and why.** The first pass of this course optimised for *coverage*:
> every lesson named the right concepts, framed why they matter, and pointed at the
> sources where the actual knowledge lived. Studying from it meant opening eight tabs
> per lesson. This spec replaces that model. A lesson is now a **self-contained
> textbook chapter**: you sit down with it, read it start to finish, and come away
> understanding the topic — without a single external lookup being *required*.
>
> The concepts and topic list are unchanged and are not up for revision. What changes
> is the depth and self-sufficiency of the content behind each one.

## The one rule

**A reader with the stated prerequisites must be able to learn the topic from the
lesson alone.** Every link in a lesson is optional enrichment. If understanding a
sentence requires clicking a link, the lesson is not finished — inline what the link
would have told them.

This replaces the old "lessons are short on purpose, go read the source" stance. It
does **not** replace *build, do not read*: the Practice section and its artifact are
still the completion gate. The lesson now has to actually teach you enough to do the
build, instead of handing you a reading list and wishing you luck.

## Section structure

The 12-section skeleton stays. What changes is the weight: the teaching sections carry
the lesson, the framing sections get you oriented and get out of the way.

| Section | Role | Target share |
|---|---|---|
| `## Where this fits` | Continuity from the previous lesson; what this unlocks. | ~3% |
| `## Why this matters` | The concrete cost of not knowing it. | ~3% |
| `## What's new here (calibration)` | What's assumed known vs genuinely new. | ~2% |
| `## Core concepts` | **The chapter.** Full teaching, from first principles. | **~55%** |
| `## Perspectives` | Same mechanism seen as developer / operator / hardware / economics. | ~5% |
| `## Real-world use cases` | Named incidents, issues, postmortems — with what each shows. | ~6% |
| `## Worked example` | One scenario driven end to end with real numbers/output. | ~12% |
| `## Practice` | The hands-on task + acceptance criteria. | ~6% |
| `## Common pitfalls` | Failure modes, each with the symptom *and* the mechanism. | ~4% |
| `## Self-check` | Questions with full answers — a revision surface. | ~3% |
| `## Connections & what's next` | Forward and backward wiring. | ~1% |
| `## References & further reading` | Optional depth, annotated. | — |

Length follows from the depth requirement; do not pad to hit a number. In practice a
finished lesson lands around **7,000–12,000 words**. If it comes in under ~6,000, the
Core concepts section is almost certainly still asserting things it should be
explaining.

## What "teaching" means here

The failure mode being corrected is the **unexplained assertion** — a true, dense
sentence that names a fact without transferring the understanding behind it.

> ❌ "The GPU has a finite number of counter-collection resources, and which fields
> can be collected together is architecture-dependent."

That is a fact to memorise. The reader still cannot reason about a case the sentence
does not cover. Replace it with the mechanism:

> ✅ "Streaming multiprocessors expose their performance counters through a fixed
> number of hardware collection slots per SM partition. A single PROF metric is not
> one counter — `PIPE_TENSOR_ACTIVE` is derived from tensor-pipe issue counters that
> physically live in the same counter bank as the SM-occupancy counters. Two metrics
> that read the same bank cannot both be programmed in one pass, because the bank
> holds one counter-select register at a time. That is why co-residency is a property
> of the silicon's counter-bank layout, and why it changes between generations: [table
> of V100 vs T4 vs A100 groupings]. `dcgmi profile -l` prints the groups for the
> hardware in front of you — metrics in different groups are the pairs that will
> multiplex."

Every load-bearing claim in a lesson should survive the question **"why is that
true?"** being asked once more than the reader expects.

Concretely, for each concept in `## Core concepts`:

1. **Name the problem** the mechanism exists to solve, before naming the mechanism.
2. **Explain how it works internally** — the steps, the data structures, the state
   transitions. Not "X handles Y", but how X handles Y.
3. **Show it.** A diagram, a code listing, a real command with real output, or a
   worked calculation. Prose alone is not enough for a mechanism.
4. **Give the numbers** with units and where they come from — bandwidths, latencies,
   limits, defaults, sizes. Ranges are fine; unsourced precision is not.
5. **State the failure mode** — what it looks like when this breaks, and why the
   symptom looks the way it does.

## Mandatory elements

### Diagrams — at least two per lesson

**ASCII art inside fenced code blocks.** Liquid is disabled site-wide and the Pages
theme does not render mermaid, so fenced ASCII is the only form that renders correctly
both on GitHub and on the built site. Do not use mermaid, HTML, or images.

Draw the thing that is hard to hold in your head: the layout, the flow, the state
machine, the timeline, the hierarchy. Label the edges. Diagrams that merely restate a
bullet list are not worth the space.

```
  Prometheus                nv-hostengine (standalone)          GPU
  ──────────                ──────────────────────────          ───
   scrape ──15s──▶ dcgm-exporter ──DCGM API──▶ field cache ◀──1s── profiling module
                    (embedded off)              │                     │
                                                │            programs HW counters
                                     watches: field group           │
                                     updateFreq 1s                  ▼
                                     maxKeepAge 300s        [ counter bank 0 ]
                                                            [ counter bank 1 ]
```

Aim for at least one **structural** diagram (what is connected to what) and one
**temporal or causal** diagram (what happens in what order, or what leads to what).

### Inlined source knowledge

Facts that currently live behind a link must be **in the lesson**. That includes:

- Field / flag / metric tables reproduced as markdown tables.
- The actual algorithm, described step by step — not "uses a ring algorithm" but the
  ring's construction, the 2(N−1) steps, and the bandwidth term that falls out.
- Real default values, real limits, real version numbers, real CLI syntax.
- The substance of a cited postmortem or issue: what happened, what the numbers were,
  what the fix was. Not just its title.

Attribute inline where a fact is specific and checkable — `(DCGM 3.3 user guide)`,
`(NCCL 2.19 release notes)`, `(Meta OPT-175B logbook, 2022)`. Keep the attribution
short; the full citation belongs in References.

**Do not copy text verbatim from sources.** Reproduce *facts, tables, and numbers* —
those aren't copyrightable — and write the explanation in your own words. Never lift
paragraphs of prose. See `docs/EXTERNAL-DEPTH.md` for the standing policy on this.

### Accuracy

Research before writing. Verify every checkable claim — flags, versions, defaults,
limits, prices, spec numbers — against a current primary source. Where a number is
genuinely variable (hardware generation, cloud region, workload), say so and give the
range and the thing it depends on, rather than inventing a precise figure.

If a source contradicts the existing lesson text, the source wins: fix the lesson and
note the correction in the References entry.

Never invent: benchmark results, incident details, issue numbers, URLs, quotes, or
version-specific behaviour. An honest "this varies by driver version; check with
`nvidia-smi -q`" beats a confident fabrication.

## Strongly encouraged

- **Full code and configs** — complete, runnable listings rather than fragments.
  Real YAML with every field that matters, real CSV, real Go with imports and error
  handling, real PromQL that would actually evaluate. Annotate the load-bearing lines.
- **Commands with their output** — show the invocation *and* a realistic transcript,
  then read the output line by line. Mark clearly if a transcript is representative
  rather than captured.
- **Worked math** — derivations step by step, units carried through, assumptions
  stated: bandwidth budgets, cardinality counts, cost-per-million-tokens, latency
  arithmetic, capacity planning. The reader should be able to re-run the calculation
  with their own inputs.
- **Comparison tables** where a choice exists: options as rows, the properties that
  decide between them as columns.

## Preserved without change

- **YAML frontmatter** — keep every existing key and value. Update `sources` to the
  real count of entries in References if it drifts.
- **The `# NN.M · Title` heading and the `> **Concept.**` blockquote** beneath it,
  including the module and deliverable links.
- **Filenames, `prev`/`next` chains, and all relative links.** Relative links must
  keep resolving on the built Pages site — directory links need the explicit
  `README.md` suffix.
- **The lesson's topic and scope.** Deepen what is there. Do not re-scope a lesson,
  merge lessons, or add new ones.

## Voice

Second person, direct, technical. The existing register — a senior engineer explaining
to a peer — is right; keep it. Assume intelligence, do not assume knowledge. No
filler, no hype, no "in today's fast-paced world". Bold sparingly, for the sentence
that is the actual takeaway.

Never claim the reader has done something they may not have ("as you saw when you ran
this last week"). Practice sections describe work to do, in the imperative.

## Definition of done for a rewritten lesson

- [ ] Readable start to finish with zero external lookups required.
- [ ] Every mechanism explained, not just named — survives one extra "why?".
- [ ] ≥2 ASCII diagrams in fenced code blocks, at least one structural and one
      temporal/causal.
- [ ] The substance behind every cited source is inlined; links are optional depth.
- [ ] All checkable facts verified against a current primary source.
- [ ] Numbers carry units and provenance; no invented precision.
- [ ] Frontmatter, title, concept blockquote, and all links preserved and resolving.
- [ ] Self-check answers are complete enough to revise from without rereading.
