---
name: scene-setup
description: >-
  Guides Editor-side scene configuration for sollertia-unity-tasks: the consolidated Task
  Parameters window, the three-monitor Display rig, swapping the LinearTreadmill and
  SimulatedLinearTreadmill controllers from the Actor section, and the optional UI lick-reward
  feedback canvas. Use when preparing a new scene for Play Mode, swapping hardware for the
  simulated treadmill, or fixing missing display / controller errors.
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
- Creating scenes or enumerating assets (see `/task-scenes`)
- Generating the task prefab dropped into a scene (see `/task-prefabs`, `/task-generator`)
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

If the user closes it manually, opening any of the above events brings it back. The docked tab
label is `Parameters`; the menu entry is `Window → Task Parameters` to disambiguate.

### Auto-created scene infrastructure

Opening the window (or opening any scene with the window already open) ensures the active scene
contains the following before the GUI renders. Existing objects are left untouched; missing ones
are created.

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
| `Actor` (or renamed)          | `Gimbl.ActorObject`                                                 | Animal avatar with display + controller                 |
| `<Display>`                   | `Gimbl.DisplayObject`                                               | Multi-monitor rig with per-monitor cameras              |
| `Linear` / `Simulated Linear` | `LinearTreadmill` / `SimulatedLinearTreadmill` + `ControllerOutput` | Input devices; pick one via Actor dropdown              |
| `<Task prefab instance>`      | `SL.Tasks.Task`                                                     | Corridor hierarchy (dropped in from `Tasks/`)           |
| `UI-Control` (optional)       | `SL.UI.LickStimulusSpawner`                                         | On-screen lick and stimulus indicators                  |

`ExperimentTemplate.unity` ships without the task prefab and UI control; both are added manually
(or `create_task_tool` from `/task-scenes` seeds the task prefab and `CreateTask.CreateSceneFromTemplate`
guarantees both controllers exist).

---

## Display rig configuration

This project defines a three-monitor VR setup arranged around the animal's viewing position;
downstream acquisition rigs adopt this layout. The Display section of the Parameters window
controls brightness and VR height; the **Camera Mapping** section binds the three per-monitor
cameras (`Left View`, `Center View`, `Right View`) to OS monitor indices.

**Scope boundary.** This skill configures the project's standard three-monitor corridor rig. Authoring a *different*
Display rig or a non-corridor scene topology has no author-derived recipe — escalate to the human supervisor and
co-design it in a generative, collaborative mode. You MUST NOT hand-author a new rig or scene topology autonomously.

### Assigning monitors

1. Open `Window → Task Parameters` and scroll to **Camera Mapping**.
2. Click **Refresh Monitor Positions** if the entries do not match the OS-reported monitors.
3. For each row, pick the matching camera from the dropdown (the Display rig auto-names cameras
   after the role, e.g., `Left View`, `Center View`, `Right View`).
4. Click **Show Full-Screen Views** in edit mode to spawn a borderless window on each assigned
   monitor. The windows are empty until the scene starts playing; the button is also disabled
   while playing, so the sequence is: click in edit mode → enter Play Mode (`/play-mode`) →
   verify each monitor shows its side of the VR corridor → exit Play Mode to close the windows.
5. If an assignment is wrong, swap entries and re-verify.

### Reboot caveat

Operating-system reboots can reorder monitor output ports. **You MUST** re-verify the
camera-to-monitor binding before starting an experimental session. The Camera Mapping section's
monitor indices are not stable across reboots.

### Display section

The Display section exposes **two editable fields** (`brightness`, `heightInVR`) plus a Blank /
Show button. The relevant state values are:

| Value               | Surface                                            | Persistence                                                                   | Effect                                        |
|---------------------|----------------------------------------------------|-------------------------------------------------------------------------------|-----------------------------------------------|
| `brightness`        | Numeric field (default `50`)                       | `Assets/VRSettings/Displays/<Display>.asset`                                  | Default brightness restored by "Show Display" |
| `heightInVR`        | Numeric field (default `0.2`)                      | `Assets/VRSettings/Displays/<Display>.asset`                                  | Y offset of the display rig from the actor    |
| `currentBrightness` | No field — written only by the Blank / Show button | Scene-serialized on `DisplayObject`; synced to `brightness` on scene creation | Live brightness override applied to rendering |

