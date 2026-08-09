---
id: "NN"
title: "Module title"
notion: "https://app.notion.com/p/<page-id>"
phase: "Phase X · Months A–B"
effort: "N weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []          # module ids that must come first, e.g. ["02b"]
unlocks: []                # what this gates, e.g. ["04","07"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# EMOJI NN — Module title

> **Goal.** One sentence, copied from the Notion callout.

- **Notion page:** https://app.notion.com/p/<page-id>
- **Phase:** … · **Est. effort:** …
- **Prerequisites:** … · **Unlocks:** …

## Objectives (must-know concepts)

Each objective maps to one lesson below.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Lesson title](lessons/01-slug.md) | `not-started` |

## Sources

Canonical reading lives on the Notion page. Local copies go under `resources/`.

## Checkpoint

The [checkpoint](checkpoint.md) is the real completion gate — answer from memory.

## Directory layout

| Path | What goes here |
|------|----------------|
| `lessons/` | One page per concept. |
| `practice/` | Code, benchmarks, commands. |
| `resources/` | Saved references, diagrams, papers. |
| `checkpoint.md` | Checkpoint answers. |
