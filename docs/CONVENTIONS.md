# Repo conventions

How this repository is structured, and the fields that make it trackable. Notion
holds the plan and progress; this repo holds the work. Every module page in Notion
links to its directory here, and every file here links back to Notion.

## Two levels

| Level | Answers | File |
|-------|---------|------|
| **Module** | "What must I master, and how do I know I'm done?" | `modules/<NN>-<slug>/README.md` |
| **Lesson** | "What's the one thing I'm learning now, and what did I build?" | `modules/<NN>-<slug>/lessons/<NN>-<slug>.md` |

A lesson maps to one **must-know concept** from the module's Notion page. Finish a
lesson in roughly one sitting (1–3 hrs).

## Mandatory fields

Both module and lesson files start with YAML frontmatter. It's mandatory because a
script (or a Notion sync) reads status from it without parsing prose.

**Module** — required: `id`, `title`, `notion`, `status`, `prerequisites`,
`unlocks`. `status` ∈ `not-started | in-progress | checkpoint-passed`.

**Lesson** — required: `lesson`, `title`, `module`, `concept`, `status`.
`status` ∈ `not-started | in-progress | done`. `concept` links the lesson up to a
module objective; `artifacts` links it down to `practice/`.

## Definition of done

- **Lesson is `done`** when *Why this matters* and *Practice* are filled and at
  least one artifact exists under `practice/`. No artifact → not done ("build, do
  not read").
- **Module is `checkpoint-passed`** when every lesson is `done` **and** every
  checkpoint question in `checkpoint.md` is answered from memory.
- On pass: set `completed`, flip `status`, and update the module's tracking row in
  Notion, linking the files here as evidence.

## The link chain

Notion module page → module `README.md` → lesson page → `practice/` artifact.
It runs both directions, so progress is never stranded in one place.

## Adding a new lesson

Copy [`docs/templates/lesson.md`](templates/lesson.md) into the module's
`lessons/` directory, name it `<NN>-<slug>.md`, fill the frontmatter, and add a row
to the module README's **Lessons** table.

## Naming

- Module dirs: `NN-slug` (sub-modules keep the letter: `01b-…`, `02b-…`).
- Lesson files: `NN-slug.md`, numbered in concept order within the module.
- Slugs: lowercase, hyphenated, no spaces.
