---
name: scene-setup
description: >-
  Guides Editor-side scene configuration for sollertia-virtual-reality: the consolidated Task
  Parameters window, the three-monitor Display rig and its displayplacer / xrandr monitor-enumeration
  prerequisite, the LinearTreadmill / SimulatedLinearTreadmill controller swap, and the optional
  UI-lick-reward canvas. Use when preparing a scene for Play Mode, swapping to the simulated
  treadmill, or fixing missing display / controller / monitor-detection errors.
user-invocable: false
---

# Sollertia Unity scene setup

Covers the **Editor-time** configuration that turns a freshly created scene into one that can
actually run a task. Generation, Play Mode, asset enumeration, and the Parameters MCP surface
live in sibling skills; this skill owns the human / GUI flow between "scene exists" and "scene is
runnable."

---

## Scope

**Covers:**
- Opening the consolidated **Task Parameters** window (`Window → Task Parameters`)
- Auto-created scene infrastructure (`Actors`, `Controllers`, `MQTT Client`, default Actor +
  Display) seeded by `MainWindow.InitializeScene`
- Three-monitor VR setup (Left / Center / Right View) defined here for downstream acquisition rigs
- The `displayplacer` (macOS) / `xrandr` (Linux) monitor-enumeration prerequisite Camera Mapping needs
- Swapping between `LinearTreadmill` (hardware) and `SimulatedLinearTreadmill` (keyboard) via the
  Actor section's Controller dropdown
- Brightness / VR height tuning via the Display section
- MQTT broker IP / port via the MQTT section (project-wide; persisted in `EditorPrefs`)
- Task-component fields (`Require Interaction`, `Require Wait`, `Track Length`, `Track Seed`) via the
  Task section
- Adding the `UI-lick-reward` canvas subsystem for experimenter feedback
- Scene-specific vs project-wide configuration state
- Pre-Play Mode checklist

**Does not cover:**
- Programmatic read / write of the Parameters window (see `/task-parameters`)
- Enumerating assets (see `/task-scenes`)
- Creating the scene and generating the task prefab dropped into it (see `/task-prefabs`,
  `/task-generator`)
- Entering / exiting Play Mode (see `/play-mode`)
- GIMBL class APIs (see `/gimbl-framework`)
- MQTT topic details (see `/mqtt-contract`)

---

## The Task Parameters window

`Window → Task Parameters` opens the single editor window that hosts every per-scene
configuration surface (`Actor`, `MQTT`, `Display`, `Camera Mapping`, `Task`). It is the only
GUI entry point for these fields — every public field on the `Task` component is individually
marked `[HideInInspector]`, and `TaskEditor` (a `[CustomEditor(typeof(Task))]`) replaces the
default Inspector with a HelpBox that points at this window.

### Auto-open behavior

`MainWindow` registers a one-shot `EditorApplication.delayCall` plus subscriptions to
`EditorSceneManager.sceneOpened` and `EditorApplication.playModeStateChanged` so the window
reappears automatically after:

- Editor start / domain reload
- Any scene open (including `open_scene_tool` from `/task-scenes`)
- Entering Play Mode

None of these hooks are registered in a batch-mode editor (`Application.isBatchMode`), so a headless
/ CI run never auto-opens the window and never runs `InitializeScene` against the throwaway startup
scene, whose unsaved changes would otherwise block the run on a save dialog batch mode cancels.

If the user closes it manually, opening any of the above events brings it back. The docked tab
label is `Parameters`; the menu entry is `Window → Task Parameters` to disambiguate.

### Auto-created scene infrastructure

Opening the window when no instance is open runs `OnEnable` and, through it, `InitializeScene`,
which ensures the active scene contains the following before the GUI renders. Existing objects are
left untouched, and missing ones are created. Opening a scene while the window is already open
skips that pass, because the `EditorSceneManager.sceneOpened` hook returns early once an instance
exists. A scene opened that way, including through `open_scene_tool`, gains the infrastructure once
the window is closed and reopened, or after the next domain reload. The window's own HelpBox says
the same thing, "Close and reopen this window to auto-create one".

