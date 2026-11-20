---
name: review-and-fix-issues
description: Code-review a phase, a component or the current branch, write a criticality-ranked recommendations doc in specification/implementation/, implement the fix-now items with regression tests, then record what was done in the SAME doc. Never releases.
---

# Skill: Review & Fix Issues

One loop over the codebase: **review → recommend → fix → record.**

1. Run a critical code review.
2. Write a single recommendations document that ranks findings by criticality and marks each **FIX NOW**
   or **DEFER →**.
3. Implement the fix-now items with regression tests.
4. **Update that same document in place**, marking what was fixed and adding a "Fixes applied" section.

The recommendations and the results live in **one document**.

Under `/ship-phase` and `/ship-solution` it runs **after** the phase build is in the owner's hands and the
owner has tested it (order: build → deploy → the owner tests → review and fixes → release); a finding from
the owner's test is an input to the review. The fixes ship with the release, never on their own.

This skill fixes only small, in-scope, high-value findings. It **never** bumps the version or cuts a
release (that stays with `/release-version`), and it never pulls deferred or larger work forward without
flagging it.

## Usage

```
/review-and-fix-issues [target]
```

- `/review-and-fix-issues v0.1`: review what the phase delivered (through its tag `v0.1.0` if released).
- `/review-and-fix-issues board`: scope the review to one component (`core` / `board` / `input` / `main` /
  `tests`).
- `/review-and-fix-issues`: review the **current branch**, i.e. everything built so far.

## Instructions

### Step 0: Scope and a green baseline

1. **Resolve the target:**
   - a phase (`vA.B`) or its tag: the commits whose subjects carry that phase's `HANOI-###` ids, plus the
     files they touched;
   - a component;
   - no argument: the whole working tree.
2. **Clean tree:** check `git status` is clean and note the branch.
3. **Green baseline:** run the automated gates (`CLAUDE.md` §Acceptance gates: import, tests, smoke). If
   the suite is **red or flaky**, say so. Fix a clear flake first (small, its own commit) or raise it and
   ask. **Never review or fix on top of a red suite.**

### Step 1: Critical code review

Read the in-scope code and take the highest-risk areas first. Be **adversarial**: hunt for *real*
defects, not restatements of what works.

- **Rules of the game (core, concept document §2 and §4.1):**
  - Can a disk other than the top one be taken?
  - Is a larger disk on a smaller one rejected, with no state change and no `move_count` increment?
  - Is a move to the same peg not a move (no count, no `moved` signal)?
  - Is `is_solved()` true only when all disks are on the right peg, never at the start or on the middle
    peg?
  - Is `min_moves()` exactly `2^n - 1`? Is the disk count clamped to 3–7?
  - Do `reset(disk_count)` and a disk-count change clear `move_count` and emit `reset_done` once?
  - Does the test solve the tower recursively for n = 3–7 and check the move count, illegal moves and the
    win state?
- **Drag lifecycle (board, §4.2):**
  - What happens on `end_drag()` without a drag, or on `try_begin_drag()` while another disk is dragged or
    a tween is still running?
  - Can the same disk be grabbed by two controllers at once?
  - Reset or disk-count change during a drag or a tween: are freed disk nodes still referenced by a rig or
    a running tween?
  - Are signals connected once, not again on every reset?
  - Release far from the pegs: does the disk return to its source peg, with no move counted?
  - Is the snap measured horizontally only, within 0.15 m, to the nearest peg?
  - Is the peg highlight green only for a legal move, red otherwise, and cleared on release and on reset?
- **Scene (§3, §5):**
  - Do the sizes match §5 in meters (table height 0.80 m, peg spacing 0.33 m, disk radius 0.05–0.14 m
    evenly by size, disk thickness 0.03 m, settle tween 0.2 s)? Do disks stack at the right heights?
  - Is everything built from primitives and `StandardMaterial3D`, with no asset files, no `RigidBody3D`,
    shadows off on the one `DirectionalLight3D`?
- **Desktop rig (§4.3):**
  - Does the mouse ray hit the disks' `Area3D` (pickable, the right collision layer)?
  - Does the drag stay in the vertical plane through the peg line?
  - Do R and 3–7 work? What happens when the button is released outside the window?
- **XR rig and mode selection (§4.3, §4.4, from v0.2):**
  - Do the grip and trigger action names match the OpenXR action map?
  - Does the haptic pulse fire on a grab and on a rejected move?
  - Does one hand grabbing while the other holds a disk break the board's state?
  - With OpenXR found but not initialized (the Mac), does `main.gd` fall back to the desktop rig?
  - Are `use_xr` set and vsync off only in XR mode?
- **Quest performance (§7, at least 72 fps):**
  - Are materials and meshes created once and shared, not per frame or per disk move?
  - Any per-frame allocations, string formatting or node lookups in `_process`?
  - Is the renderer Compatibility?
