---
name: reconcile-issues
description: Review one phase's already-generated specification/implementation/vA.B-issues.md against the REAL current implementation (and the headset and toolchain state, read-only) and correct any issue that has drifted - stale file names, changed signatures, evolved contracts, renamed Godot settings, or work already done. Edits the file in place with a visible "Reconciled" mark. Corrects issues only; never implements code or changes the version.
---

# Skill: Reconcile Issues

`/generate-issues` grounds new issues in the real implementation (its Step 0.5). This skill does the same
for a **pre-generated** issues file. It reads `specification/implementation/vA.B-issues.md`, compares each
issue's assumptions with the **actual current code**, and **corrects the issues that drifted, in the file,
with a visible change-mark**, so the record shows the original issue was modified.

It only **corrects the issues**: it never implements code, never touches the version and never uses GitHub.
Run it right before `/execute-issues-file`.

## Usage

```
/reconcile-issues <vA.B | path-to-issues-file>
```

- `/reconcile-issues v0.2` → reconciles `specification/implementation/v0.2-issues.md`

## Instructions

### Step 0: Read the issues and the real implementation

1. **The issues file:** resolve the target to `specification/implementation/vA.B-issues.md` and read all of
   it: the summary table, the dependency tree and every `### HANOI-###` section.
2. **The code:** read the **real current code** the issues touch: `project.godot`, `main.tscn` / `main.gd`,
   `core/`, `board/`, `input/`, `tests/`, `export_presets.cfg` and `addons/`. Note the actual file and class
   names, method signatures, signals, constants and project settings.
3. **Earlier records:** read the earlier phases' `specification/implementation/*-execution-report.md` and
   `*code-review*.md`, especially **"Fixes applied"** and **"Architecture impact"**.
4. **The spec:** read the phase section of
   [specification/hanoi-vr-concept.en.md](../../../specification/hanoi-vr-concept.en.md) (its steps and
   completion paragraph), its §3, §4 and §5 (the contracts), and `CLAUDE.md`.
5. **Phase v0.2 and later:** run `godot --version` and compare it with the version recorded in the issues
   file's scope notes. If it changed, or the file has none, re-read the current Godot "Deploying to Android"
   page and the Godot OpenXR Vendors plugin README (concept document §10) and check every setting name and
   install step the issues use.
6. **Owner-run work:** check the state read-only where possible (for example `adb devices`). If an owner
   step may already be done, ask the owner.

### Step 1: Find the drift

Compare each issue's **assumptions** with reality:

- **Paths:** wrong or renamed files and scenes (for example an issue naming `board/board.gd` when the code
  has `board/hanoi_board.gd`).
- **Names and signatures** that changed: `HanoiState` methods and signals, the board's drag interface,
  button ids, constants for the §5 dimensions, project setting keys, Android export option names, OpenXR
  action names.
- **Contracts** that a landed fix moved past the concept document: the core API, the board interface, mode
  selection, the scene dimensions, the key decisions of §3.
- **Work already done:** the deliverable was already shipped by an earlier fix or phase, or an owner step
  (developer account, Developer Mode, Android SDK) was already performed.
- **The code is ground truth** where it disagrees with the issue text or a stale concept document.

### Step 2: Correct the issues in place, with a visible mark

For each issue that drifted, edit its section in `vA.B-issues.md`:

1. **Fix the details** (Description, What needs to be done, How to check, Acceptance criteria) so they
   match the real implementation. Keep the **HANOI id, the step and the intent**; correct only what drifted.
2. **Add a change-mark** as a blockquote directly under the issue heading:
   > **Reconciled (<today>):** originally referenced `board/board.gd` and `start_drag()`; corrected to the
   > shipped `board/hanoi_board.gd` and `try_begin_drag()`. Reason: matches the real implementation.
3. **Moot issues** (already delivered) stay in the file with a clear mark:
   > **Reconciled (<today>):** already satisfied by `<commit / release / owner step>`; execution is
   > verification-only (add or confirm the test, or re-run the check; no new production code).
4. Use today's date. Never rewrite silently; every change is stamped. No emoji.

### Step 3: If nothing drifted

Leave issues that match reality untouched. If **no** issue needed a correction, add one note under the
file's intro and stop:
`> **Reconciled (<today>): no drift found — issues match the current implementation.**`

### Step 4: Record

Commit the corrected file (`docs: reconcile vA.B issues against the implementation`, with the running
model's `Co-Authored-By` trailer) and push if a remote exists. Report which issues were corrected and why,
which were marked moot, and which were untouched.

## Important Rules

- **Correct issues only.** Never implement code, change the version or use GitHub. That is
  `/execute-issues-file` and `/release-version`.
- **The real code is ground truth** where it disagrees with the issue or a stale concept document.
- **Every change is marked.** Keep the HANOI id, step and intent, and add a dated `Reconciled` blockquote.
- **No churn.** Don't touch issues that already match reality.
- **Ask on genuine ambiguity.** If it's unclear whether the issue or the code is right, raise it with the
  user rather than guessing.