| GameObject            | Components / behavior                                                                    | Hidden in hierarchy     |
|-----------------------|------------------------------------------------------------------------------------------|-------------------------|
| `Actors`              | Empty root for `ActorObject` instances                                                   | No                      |
| `Controllers`         | Empty root for controller GameObjects                                                    | No                      |
| `MQTT Client`         | `Gimbl.MQTTClient` singleton                                                             | Yes (`HideInHierarchy`) |
| `Actor` (default)     | `ActorObject` with the first prefab under `Resources/Actors/Prefabs/`                    | No                      |
| `<Display>` (default) | `DisplayObject` from the first prefab under `Resources/Displays/`, parented to the actor | No                      |
| `Linear`              | `LinearTreadmill` + `ControllerOutput`                                                   | No                      |
| `Simulated Linear`    | `SimulatedLinearTreadmill` + `ControllerOutput`                                          | No                      |

**Both controllers coexist permanently** under the `Controllers` root — the active controller is
selected via the Actor dropdown, not by adding or removing scripts. The Unity-default
`Main Camera` is removed automatically because the Display owns the per-monitor cameras and the
Actor owns the third-person tracking camera.

For the underlying invariants — `MainWindow.InitializeScene` / `EnsureControllers` /
`EnsureMqttDefaults` / `SyncDisplayBrightnessToSettings` mechanics, the `HideInHierarchy` flag on
`MQTT Client`, the `delayCall` autoload, and how `CreateTask.CreateSceneFromTemplate` calls the
same helpers synchronously at scene creation — see `/gimbl-framework` "MainWindow".

### Required scene contents (post-init)

A runnable scene contains:

| GameObject                    | Component                                                           | Purpose                                                 |
|-------------------------------|---------------------------------------------------------------------|---------------------------------------------------------|
| `MQTT Client`                 | `Gimbl.MQTTClient`                                                  | Broker singleton; sets `MQTTClient.Instance` on `Awake` |
| `Logger`                      | `Gimbl.MQTTConnectorObject`                                         | Calls `MQTTClient.Instance.Connect` on `OnEnable`; without it the scene never reaches the broker |
| `Actor` (or renamed)          | `Gimbl.ActorObject`                                                 | Animal avatar with display + controller                 |
| `<Display>`                   | `Gimbl.DisplayObject`                                               | Multi-monitor rig with per-monitor cameras              |
| `Linear` / `Simulated Linear` | `LinearTreadmill` / `SimulatedLinearTreadmill` + `ControllerOutput` | Input devices; pick one via Actor dropdown              |
| `<Task prefab instance>`      | `SL.Tasks.Task`                                                     | Corridor hierarchy (dropped in from `Tasks/`)           |
| `UI-Control` (optional)       | `SL.UI.LickStimulusSpawner`                                         | On-screen interaction and stimulus indicators           |

`ExperimentTemplate.unity` ships the `UI-lick-reward` canvas as a `UI-Control` prefab instance and
the hardware `Linear` controller, but **not** `Simulated Linear`. Add the task prefab manually, or
call `create_task_tool` from `/task-prefabs`, whose `CreateTask.CreateSceneFromTemplate`
instantiates the task prefab and then calls `MainWindow.EnsureControllers`, which creates the
missing `Simulated Linear` GameObject (reported back as `SceneCreationResult.SimulatedControllerAdded`).

---

## Display rig configuration

This project defines a three-monitor VR setup arranged around the animal's viewing position;
downstream acquisition rigs adopt this layout. The Display section of the Parameters window
controls brightness and VR height; the **Camera Mapping** section binds the three per-monitor
cameras (`Left View`, `Center View`, `Right View`) to OS monitor indices.

**Scope boundary.** This skill configures the project's standard three-monitor corridor rig. Authoring a *different*
Display rig or a non-corridor scene topology has no author-derived recipe — escalate to the human supervisor and
co-design it in a generative, collaborative mode. You MUST NOT hand-author a new rig or scene topology autonomously.

### Monitor-enumeration prerequisite