- **Code rules (§10):** static typing everywhere; identifiers and comments in English; no emoji in code,
  comments, in-game text or docs.
- **Secrets:** are keystore passwords or paths printed, logged or committed?
- **Spec drift:** does the code diverge from concept document §3, §4 or §5, or add what §9 excludes?

For each finding, capture:

- a **concrete failure scenario** (inputs → wrong result or crash);
- a `file:line` anchor;
- a **severity**: HIGH / MEDIUM / LOW;
- a **proposed fix**.

Cross-check against the concept document. If a gap is already scheduled for a later phase, note that
instead of treating it as new.

### Step 2: Write the recommendations document (the plan)

Write **one** doc at `specification/implementation/<scope>-code-review.md`, e.g. `v0.1-code-review.md`,
`board-code-review.md`, or `branch-code-review.md` for the whole tree. No emoji. Include:

- a header: date, reviewer, **scope**, method;
- a **criticality-ranked summary table**: `# | Severity | Finding | Recommendation | Status`.
  Recommendation is `FIX NOW` or `DEFER → <home>`; Status starts as `pending`;
- for each finding, its failure scenario and proposed fix;
- a short **"What's solid"** section, to keep the review balanced;
- **suggested next actions**.

Decide **FIX NOW vs DEFER** honestly:

- **FIX NOW** means real, small, self-contained, high-value and in scope now: a rule the core gets wrong, a
  disk left in a wrong place after a drag, a crash on reset during a drag, a secret in a log.
- **DEFER →** means larger work, or work a later phase already owns. Give the home: a later phase
  (`v0.2`), `backlog` (no phase owns it) or `cleanup (/simplify)`. Do **not** pull it forward.

Commit the doc as the plan (`docs: vA.B code review`) **and push it** if a remote exists. The review is worth
keeping even if the fix pass is interrupted.

### Step 3: Implement the FIX NOW items, with tests

For each **FIX NOW** finding, in criticality order:

1. Implement the fix following `CLAUDE.md` and the concept document. Keep it minimal.
2. **Add a regression test that would have caught the bug**, as a headless test under `tests/` (see
   `CLAUDE.md` §Acceptance gates for how a test script is written). If the behavior can only be seen in a
   rendered window or in the headset (visuals, mouse picking, XR input), write in the doc why there is no
   headless test and add a **Manual (owner)** check that verifies the fix instead.
3. **Validate:** the gates green, judged by their output. Commit only passing code.
4. **Commit** one focused change per finding: `fix(<area>): … (code review #N)`, with the running model's
   `Co-Authored-By` trailer. **Then push.** Never leave a landed fix unpushed.
5. **A contract change** updates both concept document files and the pinning test in the **same** commit.

If a fix turns out bigger than "fix now" (it touches a contract broadly or needs a design decision),
**stop and re-classify it as DEFER** in the doc, with the reason, and move on. Don't half-land it.

### Step 4: Update the SAME document (the result)

Edit the doc **in place**:

- **Status column:** flip it to `FIXED — <commit>` for each applied fix; keep `deferred` for the rest.
- **"Fixes applied" section:** for each fix, give the change, the regression test (or the owner check) and
  the verification (final gate status).
- **"Architecture impact" note:** add one for any fix that **changed a documented contract or
  design-relevant behavior**, and make sure both concept document files reflect it. The next
  `/generate-issues` or `/reconcile-issues` reads these notes.
- **"Suggested next actions":** update them. Fixes on an already-released phase suggest a patch release
  (`/release-version A.B.1`); deferred items are carried into their phase.

Commit the update (`docs: vA.B code review — fixes applied`) **and push**.

### Step 5: Report

Summarize:

- findings by severity;
- which were **fixed** (with commits) and which **deferred** (with homes);
- the owner checks added for fixes with no headless test;
- the final gate status.

If fixes landed on an already-released phase, suggest `/release-version A.B.<next>`, but do **not** run
it. Offer a deeper pass with `/code-review high` for confirmation.

## Important Rules

- **One document, updated in place.** Recommendations and results share a single doc.
- **Fix only the FIX NOW items.** Never pull deferred work forward without re-classifying it in the doc.
- **Every fix ships a regression test** where the behavior can run headless; otherwise an owner check,
  with the reason written down.
- **Green before, green after.** Commit only code that passes the gates.
- **Record architecture deltas.** A contract change updates both concept document files and its test in
  the same commit, and gets an "Architecture impact" note.
- **Never release.** No version bump, no tag.
- **Never leave work unpushed** (when a remote exists). This skill can stop mid-way, so "the next step will
  push it" is not a safe assumption.
- **No emoji** in the review doc.
- **Ask on genuine ambiguity**: an unclear scope, or a borderline fix-now-vs-defer call.
