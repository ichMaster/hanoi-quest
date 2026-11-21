---
name: plan-phase
description: Add a new version or phase to an existing specification (specification/ROADMAP.md, with the matching ARCHITECTURE.md and VISION.md updates), from the owner's description or a feature draft, in the matrix-agora format (generate-spec/FORMAT.md). Places it in dependency order, never renumbers released phases, checks it against the principles and non-goals, records the owner's decisions. Can also reorder planned phases. Writes the changes for review; does not commit.
---

# Skill: Plan Phase

Add planned work to the specification: a **new phase** inside an existing version, or a **new version** with
its phases. The ROADMAP gets the phase in the format of
[generate-spec/FORMAT.md](../generate-spec/FORMAT.md); ARCHITECTURE and VISION get what the new work changes
in them. The same skill reorders phases that are still planned.

The specification must already exist. If `specification/ROADMAP.md` is missing, stop and suggest
`/generate-spec`.

## Usage

```
/plan-phase <what> [--new-version | --in vA | --after vA.B | --before vA.B]
```

- `<what>`: the work in words, or a path to a feature draft (for example
  `specification/features/hand-tracking.md`).
- `--new-version`: the work becomes a new version `vA` with one or more phases.
- `--in vA`: a new phase at the end of version `vA`.
- `--after vA.B` / `--before vA.B`: place it there (only among phases that are still planned).
- No placement flag: propose one (Step 2) and ask.

Examples: `/plan-phase "sound effects for grab and release" --in v0` ·
`/plan-phase specification/features/hand-tracking.md --new-version` ·
`/plan-phase "move v1.2 before v1.1"`

## Instructions

### Step 0: Read the inputs

1. [generate-spec/FORMAT.md](../generate-spec/FORMAT.md), and the example
   [ROADMAP.md](../generate-spec/examples/ROADMAP.md) for the phase format.
2. `specification/VISION.md`, `ARCHITECTURE.md` and `ROADMAP.md` in full.
3. The feature draft, if one was given.
4. The code the new work touches, and the "Architecture impact" notes in
   `specification/implementation/*code-review*.md`: the plan builds on what is really there.
5. **Delivery state:** `git tag -l 'v*'`, the issues files in `specification/implementation/`, and open
   GitHub issues if a remote exists. A phase with a release tag, an issues file or GitHub issues is
   **fixed**: its id never changes.

### Step 1: Check the scope

Compare the work with VISION §Principles and §Non-goals.

- **It contradicts a non-goal or a principle** (for example hand tracking when VISION lists "hand tracking
  without controllers" as a non-goal): stop and ask the owner. If the owner confirms the change of scope,
  the VISION edit (remove or narrow the non-goal, with the reason) is part of this plan.
- **It needs a later or unplanned capability first:** say what it depends on, and plan that first or ask.
- **It is too big for one phase** (it would not be testable on its own, or has more than about 8 tasks):
  propose a split into phases, or a new version.

### Step 2: Place it

- **New phase in a version:** the next free `B` in that version, after its last phase, unless `--after` /
  `--before` says otherwise.
- **New version:** the next free `A`. Matrix-agora also inserted versions between planned ones at the
  owner's request ("put phase 8 before 6"); that is allowed only when every phase that moves is still
  planned.
- **Reordering planned phases** (the owner asks, or the placement requires it): renumber only phases that
  are planned and have no issues file or GitHub issues, and update every reference to them: the ROADMAP
  Status table and cross-references, ARCHITECTURE phase tags `(from vA.B)` and the configuration table's
  Phase column, `CLAUDE.md`.
- **Fixed phases never move.** If the requested order would move one, say so and ask.

Without a placement flag, propose the placement with the reason (dependencies, size) and ask before writing.

### Step 3: Ask for the decisions the phase needs

List what the description or the draft leaves open and that changes the design or the DoD (which approach,
which defaults, what the owner will check by hand). Ask in one batch, with concrete options. Record the
answers in the phase as `**Owner decisions** (<today>): …`. Anything still open is listed in the report,
never filled with a guess.

### Step 4: Write the ROADMAP changes

- **New version:** a `## vA — Name: what it delivers` section with its paragraph and "Depends on:", and the
  version added to the header paragraph's list ("N self-contained versions, built in order: …").
- **Each new phase:** `### vA.B — Title` with the fields of FORMAT.md in order: `**Status:** planned`,
  `**Goal:**`, the description pointing to ARCHITECTURE sections, `**Owner decisions**` (if any),
  `**Tasks:**`, `**DoD:**` (with `(Manual, owner)` items), `**Tests:**`, and `**Owner steps:**` /
  `**Not in this phase:**` where they apply.
- **The Status table:** one new row per phase, in file order, `planned`, release `--`.

### Step 5: Write the ARCHITECTURE and VISION changes

- **ARCHITECTURE:** a new mechanism gets its own section, or extends the existing one with a
  `(from vA.B)` tag; invariants as a numbered list. New interfaces, file formats or setting names go to
  §Contracts with the test that will pin them. New settings go to the configuration table with their Phase.
  New threats go to §Security and access. New parts of the tree go to the repository layout. New tests go to
  §Testing and CI.
- **VISION:** new terms go to §Glossary. A new principle or a changed non-goal only with the owner's
  confirmation from Step 1.
- **One home per fact:** the mechanism is described in ARCHITECTURE once; the phase refers to it.

### Step 6: Self-check

- every new phase has Status, Goal, Tasks, DoD and Tests; every DoD item is automated or `(Manual, owner)`;
- every `ARCHITECTURE §Section` the new phase names exists;
- every new contract names its pinning test;
- no fixed phase changed its id; every renumbered phase has all its references updated;
- the Status table matches the phase headings, in order;
- the work does not contradict VISION, or the VISION change is part of the plan;
- no emoji.

### Step 7: Report

Show the owner:

- the new or moved phases (id, title, placement and why);
- the ARCHITECTURE and VISION sections changed;
- the owner decisions recorded, and the questions still open;
- any renumbering, with every reference updated.

Do **not** commit; the owner reviews first. Suggest the commit message `docs: plan vA.B — <title>` and the
next step of the project's delivery flow (for example `/generate-issues vA.B`).

## Important Rules

- **The specification must exist.** Otherwise `/generate-spec` first.
- **Fixed phases keep their ids**: released, or with an issues file or GitHub issues.
- **Scope changes are the owner's.** A plan that contradicts VISION waits for the owner's word, and the
  VISION edit goes in with it.
- **Same format as the rest.** FORMAT.md fields, in order; ARCHITECTURE holds the mechanism, ROADMAP the
  work.
- **Nothing invented.** Open decisions are asked or listed, never guessed.
- **No emoji, no secrets.**
- **Don't commit** without the owner's confirmation.