Camera Mapping rows are built from `Monitor.EnumerateMonitors`, not from `InitializeScene`. Windows
enumerates through `user32.EnumDisplayMonitors` and needs nothing extra. macOS requires
[`displayplacer`](https://github.com/jakehilborn/displayplacer) (`brew install displayplacer`),
resolved in order from `/opt/homebrew/bin/displayplacer`, `/usr/local/bin/displayplacer`, then the
bare name on `PATH`. Linux requires `xrandr` from the X11 server utilities, resolved from `PATH`.

Without the helper the section lists **no monitors** and full-screen views cannot be assigned. The
failure surfaces as a Console *warning* rather than an exception — `Monitor enumeration: failed to
start '<command>'.`, with a `brew install displayplacer` hint appended on macOS — so every tool keeps
reporting success while returning an empty monitor list. No amount of refreshing fixes it; install
the helper first.

### Assigning monitors

1. Save the scene. An untitled scene has no persistence path, so assignments made there are lost
   (see "Per-scene state" below).
2. Open `Window → Task Parameters` and scroll to **Camera Mapping**.
3. Click **Refresh Monitor Positions** if the entries do not match the OS-reported monitors. The
   agentic counterpart is `refresh_monitors_tool` from `/task-parameters`, which shares
   `FullScreenViewManager.RefreshMonitorPositions` with the button so both paths re-detect
   identically; existing camera assignments carry across by monitor index.
4. For each row, pick the matching camera from the dropdown (the Display rig auto-names cameras
   after the role, e.g., `Left View`, `Center View`, `Right View`).
5. Enter Play Mode (`/play-mode`). `MainWindow.OnPlayModeStateChanged` calls
   `ShowFullScreenViews(closeOldViews: false)` on `ExitingEditMode`, so a borderless window opens on
   each assigned monitor automatically; the **Show Full-Screen Views** button is disabled while
   playing. Verify each monitor shows its side of the VR corridor, then exit Play Mode — each
   `FullScreenView` closes itself on `ExitingPlayMode`. **You SHOULD NOT** press the button in edit
   mode first: the Play-Mode pass does not close pre-existing views, so you end up with two stacked
   windows per monitor.
6. If an assignment is wrong, swap entries and re-verify.

An edit-mode full-screen view is not blank — `FullScreenView.OnGUI` renders its bound camera on every
`Repaint` regardless of play state. While not playing, left-clicking anywhere on a view closes that
view; that is the intended way to dismiss edit-mode views. During Play Mode clicks pass through and
the views close only on Play Mode exit or editor quit.

### Reboot caveat

Operating-system reboots can reorder monitor output ports. **You MUST** re-verify the
camera-to-monitor binding before starting an experimental session. The Camera Mapping section's
monitor indices are not stable across reboots.

### Display section

The Display section exposes **two editable fields** (`brightness`, `heightInVR`) plus a Blank /
Show button. The relevant state values are:

| Value               | Surface                                                  | Persistence                                                                   | Effect                                        |
|---------------------|----------------------------------------------------------|-------------------------------------------------------------------------------|-----------------------------------------------|
| `brightness`        | Numeric field (`DisplaySettings` initializer `50`)       | `Assets/VRSettings/Displays/<display name>.asset`                             | Default brightness restored by "Show Display" |
| `heightInVR`        | Numeric field (default `0.2`)                            | `Assets/VRSettings/Displays/<display name>.asset`                             | Y offset of the display rig from the actor    |
| `currentBrightness` | No field, written by Blank / Show and `brightness` edits | Scene-serialized on `DisplayObject`, synced to `brightness` on scene creation | Live brightness override applied to rendering |

The Blank / Show button flips `currentBrightness` between `0` and the configured `brightness`.
Editing the `brightness` field writes the new value into `currentBrightness` as well, so a
brightness edit reaches the live display right away, without a "Show Display" press.
`CreateTask.CreateSceneFromTemplate` calls `MainWindow.SyncDisplayBrightnessToSettings` after the
scene is instantiated, so a freshly created scene's `currentBrightness` matches the asset's
`brightness` rather than the `DisplayObject` field initializer. `DisplayObject.Create` also reuses
an existing `<displayName>.asset` at `Assets/VRSettings/Displays/` instead of overwriting it, so
user-customized `brightness` / `heightInVR` survive subsequent scene rebuilds — which is why the
`DisplaySettings` initializer of `50` is not what a checkout sees: the project's committed
`Assets/VRSettings/Displays/Display.asset` ships `brightness: 100`, and `Create` reuses it rather
than resetting it.

### Per-scene state

Camera Mapping assignments are **scene-specific** and persisted in
`Assets/VRSettings/Displays/<scene-name>-savedFullScreenViews.asset`. Every new scene (including
scenes created via `create_task_tool`) must have its cameras bound once.

An untitled (never-saved) active scene has no persistence path, so `LoadCameras` skips the asset
entirely and `SaveCameras` silently no-ops — assignments made there are lost. `SaveCameras` also
no-ops when zero monitors are detected, so a host missing `displayplacer` / `xrandr` cannot overwrite
an existing mapping with an empty one. **You MUST** save the scene before binding cameras.

`brightness` and `heightInVR` are stored on the **DisplaySettings asset**, keyed by the display
**GameObject name** rather than by the model prefab: `DisplayObject.Create` writes to
`Assets/VRSettings/Displays/<display GameObject name>.asset`, and `MainWindow.EnsureActorAndDisplay`
always names the auto-created display `Display`. Every scene in the project therefore shares the
single `Assets/VRSettings/Displays/Display.asset`, regardless of which `Resources/Displays/` prefab
supplied the geometry. The MQTT broker `ip` and `port` are stored in `EditorPrefs`
(`SollertiaVR_MQTT_IP` / `SollertiaVR_MQTT_Port`) and apply project-wide.

---

## Choosing an actor model

The `Actor` section's **Model** dropdown lists every prefab under `Assets/Gimbl/Resources/Actors/Prefabs/`
plus the literal `None`. The shipped project offers only `Rodent`. Selecting a different entry destroys the
existing `Model <name>` child and instantiates the chosen prefab in its place, so any manual edit made to the
old model child is lost. `None` leaves the actor without a visible mesh, which is valid for a headless
controller test but renders nothing in the display cameras.

---

## Choosing a controller

The `Actor` section's **Controller** dropdown picks among every `ControllerOutput` in the active
scene plus the literal `None`. After scene init the auto-created options are:

| Dropdown label      | Underlying script           | When to use                                              | Input source              |
|---------------------|-----------------------------|----------------------------------------------------------|---------------------------|
| `Linear`            | `LinearTreadmill`           | Running against real treadmill hardware                  | MQTT `Motion` topic       |
| `Simulated Linear`  | `SimulatedLinearTreadmill`  | Manual Editor testing without hardware                   | Keyboard only             |
| `None`              | (no controller)             | Disable actor movement (rare; debugging)                 | n/a                       |

Both controller GameObjects always exist in every scene; the dropdown only swaps which one the
actor reads. **You MUST NOT** add or remove the controller scripts manually — use the Actor
dropdown.

### Using `Simulated Linear` for keyboard testing

1. In `Window → Task Parameters → Actor`, set Controller to `Simulated Linear`.
2. Enter Play Mode (`/play-mode`).
3. Controls (Unity Input System asset `SimulatedInput.inputactions`, action map `Player`):
   - **W / Up arrow** — move forward
   - **S / Down arrow** — move backward
   - **Space (Jump action)** — simulate a single interaction (publishes `Interaction`).

Movement speed is scaled by `MovementSpeedMultiplier = 8.0f` in `SimulatedLinearTreadmill.cs`.

### Reverting to hardware

Before running a real session, set Controller back to `Linear` in the Actor section, confirm the
MQTT broker IP / port in the MQTT section, and verify the connection with **Test Connection**.
Leaving `Simulated Linear` selected in a production scene publishes spurious `Interaction` events on
every spacebar press, corrupting the session log.

The `Simulated Linear` GameObject **stays in the scene** — it is just unselected by the dropdown.
**You MUST NOT** delete it; future testing depends on its presence.

### Programmatic alternative

Agents can read and write the Controller dropdown via `/task-parameters`:

```text
write_task_parameters_tool(actor={"controller": "Simulated Linear"})
```

See that skill for the option-list contract and validation rules.

---

## Task section

The `Task` section exposes the task component's tunable parameters, all of which are mirrored on the
`Task.cs` `MonoBehaviour` but addressable only through this window or `/task-parameters`:

| Field                 | Type  | Effect                                                                         | Conditional rendering                    |
|-----------------------|-------|--------------------------------------------------------------------------------|------------------------------------------|
| `Require Interaction` | bool  | Interaction-guidance toggle (also mirrored over MQTT `RequireInteraction`)     | Hidden when scene has no `GuidanceZone`  |
| `Require Wait`        | bool  | Occupancy-guidance toggle (also mirrored over MQTT `RequireWait`)              | Hidden when scene has no `OccupancyZone` |
| `Track Length`        | float | Total length of the pre-generated random trial sequence (Unity units)          | Always visible                           |
| `Track Seed`          | int   | RNG seed for the random trial sequence (`-1` requests a nondeterministic seed) | Always visible                           |

Controls are **disabled in Play Mode** because the live guidance toggles are driven by MQTT during
runtime. To flip a toggle mid-run, publish on the matching MQTT topic instead (see `/mqtt-contract`).

---

## MQTT section

| Field   | Persistence                                | Default       | Notes                                                       |
|---------|--------------------------------------------|---------------|-------------------------------------------------------------|
| `ip`    | `EditorPrefs` (`SollertiaVR_MQTT_IP`)      | `127.0.0.1`   | Project-wide; every scene shares it                         |
| `port`  | `EditorPrefs` (`SollertiaVR_MQTT_Port`)    | `1883`        | Project-wide; every scene shares it                         |

The Test Connection button connects, logs the result to the Console, and immediately disconnects
so the client is not left dangling. Controls are disabled in Play Mode — broker changes mid-run
are not supported.

---

## UI-lick-reward subsystem

The `UI-lick-reward` folder provides an on-screen feedback canvas for experimenters running
interactive sessions: `LickMsg` renders an `Interaction` event, `RewardMsg` a delivered `Stimulus`.
It is **not** part of the task's behavioral contract — removing it changes nothing about the runtime
data. The folder, class, and prefab names predate the interaction-modality generalization and are
kept verbatim because they are SLVR's own identifiers; the concrete sensor behind an `Interaction`
(a lever, button, pressure plate, or contact sensor) is resolved by the acquisition runtime, not by
Unity.

### Components

| Asset                  | Purpose                                                                              |
|------------------------|--------------------------------------------------------------------------------------|
| `UI-Control.prefab`    | Canvas prefab carrying `LickStimulusSpawner`                                         |
| `LickMsg.prefab`       | Instantiated when `Interaction` is received                                          |
| `RewardMsg.prefab`     | Instantiated when a `Stimulus` arrives with `delivered == true`                      |
| `lickAnimation.anim`   | Short animation auto-played by the `LickMsg` prefab's legacy `Animation` component   |
| `rewardAnimation.anim` | Short animation auto-played by the `RewardMsg` prefab's legacy `Animation` component |

### Scripts

- `LickStimulusSpawner.cs`, the root MonoBehaviour on `UI-Control`. Subscribes to `Interaction` and
  `Stimulus` and instantiates the corresponding indicator prefab on the canvas. `OnStimulus` only
  counts a message whose `delivered` flag is set, so an omitted stimulus spawns nothing.
- `LickMessage.cs`, attached to `LickMsg`. Schedules the indicator's destruction `destroyTime`
  seconds after `Start`, while the prefab's `Animation` component plays `lickAnimation.anim`.
- `StimulusMessage.cs`, attached to `RewardMsg`. Schedules the indicator's destruction
  `destroyTime` seconds after `Start`, while the prefab's `Animation` component plays
  `rewardAnimation.anim`.

### Installing in a scene

Scenes copied from `ExperimentTemplate.unity`, which covers every scene `create_task_tool` produces, already carry a
`UI-Control` instance with `Canvas`, `Lick Prefab`, and `Stimulus Prefab` assigned. The steps below apply to a scene
assembled outside that template.

1. Drag `Assets/UI-lick-reward/UI-Control.prefab` into the scene hierarchy.
2. Confirm the instance's `LickStimulusSpawner` still shows `Canvas` (the `Canvas` component on the
   `Canvas` child GameObject), `Lick Prefab` (`LickMsg.prefab`), and `Stimulus Prefab`
   (`RewardMsg.prefab`) populated. The prefab asset ships all three assigned, so this step only
   catches an instance whose overrides cleared them.
