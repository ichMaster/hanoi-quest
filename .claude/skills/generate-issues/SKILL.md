---
name: generate-issues
description: Turn one phase of the concept document (v0.1 = Phase 1 screen version, v0.2 = Phase 2 port to Quest 3) into a dependency-ordered issues file at specification/implementation/vA.B-issues.md, one issue per numbered step, grounded in the real current code. The output feeds /upload-issues (GitHub flow) or /execute-issues-file (offline flow).
---

# Skill: Generate Phase Issues

Turn one **phase** of the concept document into a dependency-ordered **issues file** at
`specification/implementation/vA.B-issues.md`. The file is the input to `/upload-issues` → `/execute-issues`
(GitHub flow), or straight to `/execute-issues-file` (offline flow).

## Usage

```
/generate-issues <phase>
```

- `/generate-issues v0.1`, `/generate-issues 0.1` or `/generate-issues 1`: Phase 1 (concept document §6,
  screen version) → `specification/implementation/v0.1-issues.md`.

The phase ids are defined in `CLAUDE.md` §Delivery workflow: the concept document's "Phase N" is `v0.N`.
One file per phase. Issue ids (`HANOI-###`) are **globally sequential** across phase files **and across
regeneration runs**. Never reset them.

## Instructions

### Step 0: Read inputs

1. Normalize the argument to `v0.N` and check that the concept document
   [specification/hanoi-vr-concept.en.md](../../../specification/hanoi-vr-concept.en.md) has a
   "Phase N" section. For anything else, name it and ask.
