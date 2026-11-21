# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Tower of Hanoi prototype for Meta Quest 3, built in Godot 4 with GDScript. Phase 1 is a desktop build controlled by mouse. Phase 2 adds Touch controller input and an Android build for the Quest 3.

The source of truth is the concept document: [specification/hanoi-vr-concept.en.md](specification/hanoi-vr-concept.en.md) in English, with the Ukrainian original in [specification/hanoi-vr-concept.md](specification/hanoi-vr-concept.md). Both have the same content; when one changes, update the other. It holds the build steps, acceptance criteria, scene dimensions (section 5) and out-of-scope items (section 9). Read it before starting any step. As of this writing, no Godot project exists yet; work starts at Phase 1, step 1.

## Working rules from the spec

- Implement the steps of sections 6 and 7 one at a time. After each step, stop, say briefly what was done and how to check it, and wait for the user to confirm before starting the next step. The project must run after every step.
- GDScript with static typing everywhere. Identifiers and comments are in English.
- No emoji in code, comments, in-game text or documentation.
- Before Phase 2, run `godot --version` and check the current Godot "Deploying to Android" page and the Godot OpenXR Vendors plugin README. OpenXR APIs and export setting names change between Godot versions, so do not rely on memory for them.

## Commands

`godot` is on PATH (Homebrew). Run commands from the repository root, where `project.godot` will live.

```sh
godot --path .                                     # run the game (desktop mode on a Mac)
godot -e --path .                                  # open the editor
godot --headless --script tests/test_hanoi_state.gd  # run the core test (command fixed by the spec)
godot --headless --path . --import                 # rebuild the import and class_name cache
```

Global `class_name` lookups in a headless `--script` run depend on the cache under `.godot/`. On a fresh clone, or after adding a new `class_name`, run the `--import` command first; without it the test fails with a parse error. Godot 4.4+ writes a `.uid` file next to every script; commit it with the script. Never commit `.godot/`, `build/` or `android/`.

Phase 2 deploy (needs Android SDK platform-tools on PATH for `adb`, OpenJDK 17, and the headset in Developer Mode connected by USB-C):

```sh
adb devices                                        # check that the headset is connected
godot --headless --path . --export-debug "<preset name>" build/hanoi-vr.apk
adb install -r build/hanoi-vr.apk                  # app appears under Unknown Sources
adb logcat -s godot                                # read the game log
```

macOS has no Quest Link, so VR mode can only be tested by installing a build on the headset. All game logic must therefore be verified in Phase 1.

## Acceptance gates

Godot 4.7 exits with code 0 on a parse error, a runtime error and a failed `assert()`, both in a `--script` run and in a scene run. A gate therefore passes only when its output has no `SCRIPT ERROR` or `ERROR:` line, and, for a test, the exit code is 0.

```sh
godot --headless --path . --import                         # 1. import: class_name cache and .uid files
for t in tests/test_*.gd; do                               # 2. tests: every headless test script
  out=$(godot --headless --xr-mode off --path . --quit-after 600 --script "$t" 2>&1); rc=$?
  if [ $rc -ne 0 ] || printf '%s\n' "$out" | grep -qE 'SCRIPT ERROR|ERROR:'; then echo "FAIL $t"; printf '%s\n' "$out"; fi
done
godot --headless --xr-mode off --path . --quit-after 120 2>&1 | grep -E 'SCRIPT ERROR|ERROR:'   # 3. smoke: must print nothing
```

- **Test scripts** (`tests/test_*.gd`) extend `SceneTree`, run their checks in `_init()` (with `assert()` or explicit checks), and end with `quit(0)` or `quit(1)`. An error before `quit()` leaves the process running; `--quit-after 600` ends it after about 5 s, and the error line fails the gate.
- **`--xr-mode off`:** once OpenXR is enabled in the project settings (Phase 2), a run on the Mac without this flag prints an OpenXR `ERROR:` and never exits. With it, `XRServer.find_interface("OpenXR")` returns the interface but not initialized, so `main.gd` falls back to the desktop rig.
- **Before `project.godot` exists**, the gates are `n/a`, not passed.
- **Manual gates (owner):** playing the game with the mouse (Phase 1), playing it in the headset with two controllers at 72 fps or more (Phase 2), and each step's "how to check". A manual check counts as passed only once the owner confirms it.

## Delivery workflow (skills)

`.claude/skills/` holds a spec-driven pipeline adapted from the matrix-agora project.