3. Enter Play Mode. Interaction and stimulus events spawn short-lived indicators on the canvas.

### Intra-Unity subscription

`LickStimulusSpawner` subscribes to `Stimulus`, which is also published by `StimulusTriggerZone`
inside the same Unity process. This is an intentional intra-Unity loopback; see `/mqtt-contract`
for the multi-consumer behavior. **You MUST NOT** treat the self-delivery as a bug.

---

## Scene-specific vs project-wide state

| State                                 | Scope                       | Where stored                                                    |
|---------------------------------------|-----------------------------|-----------------------------------------------------------------|
| Camera Mapping (camera ↔ monitor)     | Per-scene                   | `Assets/VRSettings/Displays/<scene>-savedFullScreenViews.asset` |
| Actor model + controller selection    | Per-scene                   | Scene `.unity` file                                             |
| `Task` fields (require, length, seed) | Per-scene                   | Scene `.unity` file                                             |
| `Display` brightness / `heightInVR`   | Per display GameObject name | `Assets/VRSettings/Displays/<display name>.asset`               |
| MQTT broker IP / port                 | Project-wide (per user)     | `EditorPrefs` (`SollertiaVR_MQTT_*`)                            |

**Implication:** every scene must have its Camera Mapping configured independently after a fresh
checkout or a system reboot. **You SHOULD** maintain one scene per experimental protocol so the
configuration can be saved and reused.

