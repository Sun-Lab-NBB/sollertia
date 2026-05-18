---
name: task-parameters
description: >-
  Reads and writes the consolidated Task Parameters editor window in sollertia-unity-tasks via the
  sollertia-shared-assets MCP server's Unity relay. Owns read_task_parameters_tool and
  write_task_parameters_tool, which mirror the Actor, MQTT, Display, Camera Mapping, and Task
  sections of `Window → Task Parameters`. Use when inspecting or programmatically changing per-scene
  Task / Actor / Display / MQTT / Camera Mapping settings without opening the Editor window manually.
user-invocable: true
---

# Sollertia Unity Task Parameters

Programmatically reads and writes the consolidated **Task Parameters** Unity Editor window for
`sollertia-unity-tasks` using the Unity relay exposed by `slsa mcp`. This skill is the
**exclusive** owner of `read_task_parameters_tool` and `write_task_parameters_tool` — no other
skill in the marketplace may call these.

The window itself is owned by `MainWindow`
(`Assets/Gimbl/Editor/MainWindow.cs`); the read / write surface is a 1:1 mirror of the GUI controls
plus the option lists and visibility flags the GUI uses to render them. The `Task` component's
public fields are `[HideInInspector]` and `TaskEditor` replaces the default Inspector with a
HelpBox pointing at the Parameters window — so this skill is the *only* programmatic entry point
for those fields.

---

## Scope

**Covers:**
- Reading the active scene's Actor, MQTT, Display, Camera Mapping, and Task fields in one snapshot
  (`read_task_parameters_tool`)
- Writing any subset of those fields atomically and receiving the post-write snapshot
  (`write_task_parameters_tool`)
- The option lists (allowed enum values) and visibility flags returned alongside the state
- Validation rules that mirror the GUI (zone-gated `require_lick` / `require_wait`, monitor index
  bounds, controller / model / camera membership)
- Choosing between Parameters window writes and scene-file edits

