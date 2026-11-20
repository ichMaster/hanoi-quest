---
name: execute-issues-file
description: Execute one phase's issues straight from its local specification/implementation/vA.B-issues.md (no GitHub). Implement -> gates -> stop and wait for the owner's confirmation after each step (concept document §10) -> commit -> push (if a remote exists) for each issue in dependency order, then write vA.B-execution-report.md. The offline counterpart of execute-issues.
---

# Skill: Execute Issues From File

Execute one phase's issues **straight from its issues file**, `specification/implementation/vA.B-issues.md`,
with **no GitHub involvement**: no issue lookup and no closing. Each issue is implemented, validated,
reported to the owner, then committed and pushed in dependency order, and the run ends with an execution
report.

This is the offline counterpart of `/execute-issues`. The discipline is the same; only the issue list comes
from the markdown file instead of `gh issue list`.

## Usage

```
/execute-issues-file <vA.B | path-to-issues-file> [--issue HANOI-###] [--dry-run]
```

- `/execute-issues-file v0.1` → executes `specification/implementation/v0.1-issues.md`.
- `/execute-issues-file @specification/implementation/v0.2-issues.md`.
- `--issue HANOI-###`: only that issue. Its file-listed dependencies must already be committed.
- `--dry-run`: print the execution plan without changing anything.

## Instructions

### Step 0: Verify prerequisites and read the file

1. **Branch and tree:** note the current branch and check `git status`.
   - If the only uncommitted file is this phase's issues file, commit it first (`docs: add vA.B issues`).
   - Any other uncommitted change: stop and ask.
2. **Remote:** check `git remote -v`. Without a remote, the run commits but doesn't push; say so up front.
3. **Read the issues file:** resolve the target to `specification/implementation/vA.B-issues.md` and read the
   summary table, the Dependency Tree and every `### HANOI-###` section. **No `gh` is used.**
4. **Read the spec:** the phase section of
   [specification/hanoi-vr-concept.en.md](../../../specification/hanoi-vr-concept.en.md) (steps and
   completion paragraph), its §3, §4, §5 and §9, plus `CLAUDE.md` (the gates and the rules that are easy to
   break).
5. **Phase v0.2 and later:** run `godot --version` and compare it with the issues file's scope notes. If it
   differs, stop and suggest `/reconcile-issues vA.B` first.
6. **Green baseline:** run the automated gates that apply, so a later failure can be attributed.

### Step 1: Build the execution queue from the file

- **Order:** parse the ids and titles from the summary table and order them by the Dependency Tree.
- **Skip what's done:** an issue whose id already appears in a commit subject (`git log --grep "^HANOI-###:"`)
  is done; skip it. This is what makes the run resumable.
- **`--issue`:** execute only that issue, after checking that its dependencies are committed.

Show the ordered plan and proceed. With `--dry-run`, stop here.

### Step 2: Execute each issue, in dependency order

1. **Announce:** `--- Starting HANOI-###: {title} (step §{section}.{n}) ---`.
2. **Read** its section: what needs to be done, how to check, and the acceptance criteria.
3. **Implement** per `CLAUDE.md` and the concept document, routed by component. The routing is the same as
   `/execute-issues` Step 2c:
   - `core/hanoi_state.gd` is a `RefCounted` with the §4.1 API and all game rules, no scene nodes.
   - `board/` builds the scene in code from primitives and exposes the §4.2 drag interface, the only API
     the input layers call; the board's decisions stay in pure static functions.
   - `input/` reads input and calls the board; `main.gd` does the §4.4 mode selection.
   - `ops` issues produce their repo artifacts plus a numbered checklist for the owner. Building and
     installing the APK is Claude's (`release-version` §Deploy); the owner's Meta account and headset
     steps are not.
   - A **contract change** updates both concept document files and its pinning test in the same commit.
   - GDScript with static typing, English identifiers and comments, no emoji.
4. **Validate:**
   - The gates from `CLAUDE.md` §Acceptance gates (import, tests, smoke), judged by their output: no
     `SCRIPT ERROR` or `ERROR:` line, and exit code 0 for tests.
   - Every issue that touches `core/` or the board's pure logic adds or extends a headless test.
   - Record each result as pass / fail / n/a.
5. **Stop and report to the owner** (concept document §10): what was done, the gate results, how to check
   it (concrete commands and what to look at), and the issue's **Manual (owner)** items as a numbered
   checklist. **Wait for the owner's confirmation.**
   - Confirmed: record each manual item as `confirmed by owner` and commit.
   - A problem reported: fix it, run the gates again, report again; it stays the same issue and commit.
   - A manual item deferred: record it as `pending owner`.
   - Skip the pause only when the owner explicitly says so for this run, and record that in the report.
6. **Commit** (one issue = one commit, only when the gates are green):

   ```bash
   git commit -m "$(cat <<'EOF'
   HANOI-###: {title}

   {1-2 sentence summary of what was implemented}

   Co-Authored-By: <the running model's trailer> <noreply@anthropic.com>
   EOF
   )"
   ```

   There is no `Closes #…` line, since there is no GitHub issue. Commit the `.uid` files Godot creates next
   to new scripts; never commit `.godot/`, `build/` or `android/`. An `ops` issue with no repo artifact has
   no commit; it is done once the owner confirms.
7. **Push:** `git push` if a remote exists.
8. **Log** the id and title, commit, files, gate results, manual checks and status (`completed` /
   `awaiting owner` / `failed` / `skipped`).

An issue whose code is committed but whose owner checks are pending is `awaiting owner`. Continue with
issues that don't depend on it.

### Step 3: Handle failures

On a failed implementation or a red gate:

1. Don't commit.
2. Revert tracked changes with `git checkout -- .` and delete **by name** the new files this issue created
   (and their `.uid` files). Never `git clean`.
3. Log the failure.
4. Ask whether to continue with the next independent issue or stop.

### Step 3b: No automatic version bump

Never touch `config/version` in `project.godot`, the Android preset's version fields, `RELEASE.txt` or tags
here; that is `/release-version`. A phase with failed, skipped or `awaiting owner` issues is not
releasable.

### Step 4: Write the execution report

Write `specification/implementation/vA.B-execution-report.md` with the same structure as `/execute-issues`
Step 4:

- a status summary table;
- a per-issue table (HANOI id · step · title · status · commit · files · gates · manual);
- detailed results;
- next steps.

There is no GitHub column. Commit it (`docs: vA.B execution report`, with the trailer) and push if a remote
exists.

## Important Rules

- **File-driven, no GitHub.** The issue list, details and order come from `vA.B-issues.md`. Never run
  `gh issue list`/`create`/`close`, and never write `vA.B-github-report.md`.
- **Stop after each step** and wait for the owner (concept document §10), unless the owner explicitly
  waived the pause for this run.
- **One issue = one commit**, one issue at a time, in dependency order.
- **No broken code.** Commit only when the gates are green, judged by their output.
- **Tests ship with the logic**: core rules and the board's pure decisions get headless tests.
- **Manual checks need the owner.** Never report one as passed on your own.
- **Contracts stay stable**: both concept document files and the pinning test change together.
- **Secrets stay out.** Keystore passwords and paths are never printed, logged or committed.
- **Ask on ambiguity.** If an issue's scope is unclear, ask rather than guess.