---

## Pre-Play Mode checklist

**You MUST** work through this list after any scene edit and before entering Play Mode:

```text
- [ ] Scene contains the auto-created "MQTT Client" / "Actors" / "Controllers" GameObjects
- [ ] Scene contains a GameObject with MQTTConnectorObject (the template's "Logger"); InitializeScene
      does not create one
- [ ] Scene contains exactly one ActorObject under "Actors"
- [ ] Scene contains at least one DisplayObject parented under the Actor
- [ ] Actor.Controller is set to Linear OR Simulated Linear (not None, unless deliberately disabled)
- [ ] The scene has been saved (an untitled scene cannot persist Camera Mapping)
- [ ] On macOS / Linux, the monitor-enumeration helper is installed (displayplacer / xrandr)
- [ ] Camera Mapping is bound for every required monitor (Play Mode opens the views to verify)
- [ ] Task prefab instance is at transform (0, 0, 0)
- [ ] Task.actor is set (the Task section auto-fills from the cached actor when null)
- [ ] Task.configPath points to an existing YAML file
- [ ] If using Simulated Linear, confirm it is intended (not leftover from testing)
- [ ] If UI feedback is desired, UI-Control.prefab is in the scene with prefab fields assigned
- [ ] If the Controller is Linear, or any topic other than Interaction / Stimulus is exercised,
      the MQTT broker is running on the IP / port configured in the MQTT section
```

