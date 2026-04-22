---
name: scene-setup
description: >-
  Guides Editor-side scene configuration for sollertia-unity-tasks: three-monitor Display rig
  assignment, SimulatedLinearTreadmill for keyboard testing, and the UI lick-reward feedback
  canvas. Use when preparing a new scene for Play Mode, swapping hardware for the simulated
  treadmill, or fixing missing display / controller errors.
user-invocable: true
---

# Sollertia Unity scene setup

Covers the **Editor-time** configuration that turns a freshly created scene into one that can actually run a task.
Generation, Play Mode, and asset enumeration live in sibling skills; this skill owns the step between "scene exists"
and "scene is runnable."

---

## Scope

**Covers:**
- Display rig configuration via `Window → Gimbl` (Settings, Actor, and Displays panels)
- Three-monitor VR setup (Left / Center / Right) used by the mesoscope acquisition system
- Swapping between `LinearTreadmill` (hardware) and `SimulatedLinearTreadmill` (keyboard) controllers
- Adding the `UI-lick-reward` canvas subsystem for experimenter feedback
- Scene-specific vs project-wide configuration state
- Required scene actors (`ActorObject`, `MQTT Client`, task prefab instance)

**Does not cover:**
- Creating scenes or enumerating assets (see `/scenes`)
- Generating the task prefab that is dropped into a scene (see `/task-prefabs`, `/task-generator`)
- Entering / exiting Play Mode (see `/play-mode`)
- GIMBL class APIs (see `/gimbl-framework`)
- MQTT topic details (see `/mqtt-contract`)

---

## Required scene contents

A runnable scene contains:

| GameObject               | Component                     | Purpose                                         |
|--------------------------|-------------------------------|-------------------------------------------------|
| `MQTT Client`            | `Gimbl.MQTTClient`            | Broker singleton; creates `MQTTClient.Instance` |
| `<ActorName>` (e.g. Mouse) | `Gimbl.ActorObject`         | Animal avatar with display + controller         |
| `<Display rig>`          | `Gimbl.DisplayObject` (x3)    | Left / Center / Right monitor attachments       |
| `<Task prefab instance>` | `SL.Tasks.Task`               | Corridor hierarchy (dropped in from `Tasks/`)   |
| `UI-Control` (optional)  | `SL.UI.LickStimulusSpawner`   | On-screen lick and stimulus indicators          |

`ExperimentTemplate.unity` ships with the first four. The task prefab and UI control are added manually (or by
`create_scene_tool`, which seeds the task prefab).

---

## Display rig configuration

The Sollertia mesoscope rig uses three physical monitors arranged around the animal's viewing position. Each monitor
is driven by a separate `DisplayObject` attached under the `ActorObject`.

### Opening the Displays panel

Open `Window → Gimbl`. The menu opens three editor windows:

- **Settings** — MQTT broker IP/port and session export.
- **Actor** — scene actor creation and controller linking.
- **Displays** — monitor assignment and fullscreen view toggles.

If the Displays tab is not visible, it is likely docked behind the Inspector. Detach the Inspector or drag the
Displays tab out.

### Assigning monitors

1. In the **Displays** tab, click **Refresh Monitor Positions**. Unity enumerates attached monitors.
2. For each monitor entry, assign the matching display camera from the scene:
   - `Camera: LeftMonitor` → left physical monitor
   - `Camera: CenterMonitor` → center physical monitor
   - `Camera: RightMonitor` → right physical monitor
3. Click **Show Full-Screen Views** to verify each camera renders to the intended monitor. Each monitor should show
   its side of the VR corridor.
4. If an assignment is wrong, drag the camera reference to a different monitor slot and re-verify.

### Rebooting the system changes monitor ports

Operating-system reboots can reorder monitor output ports. **Always** reverify the monitor assignments before
starting an experimental session. The Displays panel's monitor indices are not stable across reboots.

### Per-scene state

Display assignments are **scene-specific**: they are persisted inside each scene's `.unity` file, not in project
settings. Every new scene (including scenes created via `create_scene_tool`) must be configured once.

