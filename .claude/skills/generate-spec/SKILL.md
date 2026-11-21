---
name: generate-spec
description: Generate the project specification as three files - specification/VISION.md, ARCHITECTURE.md and ROADMAP.md - in the matrix-agora format (FORMAT.md and examples/ next to this skill), from the source documents (a concept document, drafts, notes) and the real code. Maps every source statement to a place, asks the owner about real gaps instead of inventing, keeps phase ids already in use. Writes the files for review; does not commit.
---

# Skill: Generate Specification

Turn the project's source documents into a specification of **three files** in `specification/`:

- `VISION.md` — why and for whom: principles, non-goals, glossary;
- `ARCHITECTURE.md` — how it works: components, mechanisms, contracts, configuration, security, testing;
- `ROADMAP.md` — in what order: versions and phases `vA.B`, each with Goal, Tasks, DoD and Tests.

The format is fixed by [FORMAT.md](FORMAT.md), and [examples/](examples/) shows it filled in: the first
version of the matrix-agora specification. Use the examples for structure, depth and tone only; never copy
their content.

To add a version or phase to a specification that already exists, use `/plan-phase` instead.

## Usage

```
/generate-spec [source-file …]
```

- `/generate-spec`: the sources are every Markdown file in `specification/` except `VISION.md`,
  `ARCHITECTURE.md`, `ROADMAP.md` and the `implementation/` and `history/` folders.
- `/generate-spec specification/hanoi-vr-concept.en.md`: use the named file(s) only.

## Instructions

### Step 0: Read the inputs

1. **The format:** read [FORMAT.md](FORMAT.md) and the three files in [examples/](examples/).
2. **The sources.** Read every source document in full. When a document exists in two languages (for
   example `hanoi-vr-concept.md` and `hanoi-vr-concept.en.md`), read the English one and use the other only
   to resolve unclear wording.
3. **Project context:** `CLAUDE.md` and `README.md`.
4. **The real code**, if any exists. What is built is ground truth for the parts already done: names,
   files, settings, behavior.
5. **Delivery state:** `git tag -l 'v*'`, the files in `specification/implementation/`, and any phase ids
   used in `CLAUDE.md` or `.claude/skills/`. Phase ids already in use (released, or with an issues file or
   GitHub issues) **keep their numbers** in the new ROADMAP.
6. **Existing specification:** if any of `VISION.md`, `ARCHITECTURE.md`, `ROADMAP.md` exists, stop and ask:
   regenerate all three (overwrite), or keep them and use `/plan-phase`.

### Step 1: Map every source statement

Build a mapping table (kept for the report, not written into the spec):
`source § | statement (short) | target file § | note`.

- Every statement in the sources lands in exactly one place, by the table in FORMAT.md §The three files.
- Instructions addressed to Claude Code (how to work, when to stop, how to report) stay in `CLAUDE.md`; mark
  them `CLAUDE.md` in the table and check that `CLAUDE.md` already holds them.
- A statement that is dropped (obsolete, contradicted by the code) is listed with the reason. Nothing is
  lost silently.

### Step 2: Find the gaps and ask

List what the format needs and the sources do not say: a principle the design clearly follows but nobody
wrote down, the threats and their protections, configuration defaults, the test commands, the line between
two versions, the DoD of a phase.

- **Derivable** from the sources or the code: fill it, and note the source in the mapping table.
- **A decision** (it changes scope, order or design): ask the owner, in one batch, before writing. Use
  AskUserQuestion with concrete options where they exist.
- **Not answered:** do not invent it. Leave it out of the spec and list it under "Open questions" in the
  report.

### Step 3: Cut the roadmap into versions and phases

- **Keep the ids in use** (Step 0, item 5). New ids continue from them.
- **A phase** is one releasable slice with its own DoD that can be tested on its own (FORMAT.md §ROADMAP.md).
  Phases and steps named in the sources are the starting point; split or merge only when a phase would not
  be testable on its own, and say so in the report.
- **A version** groups phases toward one goal. The first version proves the setup end to end with the least
  logic; complexity is added by version.
- **The DoD** comes from the sources' acceptance or completion criteria. Items that need the owner or the
  live system (a device, an account, a manual playthrough) are marked `(Manual, owner)`.
- **Status:** a phase with a release tag is `completed — released vA.B.0 on <tag date>`; a phase with an
  issues file but no tag is `in progress`; the rest are `planned`.

### Step 4: Write the three files

Write them in this order, so each can refer to the ones before it: `VISION.md`, `ARCHITECTURE.md`,
`ROADMAP.md`. Follow FORMAT.md section by section.

- English; no emoji; no secrets or private data.
- ROADMAP refers to ARCHITECTURE sections by name (`ARCHITECTURE §Board`) instead of repeating mechanisms.
- ARCHITECTURE §Contracts lists every interface, file format, setting name and ordering rule that later
  phases rely on, each with the test that pins it (or the phase that will add that test).
- ARCHITECTURE §Testing and CI takes the gates from `CLAUDE.md` when it defines them, so the two never
  disagree.

### Step 5: Self-check

Before reporting, check and fix:

- every row of the mapping table is placed or listed as dropped with a reason;
- every phase has Status, Goal, Tasks, DoD and Tests, and every DoD item is either automated or marked
  `(Manual, owner)`;
- every `ARCHITECTURE §Section` that ROADMAP or VISION names exists;
- every contract names its pinning test or the phase that adds it;
- VISION holds no mechanism, ARCHITECTURE no schedule, ROADMAP no re-explained mechanism;
- the ROADMAP Status table has one row per phase, in file order;
- the phase ids in use (Step 0, item 5) are unchanged;
- no emoji anywhere.

### Step 6: The source documents and their references

1. **Ask what happens to the sources.** matrix-agora moved its superseded `SPEC.md` to `specification/history/`
   and froze it ("kept for history only, never read as a source of truth"). Offer that, or keeping the
   sources as living documents next to the spec. Do not move anything without the owner's answer.
2. **Find what still points at the sources:** `grep -rn` for the source file names in `CLAUDE.md`,
   `README.md` and `.claude/skills/`. List every hit. These references (the delivery skills' phase lookup,
   contracts and scope fence) must be repointed to the three files once the owner accepts the spec;
   matrix-agora did that in the same commit as the restructure. Offer to do it; do not edit them without
   the owner's confirmation.

### Step 7: Report

Show the owner:

- the three files with their line counts;
- the versions and phases (id, title, status), and any split or merge made in Step 3 with the reason;
- the mapping coverage: statements placed, moved to `CLAUDE.md`, dropped (with reasons);
- the open questions from Step 2;
- the decision needed on the sources and the references to repoint (Step 6).

Do **not** commit. The owner reads and edits the files first. Suggest the commit message
`docs: restructure the specification into VISION/ARCHITECTURE/ROADMAP`.

## Important Rules

- **Three files, one format.** FORMAT.md is the rule; the examples show it filled in. Never copy the
  examples' content.
- **Nothing lost, nothing invented.** Every source statement is placed or dropped with a reason; a gap is
  asked about or listed as open, never filled with a guess.
- **One home per fact.** Mechanisms in ARCHITECTURE, scope in VISION, order in ROADMAP.
- **Keep ids in use.** Released phases and phases with issues keep their `vA.B` numbers.
- **The code is ground truth** for what is already built.
- **No emoji, no secrets.**
- **Don't commit, don't move sources, don't repoint references** without the owner's confirmation.
