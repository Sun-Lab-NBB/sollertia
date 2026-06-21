---
name: task-parameters
description: >-
  Reads and writes the consolidated Task Parameters editor window in sollertia-virtual-reality via the
  sollertia-shared-assets MCP server's Unity relay. Owns read_task_parameters_tool and
  write_task_parameters_tool, which mirror the Actor, MQTT, Display, Camera Mapping, and Task
  sections of `Window → Task Parameters`. Use when inspecting or programmatically changing per-scene
  Task / Actor / Display / MQTT / Camera Mapping settings without opening the Editor window manually.
user-invocable: false
---

# Sollertia Unity task parameters

Programmatically reads and writes the consolidated **Task Parameters** Unity Editor window for
`sollertia-virtual-reality` through the Unity relay exposed by `slsa mcp` — the **exclusive** owner of
`read_task_parameters_tool` and `write_task_parameters_tool`, which no other skill in the
marketplace may call.

The window itself is owned by `MainWindow`
(`Assets/Gimbl/Editor/MainWindow.cs`); the read / write surface mirrors the GUI's *field controls*
plus the option lists and visibility flags the GUI uses to render them. Action-only GUI controls
that are not exposed through the bridge include: MQTT `Test Connection`, Camera Mapping
`Refresh Monitor Positions` and `Show Full-Screen Views`, and the Display `Blank Display` /
`Show Display` toggle button (its underlying effect is reachable through a `display.current_brightness`
write). The `Task` component's public fields are `[HideInInspector]` and `TaskEditor` replaces the
default Inspector with a HelpBox pointing at the Parameters window — so this skill is the *only*
programmatic entry point for those fields.

---

## Scope

**Covers:**
- Reading the active scene's Actor, MQTT, Display, Camera Mapping, and Task fields in one snapshot
  (`read_task_parameters_tool`)
- Writing any subset of those fields and receiving the post-write snapshot on success
  (`write_task_parameters_tool`)
- The option lists (allowed enum values) and visibility flags returned alongside the state
- Validation rules that mirror the GUI (zone-gated `require_interaction` / `require_wait`, monitor index
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

Current values for the five sections. The `actor`, `mqtt`, `display`, and `task` sections are
`None` when the corresponding scene component is absent (no `ActorObject`, no `DisplayObject`,
etc.). `camera_mapping` is always a list — possibly empty when the OS reports no monitors — and
is never `None`.

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
  "task":           {"require_interaction": true, "require_wait": false, "track_length": 15000.0, "track_seed": -1}
}
```

Per-section persistence (scene file vs `EditorPrefs` vs `DisplaySettings` asset vs
`FullScreenViewsSaved` asset) lives in `/scene-setup` "Scene-specific vs project-wide state" —
that is the canonical map. The notes below cover only the **response-shape** semantics this skill
needs to interpret:
- `actor.model` is derived from the actor's child GameObject whose name starts with `Model `
  (`"None"` when absent).
- `actor.controller` is the assigned `ControllerOutput`'s GameObject name (`"None"` when null).
- `display.current_brightness` is the live runtime brightness (the "blank display" toggle in the
  GUI flips this to 0 or back to `brightness`).
- `display.brightness` is the configured default that the "Show Display" button restores to. When
  the active `DisplayObject` has no `DisplaySettings` asset assigned, the snapshot substitutes
  `100` for `brightness` and `0` for `height_in_vr` so the response stays well-formed; writes to
  those two fields are silently dropped in that state (see [Validation rules](#validation-rules)).
- `camera_mapping` is a list whose length equals the number of monitors the OS reports for the
  current scene; `monitor` is 1-based to match the GUI labels. The `left` and `top` fields are
  output-only — they report the monitor's OS-reported pixel origin and are silently ignored if
  included in a write payload.
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
    "require_interaction": true,
    "require_wait": false
  }
}
```