A `Simulated Linear` keyboard-only run needs **no** broker: when the broker is unreachable,
`MQTTClient.Publish` falls back to routing the message straight to in-process subscribers on the
matching topic and logs one `broker unreachable` warning per topic. Only `Interaction` and `Stimulus`
are both published and subscribed inside Unity, so they alone can be exercised that way.

---

## Common failure modes

| Symptom                                                | Root cause                                                                                                                               | Resolution                                                                                       |
|--------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| Display does not follow the actor in Play Mode         | Actor.Display not assigned, so `ActorObject.Display`'s setter never ran `ParentToActor`                                                  | Close and reopen `Window → Task Parameters` to retrigger `EnsureActorAndDisplay`                 |
| Camera Mapping lists no monitors at all                | No monitors detected — on macOS `displayplacer`, on Linux `xrandr` is missing or failed (see the `Monitor enumeration:` Console warning) | Install the helper, then press **Refresh Monitor Positions** or call `refresh_monitors_tool`     |
| Camera Mapping rows are empty after a scene open       | Monitors enumerated but no camera bound yet, or the scene was created without the `MainWindow.InitializeScene` pass                      | Bind each row; if the Actor / Display are missing, close and reopen `Window → Task Parameters`   |
| Monitors show wrong content after reboot               | OS reassigned monitor ports                                                                                                              | Press **Refresh Monitor Positions** (or call `refresh_monitors_tool`) and reassign cameras       |
| Two stacked full-screen windows per monitor            | **Show Full-Screen Views** was pressed in edit mode, and the Play-Mode pass uses `closeOldViews: false`                                  | Left-click each edit-mode view to close it, then let Play Mode open the views                    |
| Camera bindings vanish after a restart                 | They were made in an untitled scene, so `SaveCameras` no-opped                                                                           | Save the scene, then rebind                                                                      |
| Keyboard input has no effect in Play Mode              | Controller dropdown is `Linear`, not `Simulated Linear`                                                                                  | Swap via the Actor section's Controller dropdown                                                 |
| Spurious `Interaction` events in session log           | Forgotten `Simulated Linear` selection in a production scene                                                                             | Swap back to `Linear`                                                                            |
| UI indicators never appear                             | `LickStimulusSpawner` canvas / prefab fields unset                                                                                       | Assign fields in the Inspector (a `Stimulus` with `delivered == false` correctly spawns nothing) |
| Task disables itself at `Start`, corridor never builds | `Task: configuration YAML not found. configPath='…', resolved='…'` — `configPath` drifted from the YAML                                  | Regenerate via `/task-prefabs` or fix the path                                                   |
| Full-screen views open on wrong monitors               | Monitor indices reordered or new monitors attached                                                                                       | Refresh Monitor Positions, reassign cameras                                                      |
| `Window → Task Parameters` shows "No Task component"   | Active scene contains no task prefab                                                                                                     | `create_task_tool(template_name=...)`, or drag a task prefab in                                  |
| Default `Main Camera` present in a new scene           | `ExperimentTemplate.unity` ships a `Main Camera` and `CreateSceneFromTemplate` does not run the removal pass                             | Close and reopen `Window → Task Parameters`, which logs the removal                              |