**Does not cover:**
- Generating or validating prefabs (see `/task-prefabs`)
- Listing, opening, creating, or inspecting scenes (see `/task-scenes`)
- Entering / exiting Play Mode (see `/play-mode`)
- Editor-time scene wiring beyond what the Parameters window exposes (see `/scene-setup`)
- MQTT topic catalog (see `/mqtt-contract`)
- The `MainWindow` GUI internals or controller / actor auto-creation behavior (see
  `/scene-setup` and `/gimbl-framework`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

---

## MCP tool surface

| Tool                          | Purpose                                                                            |
|-------------------------------|------------------------------------------------------------------------------------|
| `read_task_parameters_tool`   | Snapshot the active scene's Parameters state, options, and visibility (exclusive)  |
| `write_task_parameters_tool`  | Apply a subset of fields and return the post-write snapshot (exclusive)            |

Both tools require the Unity Editor to be running with the `McpBridge` plugin active (else
`/unity-mcp-environment-setup`). Both share a single `AcquireSceneComponents` walk per request, so
`read → write → read` is consistent and writes never race a separate enumeration pass.

---

## Response shape

`read_task_parameters_tool` and the post-write payload from `write_task_parameters_tool` return the
same dict with three top-level keys: **`state`**, **`options`**, **`visibility`**. Every section
key is a stable string; the values vary by section.

### `state`

Current values for the five sections. Each section is `None` when the corresponding scene
component is absent (no `ActorObject`, no `DisplayObject`, etc.).

```json
{
  "actor":          {"model": "Mouse",     "controller": "Simulated Linear"},
  "mqtt":           {"ip": "127.0.0.1",    "port": 1883},
  "display":        {"current_brightness": 100, "brightness": 100, "height_in_vr": 0.0},
  "camera_mapping": [
    {"monitor": 1, "left": 0,    "top": 0, "camera": "Left View"},
    {"monitor": 2, "left": 1920, "top": 0, "camera": "Center View"},
    {"monitor": 3, "left": 3840, "top": 0, "camera": "Right View"}
  ],
  "task":           {"require_lick": true, "require_wait": false, "track_length": 15000.0, "track_seed": -1}
}
```

Section semantics:

| Section          | Source script                                                | Persistence                                                     |
|------------------|--------------------------------------------------------------|-----------------------------------------------------------------|
| `actor`          | `Gimbl.ActorObject`                                          | Scene file                                                      |
| `mqtt`           | `Gimbl.MQTTClient`                                           | `EditorPrefs` (`SollertiaVR_MQTT_IP` / `_Port`)                 |
| `display`        | `Gimbl.DisplayObject` + `DisplaySettings` asset              | Scene file + `Assets/VRSettings/Displays/<Display>.asset`       |
| `camera_mapping` | `Gimbl.FullScreenViewManager` + `FullScreenViewsSaved` asset | `Assets/VRSettings/Displays/<scene>-savedFullScreenViews.asset` |
| `task`           | `SL.Tasks.Task`                                              | Scene file                                                      |

Notes:
- `actor.model` is derived from the actor's child GameObject whose name starts with `Model `
  (`"None"` when absent).
- `actor.controller` is the assigned `ControllerOutput`'s GameObject name (`"None"` when null).
- `display.current_brightness` is the live runtime brightness (the "blank display" toggle in the
  GUI flips this to 0 or back to `brightness`).
- `display.brightness` is the configured default that the "Show Display" button restores to.
- `camera_mapping` is a list whose length equals the number of monitors the OS reports for the
  current scene; `monitor` is 1-based to match the GUI labels.
- `task.track_seed == -1` is the documented sentinel for "nondeterministic seed".

### `options`

Enumerated alternatives for fields with a finite valid set. Use these as the authoritative
allow-list when constructing a `write_task_parameters_tool` payload — values outside these lists
are rejected by the bridge with a descriptive error.

```json
{
  "actor": {
    "model":      ["Mouse", "None"],
    "controller": ["None", "Linear", "Simulated Linear"]
  },
  "camera_mapping": {
    "camera":     ["None", "Left View", "Center View", "Right View"]
  }
}
```

- `actor.model` is every prefab under `Resources/Actors/Prefabs/` plus the literal `"None"`.
- `actor.controller` is the GameObject name of every `ControllerOutput` in the active scene plus
  `"None"`. `MainWindow.EnsureControllers` populates this list with `Linear` and `Simulated Linear`
  on scene init (see `/scene-setup`).
- `camera_mapping.camera` is every scene `Camera` that is **not** tagged `MainCamera` and **not**
  named `Main Camera`, plus `"None"`. This matches the filter the GUI dropdown applies so the agent
  and the user see the same option set.

No options list is returned for `mqtt`, `display`, or `task` because their fields are free-form
numeric or boolean values (not enumerations).

### `visibility`

Per-control flags that report whether the matching GUI control is currently rendered. Today only
the `task` section uses this:

```json
{
  "task": {
    "require_lick": true,
    "require_wait": false
  }
}
```

The GUI hides `Require Lick` when no `GuidanceZone` exists in the scene and hides `Require Wait`
when no `OccupancyZone` exists. The bridge mirrors that by **rejecting** writes to those fields
when the matching zone is absent, so `visibility == false` means "scene does not support this
toggle — do not attempt to write it." See [Validation rules](#validation-rules) for the exact
contract.

---

## Workflows

### Read the current state

```text
read_task_parameters_tool()
```

Returns `{"state": ..., "options": ..., "visibility": ..., "success": true}`. Use this snapshot
as the basis for any downstream decision (display rig confirmation, controller swap, MQTT broker
verification, Task field audit). The snapshot is a single scene walk; do not poll it in a tight
loop expecting different results between consecutive frames.

### Update a subset of fields

`write_task_parameters_tool` accepts five optional top-level arguments (`actor`, `mqtt`,
`display`, `camera_mapping`, `task`). Pass only the sections you intend to change; fields inside a
section are individually optional too. The response is the post-write snapshot in the same shape
as `read_task_parameters_tool`, so a `read → modify → write` loop never needs a second read.

```text
write_task_parameters_tool(
    actor={"controller": "Simulated Linear"},
    task={"track_seed": 42, "track_length": 8000.0},
)
```

The bridge marks the active scene dirty when any write succeeds and runs `EditorUtility.SetDirty`
on any modified `DisplaySettings` or `FullScreenViewsSaved` asset so a subsequent
`Ctrl+S` / `EditorSceneManager.SaveOpenScenes()` persists every change.

### Verify a write took effect

The response includes the post-write snapshot, so the simplest verification is to inspect the
returned dict directly. If you need to re-confirm against a fresh scene walk (for example after
manual Editor edits that may have raced your write), call `read_task_parameters_tool()` after the
write returns.

### Swap controllers (Linear ↔ Simulated Linear)

The auto-created scene already contains both controllers under the `Controllers` GameObject (see
`/scene-setup`). To switch the actor between them:

```text
# Inspect available controllers first so the write payload is well-formed.
state = read_task_parameters_tool()
allowed = state["options"]["actor"]["controller"]   # ["None", "Linear", "Simulated Linear"]

write_task_parameters_tool(actor={"controller": "Simulated Linear"})
```

Never type a controller name from memory — always pull the list from `options.actor.controller`
because GameObject renames or future controller additions can change the canonical names.

### Update the MQTT broker

`mqtt.ip` and `mqtt.port` writes propagate to the live `MQTTClient` and to `EditorPrefs`
(`SollertiaVR_MQTT_IP`, `SollertiaVR_MQTT_Port`) so a fresh Editor session inherits the value.
This means changes are **project-wide**, not per-scene; coordinate the change before running
multiple scenes in the same session.

### Toggle guidance modes for a runtime experiment

The Task fields (`require_lick`, `require_wait`) are also addressable at runtime via MQTT
(`RequireLick` / `RequireWait` topics carrying `BoolMessage`). Editor-time writes through this
tool persist the value in the scene; MQTT writes change the live value during Play Mode without
modifying the scene. Use this tool to set the **default** before entering Play Mode and the MQTT
path to flip it mid-experiment.

---

## Validation rules

The bridge rejects writes that the GUI would also refuse. Each rejection returns an
`{"success": false, "error": "..."}` response with a descriptive message.

| Section          | Field                          | Rejection condition                                                                             |
|------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| `actor`          | `model`                        | Value is not in `options.actor.model`                                                           |
| `actor`          | `controller`                   | Value is not in `options.actor.controller`                                                      |
| `camera_mapping` | `monitor`                      | 1-based index outside `[1, monitors.Count]`                                                     |
| `camera_mapping` | `camera`                       | Value is not in `options.camera_mapping.camera`                                                 |
| `task`           | `require_lick`                 | Scene has no `GuidanceZone` (i.e., `visibility.task.require_lick == false`)                     |
| `task`           | `require_wait`                 | Scene has no `OccupancyZone` (i.e., `visibility.task.require_wait == false`)                    |

Other fields (`mqtt.ip`, `mqtt.port`, `display.*`, `task.track_length`, `task.track_seed`) accept
any numeric / string value the underlying `Convert.ToSingle` / `Convert.ToInt32` / `Convert.ToBoolean`
can parse; the bridge does not impose a range check here. The GUI itself does not either, so an
out-of-range brightness or a negative `track_length` will write but produce runtime warnings.

The zone-gated rejection of `require_lick` and `require_wait` is **intentional**: a successful
write guarantees the flag will actually take effect at runtime. Do not paper over a rejection by
writing the underlying `Task` field through a different path — the missing zone means the toggle
has nothing to gate.

---

## Path conventions

This skill does not take filesystem paths directly; every write targets the active scene the Unity
Editor currently has open. Use `/task-scenes` to switch scenes before issuing Parameters writes:

```text
open_scene_tool(scene_path="Assets/Scenes/<name>.unity")   # /task-scenes
write_task_parameters_tool(task={"track_seed": 42})         # this skill
```

The bridge resolves every Parameters target relative to the active scene's components, so two
writes against different scenes require an `open_scene_tool` call between them.

---

## Interaction with Play Mode

The GUI disables several controls while the Editor is in Play Mode:

| Section          | Behavior in Play Mode                                                                                             |
|------------------|-------------------------------------------------------------------------------------------------------------------|
| `actor`          | Always editable (the GUI does not gray these out)                                                                 |
| `mqtt`           | Greyed out — `EditorPrefs` writes still go through, but mutating the live `MQTTClient` mid-session is unsupported |
| `display`        | Editable; `current_brightness` changes are reflected immediately on the live display                              |
| `camera_mapping` | The "Show Full-Screen Views" button is disabled; field writes still go through                                    |
| `task`           | Greyed out — flip via MQTT (`RequireLick` / `RequireWait`) for mid-run changes instead                            |

The bridge does NOT enforce these gates programmatically — writes still succeed during Play Mode.
Treat them as a soft contract: prefer the MQTT path for `task.require_lick` / `task.require_wait`
once a run is in progress, and avoid `mqtt.*` writes mid-session.

Use `get_play_state_tool` (`/play-mode`) to check `state == "edit"` before issuing
`write_task_parameters_tool` calls that the GUI would refuse to apply.

---

## Troubleshooting

| Symptom                                                                         | Cause                                                                                        | Resolution                                                                                                                                     |
|---------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| `state.actor == null` even though an Actor exists                               | The Actor was placed outside the root and `FindAnyObjectByType<ActorObject>()` missed it     | Confirm the Actor is in the active scene; `inspect_scene_tool` (`/task-scenes`) to verify                                                      |
| `state.task == null`                                                            | No `Task` component in the active scene (only the empty `ExperimentTemplate` template scene) | `create_task_tool(scene_name=..., task_prefab_path=...)` (`/task-scenes`) to seed a task                                                       |
| Write rejected: "Invalid controller '...'"                                      | The controller name is not in `options.actor.controller`                                     | Re-read `options.actor.controller`; copy the exact string (it is the GameObject name)                                                          |
| Write rejected: "Cannot set require_lick: the active scene has no GuidanceZone" | The current task prefab has no lick-mode segments                                            | The flag is not applicable — leave it alone, or open a scene whose template includes lick-mode trials                                          |
| Write rejected: "Invalid monitor index N; scene has M monitors"                 | Camera mapping payload references a 1-based monitor index outside `[1, M]`                   | Re-read `state.camera_mapping` to enumerate valid `monitor` indices                                                                            |
| `state.camera_mapping == []`                                                    | OS reported zero monitors at scene load                                                      | Click "Refresh Monitor Positions" in the GUI or replug displays, then retry                                                                    |
| Writes succeed but the GUI shows old values                                     | The Parameters window cached the value before the write landed                               | The bridge already shares the GUI's `FullScreenViewManager` for camera mapping; for other sections close and reopen `Window → Task Parameters` |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable (else /unity-mcp-environment-setup)
- [ ] read_task_parameters_tool was called before every write to capture the current options list
- [ ] Write payloads only contain field values present in options.<section>.<field>
- [ ] require_lick / require_wait writes are gated on visibility.task.<field> == true
- [ ] camera_mapping entries use 1-based monitor indices matching state.camera_mapping[*].monitor
- [ ] Post-write snapshot was inspected to confirm the new state matches the requested change
- [ ] MQTT writes were avoided while the Editor was in Play Mode (use /play-mode to confirm state)
- [ ] Did not bypass the validation rules by editing the scene file directly to set rejected fields
```

---

## Related skills

| Skill                                         | Relationship                                                                                  |
|-----------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                                                      |
| `/task-scenes` (this plugin)                  | Upstream — switches the active scene that this skill reads and writes                         |
| `/scene-setup` (this plugin)                  | Upstream — owns the `MainWindow` GUI and the auto-creation of Actors / Controllers / Displays |
| `/play-mode` (this plugin)                    | Upstream — `get_play_state_tool` gates writes that the GUI greys out at runtime               |
| `/task-prefabs` (this plugin)                 | Upstream — generates the task prefab whose `Task` component this skill mutates                |
| `/mqtt-contract` (this plugin)                | Reference for the `RequireLick` / `RequireWait` runtime alternative to `task` writes          |
| `/gimbl-framework` (this plugin)              | Reference for `ActorObject`, `DisplayObject`, `MQTTClient`, and `ControllerOutput` semantics  |
| assets plugin `/assets-mcp-environment-setup` | Upstream — owns the slsa MCP server diagnostic                                                |
