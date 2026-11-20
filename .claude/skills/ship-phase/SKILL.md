---
name: ship-phase
description: Full GitHub-backed delivery pipeline over the concept document's phases (v0.1 = Phase 1 screen version, v0.2 = Phase 2 port to Quest 3). Takes one selector or a comma-separated LIST of phases/ranges (e.g. v0.1, v0.2, v0.1-v0.2, v0). Missing earlier phases are added automatically, the set is de-duplicated, sorted into document order, and already-released phases are skipped. Per phase - generate-issues (with reconcile), upload-issues, execute-issues (stopping after each step for the owner), DEPLOY the build to the headset (automatic, never asked; n/a for the desktop phase), the OWNER TESTS it, review-and-fix-issues, then release-version A.B.0 which DEPLOYS the release. At the END of the run a HARDEN sweep runs BY DEFAULT (opt out with --no-harden). Gated; stops on failure.
---

# Skill: Ship Phase — the full delivery pipeline

Drive the whole loop over the concept document's phases. **Each phase is released before the next one is
generated**, so the next phase's issues are reconciled against the real, post-fix implementation. The
hardening sweep of deferred findings runs **by default once at the end of the run**; pass `--no-harden` to
skip it.

**The loop:**

```
PLAN = selectors → phases → de-duplicated → + missing earlier phases → sorted into document order
       → minus already-released (tag vA.B.0 exists)

for each PHASE vA.B in PLAN (in order):
    0. RECONCILE  — ground vA.B in the real implementation + all prior fixes (inside generate-issues)
    1. generate-issues vA.B
    2. upload-issues @specification/implementation/vA.B-issues.md
    3. execute-issues vA.B::phase   (implement → gates → STOP after each step for the owner → commit → push → close)
    4. DEPLOY the phase build        ← automatic, never asked (headset phases: release-version §Deploy; v0.1: n/a)
    5. OWNER TESTS it                ← the phase's completion criterion, confirmed or waived
    6. review-and-fix-issues vA.B   (review → ranked doc → fix-now fixes → same doc)
    7. release-version A.B.0        ← RELEASE PER PHASE (tag vA.B.0) and DEPLOY it (headset phases), automatic
    → REPORT the phase to chat
→ END OF RUN: HARDEN (harden-findings <plan scope> --release) — BY DEFAULT, skipped with --no-harden
→ overall summary to chat
```

This skill is a **thin orchestrator**. It sequences the sub-skills, adds the gating and the end-of-run
hardening, and releases per phase; each sub-skill keeps its own discipline.

> **This pipeline releases and deploys.** Invoking `/ship-phase` is the explicit opt-in to the automated
> per-phase releases (real tags and pushes), **to every install on the headset** and to the HARDEN sweep.
> `/release-version`'s own rules still hold: it never downgrades and it confirms the changelog. To build
> without releasing, use the individual skills.

> **The order:**
> 1. the phase's code is built (its issues closed, gates green), stopping after each step for the owner
>    (concept document §10);
> 2. **deploy** that build to the headset — automatically, never asking (n/a for v0.1, which runs on the
>    Mac);
> 3. **the owner tests it** (the phase's completion criterion);
> 4. **code review and fixes** (`review-and-fix-issues`);
> 5. **release and deploy** — the release tag, then its build installed, automatically.
>
> A fix found later — in the owner's test or by HARDEN — ships as a patch `A.B.C` through the same order
> where it applies: fix → review → release → deploy.

## Usage

```
/ship-phase <selector>[,<selector>…] [--no-harden]
```

A **selector** is a phase (`v0.1`, a bare `0.1`, or the phase number `1`), a version (`v0` = all its
phases), or a range (`v0.1-v0.2`). Pass one, or a comma-separated list of any mix; whitespace around commas
is ignored.

- `/ship-phase v0.1`: ship Phase 1 (screen version).
- `/ship-phase v0.2`: ship Phase 2, plus Phase 1 first if it isn't released yet.
- `/ship-phase v0 --no-harden`: ship every phase, with no HARDEN sweep.

> **The list is a target, not the whole plan.** The phases are cumulative: the Quest 3 port (v0.2) adds a
> control layer to the Phase 1 game. So missing earlier phases are **added automatically**, and anything
> already released is skipped.

## Instructions

### Step 0: Scope, baseline and the plan

1. **Parse the selector list.** Split on commas and trim. Each element is a phase, a version, or a range.
   Record whether `--no-harden` was passed.
2. **Resolve against the concept document.** The phase universe is the "Phase N" sections of
   [specification/hanoi-vr-concept.en.md](../../../specification/hanoi-vr-concept.en.md), **in file
   order**, with ids `v0.N` (`CLAUDE.md` §Delivery workflow). A version selector expands to all its phases;
   a range to every phase it spans.
3. **Reject nothing silently.** If an element doesn't resolve to a real phase (a typo, `v0.9`, a reversed
   range `v0.2-v0.1`), name it and ask. Never drop it and ship the rest.