MQTT broker settings in the Settings tab are the opposite — they are stored in `EditorPrefs` and shared across all
scenes in the project.

---

## Choosing a controller

The `ActorObject.Controller` field accepts any `ControllerOutput`. In practice the choice is binary:

| Controller                  | When to use                                              | Input source           |
|-----------------------------|----------------------------------------------------------|------------------------|
| `LinearTreadmill`           | Running against real mesoscope hardware                  | MQTT `<deviceName>/Data/` |
| `SimulatedLinearTreadmill`  | Manual Editor testing without hardware                   | Keyboard + mouse       |

`ExperimentTemplate.unity` ships with `LinearTreadmill` pre-configured because it is the production default. Swap to
`SimulatedLinearTreadmill` when exercising a task during development.

### Installing `SimulatedLinearTreadmill`

1. Remove the existing `LinearTreadmill` component from the actor's controller GameObject (or create a new child).
2. Add `SimulatedLinearTreadmill` to the same GameObject.
3. Set `Settings → Active` to true in the `Window → Gimbl → Actor` panel.
4. Enter Play Mode (`/play-mode`).
5. Controls:
   - **W / Up arrow** — move forward
   - **S / Down arrow** — move backward
   - **Mouse left click** — simulate a single lick (publishes `LickPort/`)

Movement speed is scaled by `MovementSpeedMultiplier = 8.0f` inside `SimulatedLinearTreadmill.cs`.

### Reverting to hardware

Before running a real session, swap `SimulatedLinearTreadmill` back out for `LinearTreadmill`, confirm the MQTT
`deviceName` matches the hardware publisher's topic prefix, and verify `Settings → isActive` is true. Leaving a
simulated controller in a production scene publishes spurious `LickPort/` events on every mouse click, corrupting
the session log.

---

## UI-lick-reward subsystem

The `UI-lick-reward` folder provides an on-screen feedback canvas for experimenters running interactive sessions. It
is **not** part of the task's behavioral contract — removing it changes nothing about the runtime data.

### Components

| Asset                              | Purpose                                                          |
|------------------------------------|------------------------------------------------------------------|
| `UI-Control.prefab`                | Canvas prefab carrying `LickStimulusSpawner`                     |
| `LickMsg.prefab`                   | Instantiated when `LickPort/` is received                        |
| `RewardMsg.prefab`                 | Instantiated when `Gimbl/Stimulus/` is received                  |
| `lickAnimation.anim`               | Short animation played by `LickMessage` on spawn                 |
| `rewardAnimation.anim`             | Short animation played by `StimulusMessage` on spawn             |

### Scripts

- `LickStimulusSpawner.cs` — the root MonoBehaviour on `UI-Control`. Subscribes to `LickPort/` and `Gimbl/Stimulus/`
  and instantiates the corresponding indicator prefab on the canvas.
- `LickMessage.cs` — attached to `LickMsg`. Drives the lick indicator animation and self-destructs.
- `StimulusMessage.cs` — attached to `RewardMsg`. Drives the stimulus indicator animation and self-destructs.

### Installing in a scene

1. Drag `Assets/UI-lick-reward/UI-Control.prefab` into the scene hierarchy.
2. In the `LickStimulusSpawner` inspector, assign:
   - `Canvas` — the `Canvas` component on `UI-Control` itself.
   - `Lick Prefab` — `LickMsg.prefab`.
   - `Stimulus Prefab` — `RewardMsg.prefab`.
3. Enter Play Mode. Licks and stimulus events spawn short-lived indicators on the canvas.

### Intra-Unity subscription

`LickStimulusSpawner` subscribes to `Gimbl/Stimulus/`, which is also published by `StimulusTriggerZone` inside the
same Unity process. This is an intentional intra-Unity loopback; see `/mqtt-contract` for the multi-consumer
behavior. Do not treat the self-delivery as a bug.

---

## Scene-specific vs project-wide state