The Blank / Show button flips `currentBrightness` between `0` and the configured `brightness`.
`CreateTask.CreateSceneFromTemplate` calls `MainWindow.SyncDisplayBrightnessToSettings` after the
scene is instantiated, so a freshly created scene's `currentBrightness` matches the asset's
`brightness` rather than the `DisplayObject` field initializer. `DisplayObject.Create` also reuses
an existing `<displayName>.asset` at `Assets/VRSettings/Displays/` instead of overwriting it, so
user-customized `brightness` / `heightInVR` survive subsequent scene rebuilds.

### Per-scene state

Camera Mapping assignments are **scene-specific** and persisted in
`Assets/VRSettings/Displays/<scene-name>-savedFullScreenViews.asset`. Every new scene (including
scenes created via `create_task_tool`) must have its cameras bound once.

`brightness` and `heightInVR` are stored on the **DisplaySettings asset** — they are shared
across scenes that use the same Display prefab. The MQTT broker `ip` and `port` are stored in
`EditorPrefs` (`SollertiaVR_MQTT_IP` / `SollertiaVR_MQTT_Port`) and apply project-wide.

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
3. Controls (Unity Input System action map `SimulatedInput`):
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
interactive sessions. It is **not** part of the task's behavioral contract — removing it changes
nothing about the runtime data.

### Components

| Asset                              | Purpose                                                          |
|------------------------------------|------------------------------------------------------------------|
| `UI-Control.prefab`                | Canvas prefab carrying `LickStimulusSpawner`                     |
| `LickMsg.prefab`                   | Instantiated when `Interaction` is received                      |
| `RewardMsg.prefab`                 | Instantiated when `Stimulus` is received                         |
| `lickAnimation.anim`               | Short animation played by `LickMessage` on spawn                 |
| `rewardAnimation.anim`             | Short animation played by `StimulusMessage` on spawn             |

### Scripts

- `LickStimulusSpawner.cs` — the root MonoBehaviour on `UI-Control`. Subscribes to `Interaction` and
  `Stimulus` and instantiates the corresponding indicator prefab on the canvas.
- `LickMessage.cs` — attached to `LickMsg`. Drives the lick indicator animation and self-destructs.
- `StimulusMessage.cs` — attached to `RewardMsg`. Drives the stimulus indicator animation and
  self-destructs.

### Installing in a scene

1. Drag `Assets/UI-lick-reward/UI-Control.prefab` into the scene hierarchy.
2. In the `LickStimulusSpawner` inspector, assign:
   - `Canvas` — the `Canvas` component on `UI-Control` itself.
   - `Lick Prefab` — `LickMsg.prefab`.
   - `Stimulus Prefab` — `RewardMsg.prefab`.
3. Enter Play Mode. Licks and stimulus events spawn short-lived indicators on the canvas.

### Intra-Unity subscription

`LickStimulusSpawner` subscribes to `Stimulus`, which is also published by `StimulusTriggerZone`
inside the same Unity process. This is an intentional intra-Unity loopback; see `/mqtt-contract`
for the multi-consumer behavior. **You MUST NOT** treat the self-delivery as a bug.

---

## Scene-specific vs project-wide state

| State                                 | Scope                   | Where stored                                                    |
|---------------------------------------|-------------------------|-----------------------------------------------------------------|
| Camera Mapping (camera ↔ monitor)     | Per-scene               | `Assets/VRSettings/Displays/<scene>-savedFullScreenViews.asset` |
| Actor model + controller selection    | Per-scene               | Scene `.unity` file                                             |
| `Task` fields (require, length, seed) | Per-scene               | Scene `.unity` file                                             |
| `Display` brightness / `heightInVR`   | Per Display prefab      | `Assets/VRSettings/Displays/<display>.asset`                    |
| MQTT broker IP / port                 | Project-wide (per user) | `EditorPrefs` (`SollertiaVR_MQTT_*`)                            |

**Implication:** every scene must have its Camera Mapping configured independently after a fresh
checkout or a system reboot. **You SHOULD** maintain one scene per experimental protocol so the
configuration can be saved and reused.

---

## Pre-Play Mode checklist

**You MUST** work through this list after any scene edit and before entering Play Mode:

```text
- [ ] Scene contains the auto-created "MQTT Client" / "Actors" / "Controllers" GameObjects
- [ ] Scene contains exactly one ActorObject under "Actors"
- [ ] Scene contains at least one DisplayObject parented under the Actor
- [ ] Actor.Controller is set to Linear OR Simulated Linear (not None, unless deliberately disabled)
- [ ] Camera Mapping is bound for every required monitor (use Show Full-Screen Views to verify)
- [ ] Task prefab instance is at transform (0, 0, 0)
- [ ] Task.actor is set (the Task section auto-fills from the cached actor when null)
- [ ] Task.configPath points to an existing YAML file
- [ ] If using Simulated Linear, confirm it is intended (not leftover from testing)
- [ ] If UI feedback is desired, UI-Control.prefab is in the scene with prefab fields assigned
- [ ] MQTT broker is running on the IP / port configured in the MQTT section
```

---

## Common failure modes

| Symptom                                              | Root cause                                                                           | Resolution                                                              |
|------------------------------------------------------|--------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| `NullReferenceException` on Play — `Display` is null | Actor.Display not assigned                                                           | Re-open `Window → Task Parameters` to retrigger `EnsureActorAndDisplay` |
| Camera Mapping rows are empty after a scene open     | Scene was created without the `MainWindow.InitializeScene` pass                      | Open `Window → Task Parameters`; `OnEnable` reruns `InitializeScene`    |
| Monitors show wrong content after reboot             | OS reassigned monitor ports                                                          | Press **Refresh Monitor Positions** and reassign cameras                |
| Keyboard input has no effect in Play Mode            | Controller dropdown is `Linear`, not `Simulated Linear`                              | Swap via the Actor section's Controller dropdown                        |
| Spurious lick events in session log                  | Forgotten `Simulated Linear` selection in a production scene                         | Swap back to `Linear`                                                   |
| UI indicators never appear                           | `LickStimulusSpawner` canvas / prefab fields unset                                   | Assign fields in the Inspector                                          |
| Task script errors "Configuration YAML not found"    | `Task.configPath` drifted from actual YAML location                                  | Regenerate via `/task-prefabs` or fix the path                          |
| Full-screen views open on wrong monitors             | Monitor indices reordered or new monitors attached                                   | Refresh Monitor Positions, reassign cameras                             |
| `Window → Task Parameters` shows "No Task component" | Active scene contains no task prefab                                                 | `create_task_tool` with a `task_prefab_path`, or drag a task prefab in  |
| Default `Main Camera` re-appears after scene open    | Editor reopened a scene saved before the cleanup; the next Parameters open clears it | Open / re-focus `Window → Task Parameters`; it logs the removal         |

---

## Verification checklist

```text
- [ ] The active scene passes the pre-Play Mode checklist above
- [ ] Camera Mapping has been verified with Show Full-Screen Views
- [ ] Controller dropdown selection matches the intended use (hardware vs simulated)
- [ ] UI-lick-reward canvas, if present, has all prefab fields assigned
- [ ] MQTT broker is reachable before entering Play Mode (Test Connection passes)
- [ ] No console errors appear during Play Mode startup
- [ ] Auto-created GameObjects (Actors, Controllers, MQTT Client) were not deleted or hidden
```

---

## Related skills

| Skill                                     | Relationship                                                                    |
|-------------------------------------------|---------------------------------------------------------------------------------|
| `/task-scenes` (this plugin)              | Upstream — creates or opens the scene this skill configures                     |
| `/task-prefabs` (this plugin)             | Upstream — generates the prefab placed into the scene                           |
| `/task-parameters` (this plugin)          | Programmatic alternative to the GUI flows described here                        |
| `/play-mode` (this plugin)                | Consumer — entered after scene setup passes the pre-Play Mode checklist         |
| `/gimbl-framework` (this plugin)          | Reference for `MainWindow` invariants, `ActorObject`, `DisplayObject`, etc.     |
| `/mqtt-contract` (this plugin)            | Topics consumed by `UI-lick-reward` and published by `Simulated Linear`         |
| `assets:task-templates`           | Upstream — owns the YAML that drove the prefab via `/task-prefabs`              |
| `assets:experiment-configuration` | Upstream — per-project instantiation of the template (drives `Task.configPath`) |