The GUI hides `Require Interaction` when no `GuidanceZone` exists in the scene and hides `Require Wait`
when no `OccupancyZone` exists. The bridge mirrors that by **rejecting** writes to those fields
when the matching zone is absent. When `visibility.task.<field> == false`, you MUST NOT include
the matching field in a write payload — the bridge will reject it. See
[Validation rules](#validation-rules) for the exact contract.

---

## Workflows

### Read the current state

```text
read_task_parameters_tool()
```

Returns `{"state": ..., "options": ..., "visibility": ..., "success": true}`. Use this snapshot
as the basis for any downstream decision (display rig confirmation, controller swap, MQTT broker
verification, Task field audit). The snapshot is a single scene walk; you MUST NOT poll it in a
tight loop expecting different results between consecutive frames.

### Update a subset of fields

`write_task_parameters_tool` accepts five optional top-level arguments (`actor`, `mqtt`,
`display`, `camera_mapping`, `task`). Pass only the sections you intend to change; fields inside a
section are individually optional too. On success the response is the post-write snapshot in the
same shape as `read_task_parameters_tool`, so a `read → modify → write → consume_snapshot` loop
never needs a second read.

```text
write_task_parameters_tool(
    actor={"controller": "Simulated Linear"},
    task={"track_seed": 42, "track_length": 8000.0},
)
```

Writes are processed sequentially in the order `actor → mqtt → display → camera_mapping → task`,
and within each section in declaration order. **Writes are not atomic**: at the first validation
failure the bridge returns `{"success": false, "error": "..."}` and stops, but any sections (or
earlier fields within the same section) that already succeeded keep their mutations on scene
components, `EditorPrefs`, scriptable-object assets, and `Display.transform`. Plan multisection
writes so a rejection of a later field does not leave the scene in a half-applied state, and read
back to confirm when in doubt.

On error the response is **only** `{"success": false, "error": "..."}` — no `state`, `options`, or
`visibility` keys. If a write fails partway through, the snapshot is not returned; a follow-up
`read_task_parameters_tool()` is the only way to recover the post-failure state.

`Undo` coverage is asymmetric: only the `task` section registers an undo step
(`Undo.RecordObject(task, "Write Task Parameters")`); writes to `actor`, `mqtt`, `display`, and
`camera_mapping` cannot be reverted with `Ctrl+Z` (`Cmd+Z` on macOS) in the Editor. You MUST NOT bundle multisection
writes expecting a single undo to roll them all back.

The bridge marks the active scene dirty when any write succeeds and runs `EditorUtility.SetDirty`
on any modified `DisplaySettings` asset (and calls `FullScreenViewManager.SaveCameras()`, which
internally runs `EditorUtility.SetDirty` plus `AssetDatabase.SaveAssets` on the
`FullScreenViewsSaved` asset). A subsequent `Ctrl+S` (`Cmd+S` on macOS) / `EditorSceneManager.SaveOpenScenes()`
persists every scene-level change. A `display.height_in_vr` write additionally translates the
`DisplayObject` GameObject by setting `display.transform.localPosition = (0, height_in_vr, 0)`,
so the scene's display rig moves in lockstep with the asset value.

### Verify a write took effect

When the write succeeds, the response includes the post-write snapshot, so the simplest
verification is to inspect the returned dict directly. When the write fails (`success == false`),
the snapshot is absent, and you MUST call `read_task_parameters_tool()` to inspect the partially
applied state.

### Swap controllers (Linear ↔ Simulated Linear)

Both controllers always exist under `Controllers` (see `/scene-setup` for the rig rationale and the
hardware-vs-keyboard contract). The programmatic swap is:

```text
# Inspect available controllers first so the write payload is well-formed.
state = read_task_parameters_tool()
allowed = state["options"]["actor"]["controller"]   # ["None", "Linear", "Simulated Linear"]

write_task_parameters_tool(actor={"controller": "Simulated Linear"})
```

You MUST NOT type a controller name from memory — always pull the list from
`options.actor.controller` because GameObject renames or future controller additions can change the
canonical names.

### Update the MQTT broker

`mqtt.ip` and `mqtt.port` writes propagate to the live `MQTTClient` and to `EditorPrefs`
(`SollertiaVR_MQTT_IP`, `SollertiaVR_MQTT_Port`) so a fresh Editor session inherits the value.
This means changes are **project-wide**, not per-scene; coordinate the change before running
multiple scenes in the same session.

### Toggle guidance modes for a runtime experiment

The Task fields (`require_interaction`, `require_wait`) are also addressable at runtime via MQTT
(`RequireInteraction` / `RequireWait` topics carrying `BoolMessage`). Editor-time writes through this
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
| `task`           | `require_interaction`          | Scene has no `GuidanceZone` (i.e., `visibility.task.require_interaction == false`)              |
| `task`           | `require_wait`                 | Scene has no `OccupancyZone` (i.e., `visibility.task.require_wait == false`)                    |

Other fields (`mqtt.ip`, `mqtt.port`, `display.*`, `task.track_length`, `task.track_seed`) accept
any numeric / string value the underlying `Convert.ToSingle` / `Convert.ToInt32` / `Convert.ToBoolean`
can parse; the bridge does not impose a range check here. The GUI itself does not either, so an
out-of-range brightness or a negative `track_length` will write but produce runtime warnings.

The zone-gated rejection of `require_interaction` and `require_wait` is **intentional**: a successful
write guarantees the flag will actually take effect at runtime. You MUST NOT paper over a
rejection by writing the underlying `Task` field through a different path — the missing zone means
the toggle has nothing to gate.

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

| Section          | GUI behavior in Play Mode                                                                                          |
|------------------|--------------------------------------------------------------------------------------------------------------------|
| `actor`          | Always editable (the GUI does not grey these out)                                                                  |
| `mqtt`           | Entire section greyed out — input fields do not accept changes and no `EditorPrefs` writes happen from the GUI     |
| `display`        | Editable; `current_brightness` changes are reflected immediately on the live display                               |
| `camera_mapping` | The "Show Full-Screen Views" button is disabled; field writes still go through                                     |
| `task`           | Greyed out — flip via MQTT (`RequireInteraction` / `RequireWait`) for mid-run changes instead                      |

The bridge does NOT enforce these GUI gates — `write_task_parameters_tool` ignores the Editor
Play Mode state and lets every section through, including `mqtt.*` writes that propagate to
`EditorPrefs` and the live `MQTTClient`, and `task.require_interaction` / `task.require_wait` writes that
the GUI would refuse. Treat the table above as a soft contract: prefer the MQTT path
(`RequireInteraction` / `RequireWait` topics) for guidance toggles once a run is in progress, and avoid
`mqtt.*` writes mid-session because mutating the live broker connection is unsupported.

Use `get_play_state_tool` (`/play-mode`) to check `state == "edit"` before issuing
`write_task_parameters_tool` calls that the GUI would refuse to apply.

---

## Troubleshooting

| Symptom                                                                         | Cause                                                                                        | Resolution                                                                                                                                     |
|---------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| `state.actor == null` even though an Actor exists                               | The Actor was placed outside the root and `FindAnyObjectByType<ActorObject>()` missed it     | Confirm the Actor is in the active scene; `inspect_scene_tool` (`/task-scenes`) to verify                                                      |
| `state.task == null`                                                            | No `Task` component in the active scene (only the empty `ExperimentTemplate` template scene) | `create_task_tool(scene_name=..., task_prefab_path=...)` (`/task-scenes`) to seed a task                                                       |
| Write rejected: "Invalid controller '...'"                                      | The controller name is not in `options.actor.controller`                                     | Re-read `options.actor.controller`; copy the exact string (it is the GameObject name)                                                          |
| Write rejected: "Cannot set require_interaction: scene has no GuidanceZone"     | The current task prefab has no interaction-mode segments                                     | The flag is not applicable — leave it alone, or open a scene whose template includes interaction-mode trials                                   |
| Write rejected: "Invalid monitor index N; scene has M monitors"                 | Camera mapping payload references a 1-based monitor index outside `[1, M]`                   | Re-read `state.camera_mapping` to enumerate valid `monitor` indices                                                                            |
| `state.camera_mapping == []`                                                    | OS reported zero monitors at scene load                                                      | Click "Refresh Monitor Positions" in the GUI or replug displays, then retry                                                                    |
| Writes succeed but the GUI shows old values                                     | The Parameters window cached the value before the write landed                               | The bridge already shares the GUI's `FullScreenViewManager` for camera mapping; for other sections close and reopen `Window → Task Parameters` |

---

## Related skills

| Skill                                         | Relationship                                                                                  |
|-----------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                                                      |
| `/task-scenes` (this plugin)                  | Upstream — switches the active scene that this skill reads and writes                         |
| `/scene-setup` (this plugin)                  | Upstream — owns the `MainWindow` GUI and the auto-creation of Actors / Controllers / Displays |
| `/play-mode` (this plugin)                    | Upstream — `get_play_state_tool` gates writes that the GUI greys out at runtime               |
| `/task-prefabs` (this plugin)                 | Upstream — generates the task prefab whose `Task` component this skill mutates                |
| `/mqtt-contract` (this plugin)                | Reference for the `RequireInteraction` / `RequireWait` runtime alternative to `task` writes   |
| `/gimbl-framework` (this plugin)              | Reference for `ActorObject`, `DisplayObject`, `MQTTClient`, and `ControllerOutput` semantics  |
| `assets:assets-mcp-environment-setup` | Upstream — owns the slsa MCP server diagnostic                                                |
| `experiment:vr-driver-interface`      | Host sets `RequireInteraction` / `RequireWait` at runtime via `set_*_guidance`                |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change that calls
`write_task_parameters_tool` or any code path that interprets a `read_task_parameters_tool`
snapshot.

```text
Task Parameters Compliance:
- [ ] Unity Editor is running and McpBridge is reachable (else /unity-mcp-environment-setup)
- [ ] read_task_parameters_tool runs before every write to capture the current options list
- [ ] Write payloads contain only field values present in options.<section>.<field>
- [ ] require_interaction / require_wait writes are gated on visibility.task.<field> == true
- [ ] camera_mapping entries use 1-based monitor indices matching state.camera_mapping[*].monitor
- [ ] Post-write snapshot is inspected to confirm the new state matches the requested change
- [ ] MQTT writes are avoided while the Editor is in Play Mode (use /play-mode to confirm state)
- [ ] Validation rules are not bypassed by editing the scene file directly to set rejected fields
```
