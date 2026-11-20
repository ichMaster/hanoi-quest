# Hanoi VR: concept document for Claude Code

This document describes Vitalii's first prototype for Meta Quest 3. It is intended for Claude Code, which will build the project step by step. The author works on macOS in VS Code and already has experience with Godot 4.

## 1. Goal

The goal of the prototype is to go the whole way from an empty project to an application that runs on the Quest 3 headset and is controlled with the controllers. Tower of Hanoi was chosen as the game because it has simple rules, a clear win state, and needs no external assets.

Development proceeds in two phases. In the first phase, the game runs on the Mac screen and is controlled with a mouse, which gives a fast test cycle without the headset. In the second phase, a controller input layer is added to the same project, and the game is built for Quest 3. The game logic is not rewritten.

## 2. What the player sees and does

The player stands in front of a table with three vertical pegs. Four disks of different sizes and colors are stacked on the left peg, with the largest at the bottom. The player takes the top disk of any peg, carries it and releases it near another peg, after which the disk smoothly settles into place.

Only the top disk can be taken. If the player tries to place a larger disk on a smaller one, or releases the disk far from the pegs, the disk smoothly returns to the peg it was taken from, and the move is not counted. Moving a disk back onto the same peg is not a move either.

Above the table there is a label with the move counter and the minimum possible number of moves, which equals 2^n - 1. When the whole tower is on the right peg, the label changes to a win message. Next to the table there are three buttons: reset the game, decrease the number of disks, and increase the number of disks, within the range from three to seven.

## 3. Key decisions

Disks are not physics bodies. Physics of rings on pegs in Godot is unstable, so while being carried a disk simply follows the hand or the cursor, and on release it is moved to the target position by a tween. RigidBody3D is not used in the project.

All visual elements are created in code from Godot primitives (CylinderMesh, BoxMesh, PlaneMesh, Label3D) and StandardMaterial3D. The project contains no model, texture or font files.

The Compatibility renderer is used because the Godot documentation recommends it for Android headsets. The unit of measurement is the meter, and all sizes are real-world values, so the scene has the correct scale in VR from the start.

No external libraries are needed in the first phase. In the second phase, grabbing is implemented with a short custom script, without Godot XR Tools, because the mechanic comes down to attaching the disk to the hand.

## 4. Architecture

The project consists of a core, a board, and two interchangeable input layers.

```
project.godot
main.tscn / main.gd            entry point, mode selection
core/hanoi_state.gd            pure logic without scene nodes
board/hanoi_board.gd           3D representation: table, pegs, disks, animations
board/hud.gd                   Label3D with the counter, and the buttons
input/desktop_rig.tscn + .gd   camera and mouse control (phase 1)
input/xr_rig.tscn + .gd        XROrigin3D and controllers (phase 2)
tests/test_hanoi_state.gd      core test in headless mode
```

### 4.1. Core: `core/hanoi_state.gd`

This is a class based on RefCounted that knows nothing about the scene. It stores three arrays of integers, where each number is the size of a disk and the last element of an array is the top disk.

The class provides the following methods: `reset(disk_count)`, `top_disk(peg)`, `can_take(peg)`, `can_place(disk, peg)`, `move(from_peg, to_peg)` returning a success flag, `is_solved()`, `min_moves()`. It also has the property `move_count` and the signals `moved(disk, from_peg, to_peg)`, `solved()` and `reset_done()`.

### 4.2. Board: `board/hanoi_board.gd`

The board is a Node3D that builds the table, pegs and disks procedurally and keeps them in sync with the core state. It gives the input layers a single interface, the same for the mouse and the controller:

- `try_begin_drag(disk_node)` returns true if the disk is on top and can be taken;
- `update_drag(world_position)` moves the taken disk to the given point in space;
- `end_drag()` finds the nearest peg within the threshold, calls `move` on the core, and starts the settle or return animation;
- `press_button(button_id)` handles the reset and disk count buttons.

Each disk has an Area3D with a collision shape, so that both the mouse ray and the controller hand area can find it. While a disk is being carried, the peg it is currently above is highlighted green if the move is allowed and red if it is not.

### 4.3. Input layers

`desktop_rig` contains a fixed Camera3D aimed at the table. Pressing the left mouse button casts a ray and takes a disk, mouse movement moves the disk in the vertical plane that passes through the line of pegs, and releasing the button completes the move. The R key resets the game, and keys 3 to 7 set the number of disks.