| State                                | Scope                | Where stored                      |
|--------------------------------------|----------------------|-----------------------------------|
| Display monitor assignments          | Per-scene            | Scene `.unity` file               |
| Fullscreen view toggle               | Per-scene            | Scene `.unity` file               |
| `ActorObject` settings reference     | Per-scene            | Scene `.unity` file               |
| MQTT broker IP / port                | Project-wide (per user) | `EditorPrefs`                   |
| Controller choice (hardware vs sim)  | Per-scene            | Scene `.unity` file               |
| GIMBL output path / session name     | Per-user             | `EditorPrefs`                     |

**Implication:** every scene must have its displays configured independently after a fresh checkout or a system
reboot. Maintain one scene per experimental protocol so the configuration can be saved and reused.

---

## Pre-Play Mode checklist

Run through this list after any scene edit and before entering Play Mode:

```text
- [ ] Scene contains a "MQTT Client" GameObject with Gimbl.MQTTClient attached
- [ ] Scene contains exactly one ActorObject
- [ ] Actor.Display is set (green indicator in the Inspector)
- [ ] Actor.Controller is set to LinearTreadmill OR SimulatedLinearTreadmill (not both)
- [ ] Three DisplayObjects are assigned in Window → Gimbl → Displays
- [ ] Task prefab instance is at transform (0, 0, 0)
- [ ] Task script's Actor field is set (not "None")
- [ ] Task script's Config Path points to an existing YAML file
- [ ] If using SimulatedLinearTreadmill, confirm it is intended (not leftover from testing)
- [ ] If UI feedback is desired, UI-Control.prefab is in the scene with prefab fields assigned
- [ ] MQTT broker is running on the IP/port configured in Window → Gimbl → Settings
```

---

## Common failure modes

| Symptom                                                | Root cause                                                 | Resolution                                    |
|--------------------------------------------------------|------------------------------------------------------------|-----------------------------------------------|
| `NullReferenceException` on Play — `Display` is null   | Actor.Display not assigned                                 | Set via Inspector or `Window → Gimbl → Actor` |
| Monitors show wrong content after reboot                | OS reassigned monitor ports                                | Press Refresh Monitor Positions, reassign     |
| Keyboard input has no effect in Play Mode              | Controller is `LinearTreadmill`, not `SimulatedLinearTreadmill` | Swap controller                           |
| Spurious lick events in session log                     | Forgotten `SimulatedLinearTreadmill` in production scene  | Swap back to `LinearTreadmill`                |
| UI indicators never appear                              | `LickStimulusSpawner` canvas / prefab fields unset         | Assign fields in the Inspector                |
| Task script errors "No configuration YAML file found"   | Task.configPath drifted from actual YAML location          | Fix the path or regenerate the task prefab    |
| Fullscreen views open on the wrong monitors             | Monitor indices reordered or new monitors attached         | Refresh Monitor Positions, reassign cameras   |

---

## Verification checklist

```text
- [ ] The scene passes the pre-Play Mode checklist above
- [ ] Display assignments have been verified with Show Full-Screen Views
- [ ] Controller choice matches the intended use (hardware vs simulated)
- [ ] UI-lick-reward canvas, if present, has all prefab fields assigned
- [ ] MQTT broker is reachable before entering Play Mode
- [ ] No console errors appear during Play Mode startup
```

---

## Related skills

| Skill                            | Relationship                                                          |
|----------------------------------|-----------------------------------------------------------------------|
| `/scenes` (this plugin)          | Upstream — creates or opens the scene this skill configures           |
| `/task-prefabs` (this plugin)    | Upstream — generates the prefab placed into the scene                 |
| `/play-mode` (this plugin)       | Consumer — entered after scene setup passes the pre-Play Mode checklist |
| `/gimbl-framework` (this plugin) | Reference for `ActorObject`, `DisplayObject`, controller classes      |
| `/mqtt-contract` (this plugin)   | Topics consumed by `UI-lick-reward` and published by `SimulatedLinearTreadmill` |
