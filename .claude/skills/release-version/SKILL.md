---
name: release-version
description: Bump the project version (phase v0.N -> 0.N.0, post-release fix -> 0.N.C), update project.godot / the Android export preset / RELEASE.txt / the CLAUDE.md status line, commit, create an annotated tag, push, then DEPLOY the release to the Quest 3 headset automatically when the phase runs there (never ask). Holds the canonical deploy procedure the other delivery skills use.
---

# Skill: Release Version

Bump the project version, update every version reference, write release notes, commit, tag, push, and
**deploy the release automatically** (Step 7).

> **Deploys are automatic.** Never ask whether to install a build on the headset: a release is deployed as
> part of releasing, a phase build before its review as part of `/ship-phase` and `/ship-solution`. What
> needs the owner is the headset itself: connecting it, accepting USB debugging, wearing it and testing.

## Usage

```
/release-version <version> [changelog line 1; changelog line 2; ...]
```

- `/release-version 0.1.0`: release phase v0.1 and generate the changelog from the commits.
- `/release-version 0.2.1 Disk no longer sticks to the hand after reset; grip action renamed`: a patch with
  given notes.

**Version scheme (`A.B.C`):** the concept document's Phase N is phase `v0.N` and releases as `0.N.0`, tag
`v0.N.0` (`CLAUDE.md` §Delivery workflow). A later fix on that phase bumps `C` (`0.N.1`, …). Releases are
cut per phase. A version beyond the prototype (for example `1.0.0`) is a decision the owner makes
explicitly.

**Never change the version without explicit user confirmation**; running this skill, or an orchestrator
that calls it, is that confirmation.

## Instructions

### Step 0: Parse arguments

1. The first argument is the target version. Accept `0.1.0` or `v0.1.0` and normalize to `0.1.0`.
2. The remaining arguments, split on `;`, become the changelog bullets.
3. Validate that the version matches `A.B.C` (digits only) and that `vA.B` is a real phase (`CLAUDE.md`
   §Delivery workflow).

### Step 1: Verify prerequisites

1. **Branch:** note the current branch.
2. **Clean tree:** check `git status`. If it is dirty, ask whether to include the uncommitted changes.
3. **Current version:** read `config/version` under `[application]` in `project.godot`, else the latest
   `v*` tag (`git describe --tags --abbrev=0`), else treat it as none.
4. **No downgrade:** refuse if the target is less than or equal to the current version (compare as the
   `A.B.C` tuple; phases release in order).
5. **Phase releases** (`A.B.0`): check the phase's execution report
   (`specification/implementation/vA.B-execution-report.md`) has no failed, skipped or `awaiting owner`
   issues. Its manual checks must be confirmed by the owner. If they aren't, stop and say what is missing;
   the owner can waive it explicitly, and the waiver goes into the release notes.

### Step 2: Generate the changelog (if not provided)

1. Collect the commits since the last tag (`git log --oneline <tag>..HEAD`), or all commits if there is no
   tag.
2. Summarize them as concise bullets: group related commits and reference the phase and `HANOI-###` ids.
3. Show the changelog to the user and ask for confirmation.

### Step 3: Update the version files

Touch only files that exist, except `RELEASE.txt`, which is created if missing.

1. **`project.godot`:** `config/version="<version>"` under `[application]`; add the line if it is missing.
   If the Godot editor has the project open, tell the owner to reload it, since the editor writes
   `project.godot` back on save.
2. **`export_presets.cfg`** (from v0.2, once the Android preset exists): in that preset's options, set
   `version/name="<version>"` and increase `version/code` by 1. The code must only grow: Android refuses to
   install a lower code over a higher one. Check the key names in the file; they change between Godot
   versions (concept document §10).
3. **`README.md`:** update it only if it already carries a version string. Never add one.
4. **`CLAUDE.md`:** update the `Latest release:` line under **Project status**, e.g.
   `Latest release: v0.1.0 (phase v0.1 — screen version).` On the first release, add the line at the end of
   that section.
5. **`RELEASE.txt`:** prepend a block at the top (after any header) and keep the older entries unchanged:

   ```
   Version <version> (YYYY-MM-DD)
   ---------------------------
   - <changelog item 1>
   - <changelog item 2>
   ```

No emoji in any of these files.

### Step 4: Commit

Stage only the version files that **exist**. `git add` is fatal on a pathspec that matches nothing:

```bash
for f in project.godot export_presets.cfg RELEASE.txt README.md CLAUDE.md; do
  if [ -e "$f" ]; then git add "$f"; fi
done
```

Use the `if` form, not `[ -e "$f" ] && git add "$f"`. The latter leaves the loop's exit status at 1 when the
last file is absent.

```bash
git commit -m "$(cat <<'EOF'
Release v<version>

<1-2 sentence summary of what this release includes>

Co-Authored-By: <the running model's trailer> <noreply@anthropic.com>
EOF
)"
```

### Step 5: Tag

```bash
git tag -a v<version> -m "<one-line summary of the release>"
```

### Step 6: Push

If a remote exists, push the branch and then **only the tag just created**, never `--tags` or
`--follow-tags`:

```bash
git push
git push origin "v<version>"
```

Without a remote, say that the release is local only.

### Step 7: Deploy — automatic, never asked

- **A phase that runs on the headset** (v0.2 and later, or any phase with an Android preset in
  `export_presets.cfg`): run **§Deploy** below from the tagged commit. A patch is deployed the same way as a
  phase.
- **v0.1** (desktop only): there is nothing to install. The release is the tagged commit; the report says
  `Deployed: n/a (desktop build, run with godot --path .)`.

### Step 8: Report

```
Released v<version>
  Branch: <branch>
  Commit: <short hash>
  Tag:    v<version>
  Pushed: yes / no (no remote)
  Deployed: build/hanoi-vr.apk (version code <n>) installed at HH:MM / n/a (desktop) / not deployed (why)
  Files updated:
    - project.godot
    - export_presets.cfg
    - RELEASE.txt
    - CLAUDE.md
```

## Deploy (the procedure every delivery skill uses)

Building and installing on the headset is Claude's job, done without asking. The owner only tests what is
installed.

1. **adb:** use `adb` from PATH, else `$ANDROID_HOME/platform-tools/adb`, else
   `~/Library/Android/sdk/platform-tools/adb` (the Android Studio default on macOS). If none exists, stop
   and report that the Android SDK platform-tools are missing (concept document §8).
2. **The headset:** `adb devices` must list exactly one device in state `device`.
   - `unauthorized`: ask the owner to put on the headset and accept **Allow USB Debugging**.
   - No device: ask the owner to connect the headset by USB-C and wake it.
   These are owner actions, not a question about whether to deploy. If the owner can't do it now, the
   build is `not deployed (headset not connected)` and the report says so.
3. **Build:** `mkdir -p build`, then
   `godot --headless --path . --export-debug "<Android preset name from export_presets.cfg>" build/hanoi-vr.apk`.
   The export fails the deploy if its output has an `ERROR:` line or the APK was not written. The
   prototype is installed as a debug build, signed with the debug keystore from Editor Settings.
4. **Install:** `adb install -r build/hanoi-vr.apk`. `INSTALL_FAILED_VERSION_DOWNGRADE` means
   `version/code` went down: stop and report.
5. **Check:** `adb shell pm list packages <package/unique_name from export_presets.cfg>` lists the app. Tell
   the owner it is under **Unknown Sources** in the headset library. After the owner starts it, read
   `adb logcat -d -s godot` and report the count of `ERROR` and `SCRIPT ERROR` lines.
6. **Record:** say in the report what was installed (version name and code) and when. A failed deploy is
   reported with what failed; never uninstall or roll back on your own.

`build/` and the Android build template `android/` are never committed.

## Important Rules

- **Never downgrade.** Refuse a target less than or equal to the current version; never lower
  `version/code`.
- **Clean tree first.** If there are uncommitted changes, ask before proceeding.
- **Phase releases need a complete phase.** No failed, skipped or `awaiting owner` issues, and the manual
  checks confirmed by the owner (or explicitly waived, and noted).
- **Annotated tags only** (`git tag -a`).
- **Version metadata only.** This skill touches `project.godot`'s `config/version`, the Android preset's
  version fields, `RELEASE.txt`, `README.md` and CLAUDE.md's status line, never game code.
- **Confirm the changelog** when it is auto-generated.
- **Deploy every headset release, never ask** (§Deploy). The only deploy-related pauses are a headset that
  isn't connected or authorized, and a failed export or install, reported as such.
- **Secrets stay out.** Keystore passwords and paths are never printed, logged or committed.
- **Plain-text release notes** in `RELEASE.txt`.