4. **Expand, de-duplicate and fill.** Resolve the elements to a set of phases. Then add **every earlier
   phase** before the latest one that isn't already in the set. These are requirements, not scope creep:
   report them at confirmation but don't ask permission.
5. **Sort into document order.** The set is never a running order: `/ship-phase v0.2,v0.1` ships v0.1
   first. If the order differs from what was typed, say so.
6. **Skip released phases** (tag `vA.B.0` exists). A phase that is partly done (issues file or GitHub
   issues exist, but no tag) resumes from its remaining steps. The sub-skills are idempotent: generate asks
   before overwriting, upload skips existing issues, execute skips closed ones, release refuses a downgrade.
7. **Preconditions:**
   - `gh` is authenticated and the repo has a GitHub remote. If either is missing, stop and offer
     `gh repo create` (run by the user) or the offline `/ship-solution`.
   - The tree is clean.
   - The automated gates are green, or `n/a` before `project.godot` exists. Never start on a red suite.
8. **Flag the owner's work up front.** Tell the user at confirmation that the run **will pause**:
   - after **every step** (concept document §10), with what was done and how to check it;
   - for the live test of each phase: playing with the mouse (v0.1); playing in the headset with two
     controllers at 72 fps or more (v0.2);
   - for owner-run steps of v0.2: the Meta developer account and team, Developer Mode, connecting the
     headset and accepting USB debugging.
   It never pauses to ask whether to deploy.
9. **Confirm the plan once.** Show:
   - the ordered phase list;
   - the filled-in phases, any reordering and the skips;
   - whether HARDEN runs;
   - the expected owner pauses.

   Then run. Don't re-confirm before each sub-step; pause only for the per-step stops and the blockers in
   the rules below.

**Worked example:** `/ship-phase v0.2` on a repo with nothing released yet.

```
selectors : v0.2
expanded  : v0.2
filled    : v0.1                      ← required earlier phase, not released
skipped   : (none)
ordered   : v0.1 → v0.2

PLAN: v0.1, v0.2 → HARDEN v0.1-v0.2 → summary
Owner pauses: after each step (§6.1–6.5, §7.1–7.6); the mouse playthrough (v0.1);
              the Meta account, Developer Mode and USB debugging, then the headset playthrough at 72 fps (v0.2)
```

### Step 1: For each phase, the steps, gated

Run the phases **strictly in sequence**: phase N+1 starts only after phase N is **released**. Invoke each
sub-skill through the **Skill tool** and follow its instructions fully.

0. **RECONCILE.** This happens inside `generate-issues` (its Step 0.5). It reads the real code, the
   earlier execution reports and the earlier code-review docs ("Fixes applied", "Architecture impact").
   Where fixes moved the code away from the concept document, the code is ground truth. For v0.2 it also
   checks `godot --version` against the current Android and OpenXR Vendors documentation.
1. **`generate-issues vA.B`** writes `specification/implementation/vA.B-issues.md`.
2. **`upload-issues @specification/implementation/vA.B-issues.md`** creates the GitHub issues, labels and
   dependency comments, and `vA.B-github-report.md`, and commits them.
3. **`execute-issues vA.B::phase`** implements the issues one per commit, stopping after each step for the
   owner's confirmation, runs the gates, pushes and closes them. An issue whose owner checks need the
   deployed build stays open for steps 4–5.
4. **DEPLOY the phase build — automatically, never asked.**
   - Headset phases (v0.2 and later): build and install from the current commit with the procedure in
     `release-version` §Deploy. A failed export or install stops the run and is reported.
   - v0.1: n/a; the owner runs the game on the Mac with `godot --path .`.
