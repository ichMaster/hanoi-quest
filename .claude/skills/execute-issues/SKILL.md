---
name: execute-issues
description: Execute one phase's GitHub issues (label vA.B::phase) sequentially in dependency order - implement, run the acceptance gates, stop after each step and wait for the owner's confirmation (concept document §10), commit, push, close - then write vA.B-execution-report.md.
---

# Skill: Execute GitHub Issues

Execute one phase's GitHub issues sequentially. For each one: implement, validate, stop and report to the
owner, then commit, push and close once the owner confirms. Then write an execution report.

## Usage

```
/execute-issues <label|phase> [--issue HANOI-###] [--dry-run]
```

The label is the phase label exactly as it appears on GitHub (`v0.1::phase`). A bare `v0.1` also works.

- `/execute-issues v0.1::phase`: execute every open issue of phase v0.1.
- `/execute-issues v0.1::phase --issue HANOI-003`: one issue (its dependencies must already be closed).
- `/execute-issues v0.1::phase --dry-run`: show the execution plan without changing anything.

## Instructions

### Step 0: Verify prerequisites

1. **Branch:** note the current branch.
2. **Clean tree:** run `git status`. If the only uncommitted files are this phase's issues file or GitHub
   report under `specification/implementation/`, commit them first (`docs: add vA.B issues`). Any other
   uncommitted change: stop and ask.
3. **GitHub:** `gh` is authenticated and the repo has a remote.
4. **Fetch the open issues:** `gh issue list --label "vA.B::phase" --state open --limit 100`.
5. **Read the phase files:** the issues file `specification/implementation/vA.B-issues.md`, and
   `vA.B-github-report.md` for the `HANOI-###` → `#number` mapping.
6. **Read the spec:** the phase section of
   [specification/hanoi-vr-concept.en.md](../../../specification/hanoi-vr-concept.en.md) (steps and
   completion paragraph), its §3 (key decisions), §4 (architecture and interfaces), §5 (dimensions) and §9
   (out of scope), plus `CLAUDE.md`: the gates and the rules that are easy to break.
7. **Phase v0.2 and later:** run `godot --version` and compare it with the version in the issues file's
   scope notes. If it differs, stop and suggest `/reconcile-issues vA.B` first.
8. **Green baseline:** run the automated gates that apply (`CLAUDE.md` §Acceptance gates), so a later
   failure can be attributed. Never start on a red suite.

### Step 1: Build the execution queue

- **Order:** parse `HANOI-###` ids from the issue titles (`HANOI-###: {title}`) and order them by the issues
  file's Dependency Tree. Issues with no unmet dependency go first.
- **Resuming:** closed issues are already excluded (`--state open`), so a re-run resumes where the last one
  stopped.
- **`--issue`:** execute only that issue, after checking that its dependencies are closed.

Show the plan and ask for confirmation. With `--dry-run`, stop here.

### Step 2: Execute each issue (loop)

#### 2a. Announce

Print `--- Starting HANOI-###: {title} (step §{section}.{n}) ---`.

#### 2b. Read the issue

Read its detailed section in the issues file: what needs to be done, how to check, and the acceptance
criteria.

#### 2c. Implement

Follow `CLAUDE.md` and the concept document. Route by component:

- **`core/hanoi_state.gd`:** a `RefCounted` class with no scene nodes; the API of concept document §4.1
  (methods, `move_count`, the signals `moved`, `solved`, `reset_done`). All game rules live here.
- **`board/hanoi_board.gd`, `board/hud.gd`:** a `Node3D` that builds the table, pegs and disks in code from
  Godot primitives and `StandardMaterial3D`, kept in sync with the core through its signals. The drag
  interface of §4.2 (`try_begin_drag`, `update_drag`, `end_drag`, `press_button`) is the only API the input
  layers call. Disks are not physics bodies: a dragged disk follows the cursor or hand, and a tween moves it
  on release. The §5 dimensions are named constants in meters. Keep decision logic (nearest peg, legal-move
  highlight, radius from size) in pure static functions so a headless test can check it.
- **`input/desktop_rig`, `input/xr_rig`:** read input and call the board interface; no game rules here.
- **`main.tscn` / `main.gd`:** the mode selection of §4.4.
- **`config`:** `project.godot` settings, `export_presets.cfg`, `addons/`, the Android build template. From
  v0.2 use the setting names for the installed Godot version (concept document §10), not remembered ones.
- **Contract changes** (concept document §3, §4.1, §4.2, §4.4, §5) update both concept document files
  (`hanoi-vr-concept.en.md` and the Ukrainian `hanoi-vr-concept.md`) and the test that pins the contract,
  in the same commit.
- **Style:** GDScript with static typing; identifiers and comments in English; no emoji in code, comments,
  in-game text or docs. No asset files, no `RigidBody3D`, no external libraries except the Godot OpenXR
  Vendors plugin from v0.2 (no Godot XR Tools).
- **Scope:** stay inside the step and the phase; nothing from §9.

#### 2d. Validate

1. **Gates** from `CLAUDE.md` §Acceptance gates: import, tests, smoke. Each must pass, or be `n/a` (before
   `project.godot` exists). Godot exits 0 on script errors, so a gate passes only when its output has no
   `SCRIPT ERROR` or `ERROR:` line, as `CLAUDE.md` describes.
2. **Tests:** every issue that touches `core/` or the board's pure logic adds or extends a headless test.
3. **Acceptance criteria:** walk each criterion against the phase's completion paragraph.