2. Read that phase section: its numbered **steps** and its completion paragraph ("The phase is complete
   when ..."), which is the phase **DoD**. Note which DoD items need the owner: playing the game with the
   mouse, playing it in the headset, the frame rate.
3. Read the rest of the concept document: §2 (rules of the game), §3 (key decisions), §4 (architecture and
   interfaces), §5 (scene dimensions), §8 (environment and deployment), §9 (out of scope) and §10
   (instructions for Claude Code). The scope fence is the phase itself plus §9: never pull a later phase's
   work in early, and never add what no phase asks for.
4. Read `CLAUDE.md` for the acceptance gates and the rules that are easy to break.
5. **Find the next free `HANOI-###` id. Never restart the numbering.** Check both sources and continue
   from the higher:

   ```bash
   # (a) GitHub: survives a wiped working tree (skip if the repo has no remote)
   gh issue list --state all --limit 1000 --json title \
     --jq '.[].title | capture("HANOI-(?<n>[0-9]+)").n' 2>/dev/null | sort -n | tail -1

   # (b) local issues files: covers ids drafted but not uploaded
   grep -rhoE 'HANOI-[0-9]+' --include='*-issues.md' specification/implementation/ 2>/dev/null \
     | grep -oE '[0-9]+' | sed 's/^0*//' | sort -n | tail -1
   ```

   Pass the directory with `--include`, not a `*-issues.md` glob. Under zsh an unmatched glob aborts the
   command before `2>/dev/null` applies, and `specification/implementation/` may not exist yet.

   The next id is `max(a, b) + 1`, zero-padded to three digits. Start at `HANOI-001` only if both sources
   come back empty. Say in the report which sources were checked: with no remote, or with `gh`
   unauthenticated, (b) alone governs.
6. If `specification/implementation/vA.B-issues.md` already exists, ask whether to overwrite or append.

### Step 0.5: Reconcile with the real implementation

Ground the phase in what was actually built and fixed, not only in what the concept document describes.
Review fixes and hardening in earlier phases may have moved the code away from the document.

1. Read the **real current code** this phase builds on: `project.godot`, `main.tscn` / `main.gd`, `core/`,
   `board/`, `input/`, `tests/`, `export_presets.cfg` and `addons/` (whichever exist). Note the actual file
   names, class names, method signatures, signals, constants and project settings.
2. Read the earlier phases' `specification/implementation/*-execution-report.md` and `*code-review*.md`,
   especially their **"Fixes applied"** and **"Architecture impact"** notes.
3. Where the concept document is stale relative to a landed fix, the **code is ground truth** for this
   phase's issues. Note the drift, and put the concept document correction (both language files) into the
   issue that touches that contract.
4. **Phase v0.2 and later** (concept document §10): run `godot --version`, then read the current Godot
   "Deploying to Android" documentation page and the Godot OpenXR Vendors plugin README for that version.
   Use the version-specific setting names, export options and plugin install steps from those pages in the
   issues, not names from memory. Record the Godot version and the pages read in the scope notes.
5. **Owner-run state:** check what can be checked read-only from the Mac, for example `adb devices`
   (headset connected and authorized) or whether the Android SDK and JDK 17 are installed. Ask the owner
   about the rest: the Meta developer account and team, Developer Mode, the Android SDK path in Editor
   Settings. A step that is already done becomes a verification-only issue, not new work.

### Step 1: Decompose the phase

**One issue per numbered step of the phase.** The concept document §10 makes the step the unit of work:
after each step Claude stops, reports, and waits for the owner. So do not merge steps. Split a step into
two issues only when it is clearly L and the owner agrees to the split.

- **Size** by complexity:
  - **S:** one script, or a settings change.
  - **M:** a feature across a few files (for example the board with its animations).
  - **L:** a new component or a contract change.
- **Area**, one of:
  - `core`: `core/`.
  - `board`: `board/`.
  - `input`: `input/`.
  - `main`: `main.tscn`, `main.gd`.
  - `tests`: `tests/`.
  - `config`: `project.godot`, `export_presets.cfg`, `.gitignore`, `addons/`, the Android build template.
  - `docs`: the `specification/` files, README.md, CLAUDE.md.
  - `ops`: the owner's steps outside the repo: the Meta developer account and team, Developer Mode, USB
    debugging, wearing the headset, tuning in the headset. Building and installing the APK is Claude's,
    automatic, never asked (`release-version` §Deploy); its issue lists the owner's checks in the headset
    as **Manual (owner):**.
- **Order by dependency.** The concept document's step order is the dependency order. In v0.1 the first
  issue is the project skeleton, then the core and its test, the board, the desktop rig, the HUD.
- **Tests in every issue that touches `core/`:** add to or extend `tests/test_hanoi_state.gd`. Keep the
  board's decision logic pure as well (nearest peg within the snap threshold, whether a move is legal for
  the highlight, disk radius from its size, stack height), so a headless test can check it with plain data.
  Visuals, mouse input and XR are covered by the smoke gate and the owner's checks.
- **How to check:** every issue says, in one or two lines, how the owner checks the result (the command to
  run and what to look at). The executor prints it when it stops after the step.
- **Manual DoD checks** from the phase's completion paragraph go into the acceptance criteria as
  **Manual (owner):** items. The executor cannot pass them on its own.
- **Contract changes:** the contracts are concept document §3 (key decisions), §4.1 (core API), §4.2
  (board interface), §4.4 (mode selection) and §5 (scene dimensions). A change to one carries the update of
  both concept document files (`hanoi-vr-concept.en.md` and the Ukrainian `hanoi-vr-concept.md`) and the
  test that pins it, in the **same** issue.
- **Owner-run (`ops`) issues:**
  - List the exact commands from the concept document §7 and §8.
  - Mark which checks Claude can run read-only from the Mac.
  - Name any repo artifact the issue produces (for example `export_presets.cfg`).
- **Stay within the phase.** No XR code in v0.1, nothing from §9 (passthrough, hand tracking, locomotion,
  sound, menus, saving results, store publishing).

### Step 2: Write the issues file

Write `specification/implementation/vA.B-issues.md` in English, with no emoji. Use **exactly** this format:

````markdown
# vA.B — Issues