5. **The OWNER TESTS it.** Give the owner the phase's completion criterion as a short checklist and wait
   for the owner's word: each check confirmed, or waived. Fix what the test finds first (one commit each,
   `fix(<area>): … (owner test vA.B)`, gates green, deployed again the same way) before the review. Then
   close the remaining issues and update `vA.B-execution-report.md` with the test results.
6. **`review-and-fix-issues vA.B`** writes the ranked review doc, fixes only the FIX NOW items (with
   regression tests) and records them in that same doc.
7. **`release-version A.B.0`** bumps the version, tags `vA.B.0`, pushes, and **deploys the release** to the
   headset (headset phases) automatically.

**Gate the hand-offs:**

- upload only after generate wrote the file;
- execute only after the issues exist;
- deploy only after every code issue is closed and the gates are green;
- review only after the owner has tested the build (confirmed or waived) and every issue is closed;
- release only after the review's fix-now items are committed and the gates are green;
- start the next phase only after this one is released **and deployed** (where it deploys).

**Every phase boundary ends pushed and clean.** Before the next phase, check that `git status` is clean and
there are no unpushed commits, and `git push` if there are. This skill **stops on failure by design**, so a
stop must never leave a phase's work on one machine only.

### Step 2: REPORT each phase to chat

After each release, report:

- the `HANOI-###` range → GitHub numbers;
- the execution commit range and the gate status;
- the deploys (the test build and the release build, with their version codes and times), or n/a;
- the manual checks the owner confirmed or waived;
- the review findings (fixed now / deferred, with homes);
- any "Architecture impact" notes;
- the release tag.

Then continue with the next phase.

### Step 3: END OF RUN — HARDEN (default; `--no-harden` skips it)

Once the last phase in the plan is released, invoke **`harden-findings <first>-<last> --release`** through
the Skill tool, with the run's phase range.

- **What it does:** fixes every still-unfixed HIGH / MEDIUM finding from the run's review docs, each with a
  regression test, updates the docs in place, and ships a patch release on the latest phase (e.g.
  `v0.2.1`).
- **Held findings:** its escape hatch applies, so a fix that can't land cleanly is held with a reason, not
  forced.
- **With `--no-harden`:** skip it, and list the outstanding HIGH/MEDIUM findings and their homes in the
  summary.

Then give a short **overall summary**: phases shipped, phases skipped as already released, the HARDEN
outcome, anything that stopped early and what remains, and what's next.

## Important Rules

- **The order is fixed:** build (stopping after each step) → deploy → the owner tests → review and fixes →
  release and deploy.
- **Deploys are automatic — never ask.** Every headset phase build and every headset release is installed
  by Claude (`release-version` §Deploy). Only a disconnected headset or a failed export or install pauses,
  as such.
- **Release per phase (`A.B.0`)**, after it is built, deployed, owner-tested, reviewed and fix-now-fixed.
  Never batch phases into one release; never release mid-phase.
- **Later fixes are patches** (`A.B.1`, …) — reviewed, released and deployed the same way.
- **Next phase only after the previous one is released.** This strict order is what makes reconciliation
  meaningful.
- **HARDEN runs once at the end of the run, by default.** Invoking the skill is the consent; `--no-harden`
  is the opt-out.
- **Every phase boundary ends pushed and clean.**
- **Stop on failure; don't paper over it.** A failed sub-skill or a red gate halts the run. Report what
  completed and what remains. Never release a phase whose gates aren't green.
- **Pause for the owner** after every step, for the live test and for owner-run steps: manual checks are
  confirmed (or waived) by the owner, never assumed. Deploys are not owner steps.
- **Pause for real decisions:** an id or tag collision, an overwrite-or-append prompt, a held HARDEN
  finding, a failure. Routine confirmations run straight through.
- **Delegate, never duplicate.** This skill sequences `generate-issues`, `upload-issues`, `execute-issues`,
  `review-and-fix-issues`, `release-version` and `harden-findings`, and adds gating; it has no logic of its
  own.
- **The plan is document-ordered and dependency-complete, always.** De-duplicate, fill the earlier phases,
  sort into concept document order, then drop the released ones. Report the fill, the reordering and the
  skips.