- **Phases:** the concept document's Phase N is phase `v0.N`: `v0.1` = §6 (screen version), `v0.2` = §7 (port to Quest 3). A phase added to the document later gets the next id.
- **Issues:** each phase becomes `specification/implementation/v0.N-issues.md`, one issue per numbered step. Issue ids are `HANOI-###`, numbered globally and never reset.
- **Implementation:** each issue is implemented, gated, then reported to the owner with how to check it; Claude waits for the owner's confirmation (concept document §10) and makes one commit per issue. Each phase is then reviewed, and the fix-now findings are fixed.
- **Releases:** phase `v0.N` ships as `0.N.0` (tag `v0.N.0`); post-release fixes bump the last digit (`0.N.1`, ...). The version lives in `project.godot` (`config/version`) and, from Phase 2, in the Android preset's `version/name` and `version/code`; release notes go to `RELEASE.txt`.
- **Deploy:** from Phase 2, a build is exported as a debug APK and installed with `adb install -r`, automatically and without asking, whenever the headset is connected (`release-version` §Deploy). Phase 1 has nothing to deploy.
- **GitHub flow:** `/ship-phase <selector>` runs `/generate-issues` → `/upload-issues` → `/execute-issues` → deploy → the owner tests → `/review-and-fix-issues` → `/release-version` for each phase, then `/harden-findings` at the end of the run.
- **Offline flow:** `/ship-solution` runs `/reconcile-issues` → `/execute-issues-file` → deploy → the owner tests → `/review-and-fix-issues` → `/release-version` over issues files that already exist.
- **Contracts:** concept document §3 (key decisions), §4.1 (core API), §4.2 (board interface), §4.4 (mode selection) and §5 (dimensions). Changing one updates both concept document files and the test that pins it, in the same commit.
- **Specification:** `/generate-spec` writes `specification/VISION.md`, `ARCHITECTURE.md` and `ROADMAP.md` from the source documents, in the matrix-agora format (`.claude/skills/generate-spec/FORMAT.md`, with examples in `examples/`); `/plan-phase` adds a version or phase to them. Until the three files exist and the delivery skills are repointed to them, the concept document stays the source of truth.
- **Who releases:** versions are bumped and tagged only by `/release-version`, `/ship-phase`, `/ship-solution` or `/harden-findings --release`.

## Architecture

The planned layout (spec section 4) is a pure game-state core, one 3D board, and two interchangeable input layers. Game logic is written once and is not changed for Phase 2.

- `core/hanoi_state.gd` is a `RefCounted` class with no scene nodes. It holds three arrays of ints (disk sizes; the last element is the top disk) and exposes `reset(disk_count)`, `top_disk`, `can_take`, `can_place`, `move(from, to) -> bool`, `is_solved`, `min_moves` (2^n - 1), the property `move_count`, and the signals `moved(disk, from_peg, to_peg)`, `solved()`, `reset_done()`. Moving a disk to the peg it came from is not a move.
- `board/hanoi_board.gd` is a `Node3D` that builds the table, pegs and disks in code and keeps them in sync with the core through its signals. It is the only interface the input layers use: `try_begin_drag(disk_node)`, `update_drag(world_position)`, `end_drag()`, `press_button(button_id)`. `end_drag()` picks the nearest peg within 0.15 m horizontally, calls `move` on the core, and tweens the disk to its new place or back to its source peg. While a disk is dragged, the peg under it is highlighted green if the move is legal and red if not. Each disk has an `Area3D` so that both the mouse ray and the controller hand area can detect it. `board/hud.gd` owns the `Label3D` move counter, the win message and the three buttons (reset, fewer disks, more disks; disk count 3 to 7).
- `input/desktop_rig` (Phase 1): fixed `Camera3D`; left mouse button ray-picks a disk, mouse motion moves it in the vertical plane through the peg line, release ends the move; key R resets, keys 3 to 7 set the disk count.
- `input/xr_rig` (Phase 2): `XROrigin3D`, `XRCamera3D`, two `XRController3D` with sphere meshes and an `Area3D` of about 6 cm radius. Grip grabs and releases a disk; trigger presses a table button while the hand is inside it; a short haptic pulse on a successful grab and on a rejected move.
- `main.gd` chooses the mode at startup: if `XRServer` finds an initialized OpenXR interface, it sets `use_xr` on the main viewport, turns off vsync and adds `xr_rig`; otherwise it adds `desktop_rig`. This lets the same project run from the editor on a Mac and on the headset.

## Fixed technical decisions

- No physics bodies. Do not use `RigidBody3D`. A dragged disk follows the cursor or hand directly, and on release it is moved to its target by a tween (0.2 s).
- No asset files (models, textures, fonts). All visuals are Godot primitives (`CylinderMesh`, `BoxMesh`, `PlaneMesh`, `Label3D`) with `StandardMaterial3D`, created in code. The floor grid comes from a shader or procedural material; the sky is a `ProceduralSkyMaterial`; lighting is one `DirectionalLight3D` with shadows off.
- Renderer: Compatibility (recommended by Godot for Android headsets).
- Units are meters with real-world sizes, so the scene has the correct scale in VR. Use the values from spec section 5.
- No external libraries in Phase 1. In Phase 2, the only addition is the Godot OpenXR Vendors plugin. Grabbing is a short custom script, not Godot XR Tools.
- Phase 2 Android export preset: Gradle Build on, arm64-v8a, XR Mode OpenXR, Meta selected under XR Features. Target is at least 72 fps on the headset.