---

## Verification checklist

```text
- [ ] The active scene passes the pre-Play Mode checklist above
- [ ] Camera Mapping has been verified by entering Play Mode and checking each full-screen view
- [ ] Controller dropdown selection matches the intended use (hardware vs simulated)
- [ ] UI-lick-reward canvas, if present, has all prefab fields assigned
- [ ] For a hardware run, the MQTT broker is reachable before entering Play Mode (Test Connection
      passes); a Simulated Linear keyboard-only run does not require one
- [ ] No console errors appear during Play Mode startup
- [ ] Auto-created GameObjects (Actors, Controllers, MQTT Client) were not deleted or hidden
```

---

## Related skills

| Skill                             | Relationship                                                                 |
|-----------------------------------|------------------------------------------------------------------------------|
| `/task-scenes` (this plugin)      | Upstream, opens the scene this skill configures                              |
| `/task-prefabs` (this plugin)     | Upstream, creates the scene and the task prefab placed into it               |
| `/task-generator` (this plugin)   | Reference for the `CreateTask` pipeline that builds the prefab placed here   |
| `/task-parameters` (this plugin)  | Programmatic alternative to the GUI flows here; owns `refresh_monitors_tool` |
| `/play-mode` (this plugin)        | Consumer — entered after scene setup passes the pre-Play Mode checklist      |
| `/gimbl-framework` (this plugin)  | Reference for `MainWindow` invariants, `ActorObject`, `DisplayObject`, etc.  |
| `/mqtt-contract` (this plugin)    | Topics consumed by `UI-lick-reward` and published by `Simulated Linear`      |
| `assets:task-templates`           | Upstream — owns the YAML that drove the prefab and backs `Task.configPath`   |
| `assets:experiment-configuration` | Upstream — per-project, per-system experiment configuration author           |