Never go on to 2e with a red gate.

#### 2e. Stop and report to the owner (concept document §10)

After each step, stop and report:

- what was done (files and the main points, 3–6 lines);
- the gate results;
- **how to check it**: the issue's How to check, as concrete commands (for example `godot --path .`) and
  what to look at;
- the **Manual (owner)** items of this issue, as a numbered checklist.

Then **wait for the owner's confirmation** before committing and before starting the next issue.

- **The owner confirms:** go to 2f. Record each manual item as `confirmed by owner`.
- **The owner reports a problem:** fix it, run the gates again, and report again. The fix belongs to the
  same issue and the same commit.
- **The owner defers a manual item:** record it as `pending owner`; the issue stays open (2h).
- The pause can be skipped only when the owner explicitly says so for this run (for example "do not stop
  between steps"). Record that in the execution report.

#### 2f. Commit

```bash
git add {specific files created or modified}
git commit -m "$(cat <<'EOF'
HANOI-###: {title}

{1-2 sentence summary of what was implemented}

Closes #{github-issue-number}

Co-Authored-By: <the running model's trailer> <noreply@anthropic.com>
EOF
)"
```

- **Commit the `.uid` files** Godot creates next to new scripts and resources, together with them. Never
  commit `.godot/`, `build/` or `android/`.
- **Manual checks still pending:** write `Refs #N` instead of `Closes #N`, so that pushing doesn't close an
  issue the owner hasn't verified.
- **`ops` issue with no repo artifact:** it has no commit; go straight to 2h once the owner confirms.
- **Installing on the headset is never asked about:** an `ops` issue that builds and installs the APK is
  Claude's, done automatically by `release-version` §Deploy when the headset is connected; only the
  owner's checks in the headset wait for the owner.

#### 2g. Push

`git push`.

#### 2h. Close or hold the issue

- **All criteria met:** close the issue with a summary.

  ```bash
  gh issue close {number} --comment "$(cat <<'EOF'
  ## Implementation Summary

  **Commit:** {hash or "none (owner steps only)"}
  **Files changed:** {count}

  ### What was done
  {bullets}

  ### Validation
  {import / tests / smoke: pass, fail or n/a; manual: confirmed by owner}

  ### Acceptance criteria
  {checklist}
  EOF
  )"
  ```

- **Owner checks still pending:** leave the issue open. Comment with the pending checklist, mark it
  `awaiting owner` in the log, and continue with issues that don't depend on it.

#### 2i. Log progress

Add to the execution log: the issue id and title, commit, files, gate results, manual checks, and status
(`completed` / `awaiting owner` / `failed` / `skipped`).

### Step 3: Handle failures

If implementation or validation fails:

1. Do not commit broken code.
2. Revert: `git checkout -- .` for tracked files, and delete **by name** the new files this issue created
   (and their `.uid` files). Never `git clean`.
3. Comment on the GitHub issue explaining what failed.
4. Log the failure.
5. Ask the user whether to continue with the next issue (if nothing depends on the failed one) or stop.

### Step 3b: No automatic version bump

Never change `config/version` in `project.godot`, the Android preset's version fields, `RELEASE.txt` or
tags here. That is `/release-version`, on explicit confirmation. A phase with failed, skipped or
`awaiting owner` issues is **not** releasable; say so in the report.

### Step 4: Write the execution report

Write `specification/implementation/vA.B-execution-report.md`:

```markdown
# Phase vA.B — Execution Report

**Date:** {date}
**Branch:** {branch}
**Label:** vA.B::phase
**Target release:** vA.B.0
**Godot version:** {godot --version}
**Executed by:** Claude Code

## Summary

| Status | Count |
|--------|-------|
| Completed | {n} |
| Awaiting owner | {n} |
| Failed | {n} |
| Skipped | {n} |
| Remaining | {n} |

## Issues

| # | HANOI ID | Step | Title | Status | Commit | Files | Gates | Manual |
|---|----------|------|-------|--------|--------|-------|-------|--------|
| 1 | HANOI-001 | §6.1 | ... | completed | a1b2c3d | 4 | import pass, tests n/a, smoke pass | confirmed |

## Detailed Results

### HANOI-001: ...
**Status:** completed · **Commit:** a1b2c3d · **GitHub:** #1
**Gates:** [x] import · [ ] tests (n/a) · [x] smoke
**Manual (owner):** {each check: confirmed by owner / pending owner}

## Next Steps
{remaining or awaiting-owner issues, their dependencies, and whether the phase is releasable}
```

Commit the report (`docs: vA.B execution report`, with the trailer) and push.

## Important Rules

- **One issue at a time**, in dependency order. Never start an issue whose dependencies aren't closed.
- **Stop after each step** and wait for the owner (concept document §10), unless the owner explicitly
  waived the pause for this run.
- **One issue = one commit.** Never mix work across ids.
- **No broken code.** Commit only when the gates that apply are green, judged by their output, not only by
  the exit code.
- **Tests ship with the logic.** Core rules and the board's pure decisions get headless tests.
- **Manual checks need the owner.** Never report a manual check as passed without the owner's
  confirmation, and never `Closes #N` an issue with pending owner checks.
- **Contracts stay stable.** A contract change updates both concept document files and its pinning test in
  the same commit.
- **Secrets stay out:** keystore passwords and paths in Editor Settings or `.godot/export_credentials.cfg`
  are never printed, logged or committed.
- **Ask on ambiguity.** If an issue is unclear, ask rather than guess.