Issues for phase **vA.B — {title}** (concept document §{section}, Phase {N}), derived from the steps and the
completion criterion in [hanoi-vr-concept.en.md](../hanoi-vr-concept.en.md) §{section} and the interfaces in
§4. This file covers one phase; ids continue from the previous phase (HANOI-{prev} → **HANOI-{first}…{last}**).

{1–3 sentences: what the phase delivers, what it builds on, what it leaves for later phases.}

## Issues Summary Table

| # | ID | Step | Title | Size | Area | Phase | Dependencies |
|---|----|------|-------|------|------|-------|--------------|
| 1 | HANOI-{first} | §6.1 | {title} | S | config | vA.B | -- |
| 2 | HANOI-{…} | §6.2 | {title} | M | core | vA.B | HANOI-{first} |

**Size legend:** S = one script or a settings change · M = a feature across a few files · L = a new component or a contract change

---

## Dependency Tree

```
HANOI-{first} ({gate})
  |
  +-- HANOI-{…} (…)
        |
        +-- HANOI-{…} (…)  => {phase DoD}
```

**Parallelization hints:** {what must go first; what is independent}.

---

## vA.B — {title}

### HANOI-{id} — {Title}

**Step:** §{section}.{n} — {the step's text from the concept document, shortened}

**Description:**
{1–3 sentences; name the files it touches.}

**What needs to be done:**
- {bullet}

**Dependencies:** {HANOI ids, or None}

**Expected result:**
{one sentence}

**How to check:**
{the command to run and what the owner looks at}

**Acceptance criteria:**
- [ ] {functional criterion}
- [ ] **Tests:** {the headless tests added or extended} — *(issues that touch core/ or the board's pure logic)*
- [ ] **Contract:** {contract + the test that pins it + both concept document files updated} — *(only if a contract changes)*
- [ ] **Gates:** {which of import / tests / smoke apply}
- [ ] **Manual (owner):** {check from the phase's completion paragraph} — *(only where the DoD needs the owner)*

---

{repeat the `### HANOI-{id} …` block per issue}

## vA.B scope notes

**Critical path:** HANOI-{…} → … → HANOI-{…}.
**Phase DoD (concept document §{section}):** {restate the completion paragraph}.
**Contracts touched:** {contracts + their tests, or "none"}.
**Owner steps:** {what the owner does outside the repo, or "none"}.
**Godot version:** {output of `godot --version` and the documentation pages read} — *(v0.2 and later)*
**Not in this phase:** {nearby work that belongs to a later phase or to §9}.
**Generated later:** `vA.B-github-report.md` (on upload), `vA.B-execution-report.md` (on execution).
````

### Step 3: Report

Show the user the file path, the issue count, the `HANOI-###` range, the critical path, and which id
sources were checked. Suggest the next step:

```
/upload-issues @specification/implementation/vA.B-issues.md   # GitHub flow
/execute-issues-file vA.B                                     # offline flow
```

Do **not** create GitHub issues or commit here. This skill only writes the local file, so the user can
read and edit it first.

## Important Rules

- **One file per phase**, at `specification/implementation/vA.B-issues.md`.
- **One issue per numbered step**, so the per-step stop of concept document §10 maps onto issues.
- **Ids are globally sequential** (`HANOI-###`) across phase files and regeneration runs. Resolve the next
  id as `max(GitHub, local issues files) + 1`.
- **Tests in every issue that touches `core/`** or the board's pure logic. Manual DoD checks are labeled
  **Manual (owner):**.
- **Contract change = both concept document files + the pinning test**, all in the same issue.
- **Stay within the phase** and outside concept document §9. Prototype simplicity beats completeness.
- **Honor the DoD.** Together, the issues must satisfy the phase's completion paragraph.
- **Check the Godot version before Phase 2** (concept document §10); never rely on remembered OpenXR or
  export setting names.
- **No emoji** in the issues file (concept document §10).
- **Ask on ambiguity.** If a step is under-specified, ask before inventing scope.
- **Don't touch GitHub.** `/upload-issues` does that.