`xr_rig` contains XROrigin3D, XRCamera3D and two XRController3D nodes, with simple spheres instead of hand models and an Area3D with a radius of about 6 cm. Pressing grip takes the disk that is inside the hand area, and releasing grip completes the move. The buttons on the table are pressed with the trigger while the hand is inside the button area. On a successful grab and on a rejected move, the controller vibrates briefly.

### 4.4. Mode selection: `main.gd`

At startup, the script looks for the OpenXR interface through XRServer. If the interface is found and initialized, `use_xr` is enabled on the main viewport, vsync is turned off, and `xr_rig` is added. Otherwise, `desktop_rig` is added. This way the same project runs both from the editor on the Mac and on the headset.

## 5. Scene dimensions

| Element | Value |
| --- | --- |
| Tabletop height | 0.80 m |
| Tabletop | 1.10 x 0.45 x 0.04 m |
| Distance between pegs | 0.33 m |
| Peg | height 0.26 m, radius 0.008 m |
| Disk thickness | 0.03 m |
| Disk radius | from 0.05 m to 0.14 m, evenly spaced by size |
| Peg snap threshold | 0.15 m horizontally |
| Player distance from table center | 0.55 m |
| Counter label | 0.55 m above the tabletop, behind the pegs |
| Settle animation duration | 0.2 s |

The floor is a 6 x 6 m plane with a simple grid drawn by a shader or a procedural material. The sky is set with ProceduralSkyMaterial, and the lighting is one DirectionalLight3D without shadows.

## 6. Phase 1: screen version

The steps are done in the following order, and the project must run after each of them.

1. Create a Godot 4 project with the Compatibility renderer and the folder structure from section 4.
2. Implement `hanoi_state.gd` and a test that solves the tower recursively for n from 3 to 7 and checks the number of moves, the rejection of illegal moves, and the win state. The test must run with the command `godot --headless --script tests/test_hanoi_state.gd`.
3. Implement the board with procedural scene construction and animations.
4. Implement `desktop_rig` with mouse control.
5. Add the counter, the win message and the buttons.

The phase is complete when the game can be played with the mouse from start to win, illegal moves are rejected and the disk returns, reset and changing the number of disks work, and the core test passes.

## 7. Phase 2: port to Quest 3

1. Enable OpenXR in the project settings (XR, OpenXR, Enabled) and XR Shaders.
2. Install the Godot OpenXR Vendors plugin from the Asset Library and install the Android build template.
3. Implement `xr_rig` and the mode selection logic in `main.gd`.
4. Create an Android export preset: Gradle Build enabled, architecture arm64-v8a, XR Mode set to OpenXR, Meta selected in the XR Features section.
5. Build the APK and install it on the headset.
6. In the headset, adjust the table height, the distance to the table and the grab area radius.

The phase is complete when the game can be played in the headset with two controllers, seated or standing, at no less than 72 frames per second.

## 8. Environment and deployment

The Mac needs the current stable version of Godot 4, OpenJDK 17 and the Android SDK (easiest to get through Android Studio). The Android SDK path is set in Editor Settings, under Export, Android.

Installing on the headset requires a Meta developer account with a team (organization) created. After that, Developer Mode for the headset is enabled in the Meta Horizon mobile app. The headset is connected to the Mac with a USB-C cable, and the Allow USB Debugging prompt is confirmed in the headset. The connection can be checked with the command `adb devices`.

The build can be run with the one-click deploy button in the Godot editor or installed manually with the command `adb install -r build/hanoi-vr.apk`. The application appears in the headset library under Unknown Sources. Logs are read with the command `adb logcat -s godot`.

macOS has no Quest Link, so VR mode can only be tested by installing a build on the headset. This is why all the logic must be verified in the first phase.

## 9. Out of scope

The prototype deliberately excludes passthrough and mixed reality, hand tracking without controllers, stick locomotion and teleportation, sound, menus, saving results, and publishing to the Meta Horizon Store. Each of these items can become a separate next step.

## 10. Instructions for Claude Code

The OpenXR API in Godot and the names of export settings change between versions. Before the second phase, find out the installed Godot version with the command `godot --version` and check the current "Deploying to Android" documentation page and the Godot OpenXR Vendors plugin README, instead of relying on memory.

Code is written in GDScript with static typing. Identifiers and comments in the code are in English. Emoji are not used in code, comments, in-game text or documentation.

After each step from sections 6 and 7, stop, briefly report what was done and how to check it, and wait for confirmation before starting the next step.
