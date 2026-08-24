---
name: task-parameters
description: >-
  Reads and writes the consolidated Task Parameters editor window in sollertia-virtual-reality via
  the sollertia-shared-assets MCP server's Unity relay. Owns read_task_parameters_tool,
  write_task_parameters_tool, and refresh_monitors_tool, which mirror the window's five sections. Use
  when inspecting or changing per-scene Task, Actor, Display, MQTT, or Camera Mapping settings, or
  re-detecting the host's monitors, without opening the Editor window manually.
user-invocable: false
---

# Sollertia Unity task parameters

Programmatically reads and writes the consolidated **Task Parameters** Unity Editor window for
`sollertia-virtual-reality` through the Unity relay exposed by `slsa mcp`. This skill is the **exclusive** owner of
`read_task_parameters_tool`, `write_task_parameters_tool`, and `refresh_monitors_tool`, which no other skill in the
marketplace may call. That exclusivity binds marketplace *skills*, and the acquisition runtime's own HTTP bridge client
is a peer that drives the same endpoints directly during a session (see `experiment:vr-driver-interface`).

The window itself is owned by `MainWindow` (`Assets/Gimbl/Editor/MainWindow.cs`), and the read / write surface mirrors
the GUI's *field controls* plus the option lists and visibility flags the GUI uses to render them. Camera Mapping's
`Refresh Monitor Positions` button **is** exposed, as `refresh_monitors_tool`, and both paths call
`FullScreenViewManager.RefreshMonitorPositions`, so they re-detect identically. Action-only GUI controls that are not
exposed through the bridge include: MQTT `Test Connection`, Camera Mapping `Show Full-Screen Views`, and the Display
`Blank Display` / `Show Display` toggle button (its underlying effect is reachable through a
`display.current_brightness` write). The `Task` component's public fields are `[HideInInspector]` and `TaskEditor`
replaces the default Inspector with a HelpBox pointing at the Parameters window, so this skill is the *only*
programmatic entry point for those fields.

---

## Scope

**Covers:**
- Reading the active scene's Actor, MQTT, Display, Camera Mapping, and Task fields in one snapshot
  (`read_task_parameters_tool`)
- Writing any subset of those fields and receiving the post-write snapshot on success (`write_task_parameters_tool`)
- Re-detecting the host's monitors mid-session and receiving the post-refresh snapshot (`refresh_monitors_tool`)
- The option lists (allowed enum values) and visibility flags returned alongside the state
- Validation rules the bridge enforces (zone-gated `require_interaction` / `require_wait`, monitor index bounds,
  broker-port and numeric-finiteness bounds, controller / model / camera membership)
- Choosing between Parameters window writes and scene-file edits

**Does not cover:**
- Generating or validating prefabs, and creating the task prefab plus scene bundle (see `/task-prefabs`)
- Listing, opening, or inspecting scenes (see `/task-scenes`)
- Entering / exiting Play Mode (see `/play-mode`)
- Editor-time scene wiring beyond what the Parameters window exposes (see `/scene-setup`)
- MQTT topic catalog (see `/mqtt-contract`)
- The `MainWindow` GUI internals or controller / actor auto-creation behavior (see `/scene-setup` and
  `/gimbl-framework`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

---

## MCP tool surface

| Tool                         | Purpose                                                                           |
|------------------------------|-----------------------------------------------------------------------------------|
| `read_task_parameters_tool`  | Snapshot the active scene's Parameters state, options, and visibility (exclusive) |
| `write_task_parameters_tool` | Apply a subset of fields and return the post-write snapshot (exclusive)           |
| `refresh_monitors_tool`      | Re-detect the host's monitors and return the post-refresh snapshot (exclusive)    |

All three tools require the Unity Editor to be running with the `McpBridge` plugin active (else
`/unity-mcp-environment-setup`). All three share a single `AcquireSceneComponents` walk per request, so `read → write →
read` is consistent and writes never race a separate enumeration pass.

---

## Response shape

`read_task_parameters_tool`, the post-write payload from `write_task_parameters_tool`, and the post-refresh payload from
`refresh_monitors_tool` all return the same dict with three top-level keys: **`state`**, **`options`**,
**`visibility`**. Every section key is a stable string, and the values vary by section.

### `state`

Current values for the five sections. The `actor`, `mqtt`, `display`, and `task` sections are `None` when the
corresponding scene component is absent (no `ActorObject`, no `DisplayObject`, etc.). `camera_mapping` is always a list,
possibly empty when the OS reports no monitors, and is never `None`.

```json
{
  "actor":          {"model": "Rodent",    "controller": "Simulated Linear"},
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

Per-section persistence (scene file vs `EditorPrefs` vs `DisplaySettings` asset vs `FullScreenViewsSaved` asset) lives
in `/scene-setup` "Scene-specific vs project-wide state", which is the canonical map. The notes below cover only the
**response-shape** semantics this skill needs to interpret:
- `actor.model` is derived from the actor's child GameObject whose name starts with `Model ` (`"None"` when absent).
- `actor.controller` is the assigned `ControllerOutput`'s GameObject name (`"None"` when null).
- `display.current_brightness` is the live runtime brightness (the "blank display" toggle in the GUI flips this to 0 or
  back to `brightness`).
- `display.brightness` is the configured default that the "Show Display" button restores to. When the active
  `DisplayObject` has no `DisplaySettings` asset assigned, the snapshot substitutes `100` for `brightness` and `0` for
  `height_in_vr` so the response stays well-formed, and writes to those two fields are silently dropped in that state
  (see [Validation rules](#validation-rules)).
- A `display.brightness` write updates only the `DisplaySettings` asset. The GUI's brightness field additionally copies
  the new default into `display.currentBrightness` while the bridge does not, so pass both `brightness` and
  `current_brightness` when the live display must follow the new default.
- `camera_mapping` is a list whose length equals the number of monitors the OS reports for the current scene, and
  `monitor` is 1-based to match the GUI labels. The `left` and `top` fields are output-only, so they report the
  monitor's OS-reported pixel origin and are silently ignored if included in a write payload. The enumeration runs once
  per active scene and is cached (`McpBridge._cachedFullScreenManager`, cleared on every active-scene change), because
  detecting monitors spawns an OS subprocess on Linux and macOS. A read taken after the physical monitor arrangement
  changed still reports the old geometry until `refresh_monitors_tool()` (or the GUI's `Refresh Monitor Positions`
  button) re-detects.
- `task.track_seed == -1` is the documented sentinel for "nondeterministic seed".

### `options`

Enumerated alternatives for fields with a finite valid set. Use these as the authoritative allow-list when constructing
a `write_task_parameters_tool` payload, because values outside these lists are rejected by the bridge with a descriptive
error.

```json
{
  "actor": {
    "model":      ["Rodent", "None"],
    "controller": ["None", "Linear", "Simulated Linear"]
  },
  "camera_mapping": {
    "camera":     ["None", "Left View", "Center View", "Right View"]
  }
}
```

- `actor.model` is every prefab under `Resources/Actors/Prefabs/` plus the literal `"None"`.
- `actor.controller` is the GameObject name of every `ControllerOutput` in the active scene plus `"None"`.
  `MainWindow.EnsureControllers` populates this list with `Linear` and `Simulated Linear` on scene init (see
  `/scene-setup`).
- `camera_mapping.camera` is every scene `Camera` that is **not** tagged `MainCamera` and **not** named `Main Camera`,
  plus `"None"`. This matches the filter the GUI dropdown applies so the agent and the user see the same option set.

`state.camera_mapping[*].camera` and `options.camera_mapping.camera` are **not** built from the same query: the state
resolves the persisted assignment through `EditorUtility.EntityIdToObject` regardless of the GameObject's active state,
while the options list comes from `FindObjectsByType<Camera>(FindObjectsSortMode.None)`, which skips inactive objects. A
camera bound to a deactivated GameObject therefore appears in `state` but not in `options`, and echoing that state name
back in a write payload is rejected with `Invalid camera '...' for monitor N`. You MUST filter a round-tripped payload
against `options`, not against `state`.

No options list is returned for `mqtt`, `display`, or `task` because their fields are free-form numeric or boolean
values (not enumerations).

### `visibility`

Per-control flags that report whether the matching GUI control is currently rendered. Today only the `task` section uses
this:

```json
{
  "task": {
    "require_interaction": true,
    "require_wait": false
  }
}
```

The GUI hides `Require Interaction` when no `GuidanceZone` exists in the scene and hides `Require Wait` when no
`OccupancyZone` exists. The bridge mirrors that by **rejecting** writes to those fields when the matching zone is
absent. When `visibility.task.<field> == false`, you MUST NOT include the matching field in a write payload, because the
bridge will reject it. See [Validation rules](#validation-rules) for the exact contract.

---

## Workflows

### Read the current state

```text
read_task_parameters_tool()
```

Returns `{"state": ..., "options": ..., "visibility": ..., "success": true}`. Use this snapshot as the basis for any
downstream decision (display rig confirmation, controller swap, MQTT broker verification, Task field audit). The
snapshot is a single scene walk, so you MUST NOT poll it in a tight loop expecting different results between consecutive
frames.

### Update a subset of fields

`write_task_parameters_tool` accepts five optional top-level arguments (`actor`, `mqtt`, `display`, `camera_mapping`,
`task`). Pass only the sections you intend to change, and fields inside a section are individually optional too. On
success the response is the post-write snapshot in the same shape as `read_task_parameters_tool`, so a `read → modify →
write → consume_snapshot` loop never needs a second read for plain field values.

```text
write_task_parameters_tool(
    actor={"controller": "Simulated Linear"},
    task={"track_seed": 42, "track_length": 8000.0},
)
```

**Writes are atomic.** `WriteTaskParameters` runs `ValidateTaskParameterWrites` over the entire request before applying
anything, so a rejection leaves the scene, `EditorPrefs`, the scriptable-object assets, and `Display.transform`
completely untouched. Validation visits the sections in the order `actor → mqtt → display → camera_mapping → task` and
returns the first failure it finds. When validation passes, the sections apply in that same order, and within each
section in declaration order.

The post-write snapshot is rendered from the component references the single pre-write scene walk acquired. Field values
it re-reads off those components are current, but anything derived from scene *structure* still describes the pre-write
scene, which covers `options.actor.controller`, `options.camera_mapping.camera`, and both `visibility.task` flags. A
write that changes the hierarchy requires a follow-up `read_task_parameters_tool()` before those lists and flags can be
trusted. The two such writes are `actor.model`, which destroys the previous `Model <name>` child and re-instantiates a
new one, and `actor.controller`, which can change what `ControllerOutput` list a later read returns.

On error the response is **only** `{"success": false, "error": "..."}`, with no `state`, `options`, or `visibility`
keys. A rejected write applies nothing, so the state the failed call reports on is exactly the state the preceding
`read_task_parameters_tool()` returned, and no recovery read is required after a rejection.

`Undo` coverage is asymmetric. Only the `task` section registers an undo step (`Undo.RecordObject(task, "Write Task
Parameters")`), and writes to `actor`, `mqtt`, `display`, and `camera_mapping` cannot be reverted with `Ctrl+Z` (`Cmd+Z`
on macOS) in the Editor. You MUST NOT bundle multi-section writes expecting a single undo to roll them all back.

The bridge marks the active scene dirty when any write succeeds and runs `EditorUtility.SetDirty` on any modified
`DisplaySettings` asset (and calls `FullScreenViewManager.SaveCameras()`, which internally runs `EditorUtility.SetDirty`
plus `AssetDatabase.SaveAssets` on the `FullScreenViewsSaved` asset). It also runs `Undo.RecordObject(task, "Write Task
Parameters")` plus `EditorUtility.SetDirty(task)` on the `Task` component whenever a `task` section object is supplied
and a `Task` exists, even a section carrying no recognized fields. `ApplyCameraMappingSection` is the one applier that
explicitly guards against that. A `camera_mapping` list whose rows are all skipped (no `monitor` key, no string
`camera`) returns before `SaveCameras()`, so it neither rewrites the `FullScreenViewsSaved` companion asset nor dirties
the scene. A subsequent `Ctrl+S` (`Cmd+S` on macOS) or `EditorSceneManager.SaveOpenScenes()` persists every scene-level
change. A `display.height_in_vr` write additionally translates the `DisplayObject` GameObject by setting
`display.transform.localPosition = (0, height_in_vr, 0)`, so the scene's display rig moves in lockstep with the asset
value.

### Verify a write took effect

When the write succeeds, the response includes the post-write snapshot, so the simplest verification is to inspect the
returned dict directly, with the caveat above that its `options` and `visibility` sections describe the pre-write scene.
When the write fails (`success == false`) the snapshot is absent, but nothing was applied: the scene still holds the
values the last successful read reported. Fix the rejected field and resend the whole payload.

### Refresh the monitor list

```text
refresh_monitors_tool()
```

Takes no arguments and returns the same `state` / `options` / `visibility` shape the other two tools return, with
`state.camera_mapping` rebuilt from a fresh `Monitor.EnumerateMonitors()` pass. Call it whenever the physical monitor
arrangement changed mid-session: the bridge builds one `FullScreenViewManager` per active scene and only discards it on
an active-scene change, so a plain `read_task_parameters_tool()` keeps reporting the stale geometry indefinitely. The
GUI's `Refresh Monitor Positions` button is the same code path (`FullScreenViewManager.RefreshMonitorPositions`), so the
two re-detect identically.

Two consequences you MUST account for:

- **Assignments carry across by monitor index, not by identity.** Removing a monitor from the middle of the arrangement
  shifts every later assignment up one slot. Re-read `state.camera_mapping` after a refresh and re-bind explicitly
  rather than assuming the bindings survived.
- **The refreshed list is not persisted.** It stays in memory until a camera assignment is written through
  `write_task_parameters_tool`, which is what calls `SaveCameras()` on the per-scene `FullScreenViewsSaved` companion
  asset.

A refresh is also the prerequisite for the zero-monitor write refusal: `write_task_parameters_tool` rejects any
`camera_mapping` payload while the host reports no monitors, rather than erasing the saved assignments (see [Validation
rules](#validation-rules)).

### Swap controllers (Linear ↔ Simulated Linear)

Both controllers always exist under `Controllers` (see `/scene-setup` for the rig rationale and the hardware-vs-keyboard
contract). The programmatic swap is:

```text
# Inspect available controllers first so the write payload is well-formed.
state = read_task_parameters_tool()
allowed = state["options"]["actor"]["controller"]   # ["None", "Linear", "Simulated Linear"]

write_task_parameters_tool(actor={"controller": "Simulated Linear"})
```

You MUST NOT type a controller name from memory. Always pull the list from `options.actor.controller` because GameObject
renames or future controller additions can change the canonical names.

### Update the MQTT broker

`mqtt.ip` and `mqtt.port` writes propagate to the live `MQTTClient` and to `EditorPrefs` (`SollertiaVR_MQTT_IP`,
`SollertiaVR_MQTT_Port`) so a fresh Editor session inherits the value. This means changes are **project-wide** rather
than per-scene, so coordinate the change before running multiple scenes in the same session.

### Toggle guidance modes for a runtime experiment

The Task fields (`require_interaction`, `require_wait`) are also addressable at runtime via MQTT (`RequireInteraction` /
`RequireWait` topics carrying `BoolMessage`). Editor-time writes through this tool persist the value in the scene, while
MQTT writes change the live value during Play Mode without modifying the scene. Use this tool to set the **default**
before entering Play Mode and the MQTT path to flip it mid-experiment.

---

## Validation rules

`ValidateTaskParameterWrites` runs the whole set below before any section applies. Each rejection returns an
`{"success": false, "error": "..."}` response with a descriptive message and mutates nothing.

| Section          | Field                 | Rejection condition                                                                                |
|------------------|-----------------------|----------------------------------------------------------------------------------------------------|
| `actor`          | `model`               | Value is not in `options.actor.model`                                                              |
| `actor`          | `controller`          | Value is not in `options.actor.controller`                                                         |
| `mqtt`           | `port`                | Not a whole number, or outside `[0, 65535]`                                                        |
| `display`        | `current_brightness`  | Value does not convert to a finite float                                                           |
| `display`        | `brightness`          | Value does not convert to a finite float                                                           |
| `display`        | `height_in_vr`        | Value does not convert to a finite float                                                           |
| `camera_mapping` | (whole section)       | The host reported zero monitors, so the write is refused rather than erasing the saved assignments |
| `camera_mapping` | `monitor`             | Row carries no `monitor` key, or the value is not a whole number                                   |
| `camera_mapping` | `monitor`             | 1-based index outside `[1, monitors.Count]`                                                        |
| `camera_mapping` | `camera`              | Value is not in `options.camera_mapping.camera`                                                    |
| `task`           | `require_interaction` | Scene has no `GuidanceZone` (i.e., `visibility.task.require_interaction == false`)                 |
| `task`           | `require_wait`        | Scene has no `OccupancyZone` (i.e., `visibility.task.require_wait == false`)                       |
| `task`           | `require_interaction` | Value does not convert to a boolean                                                                |
| `task`           | `require_wait`        | Value does not convert to a boolean                                                                |
| `task`           | `track_length`        | Not a positive, finite number                                                                      |
| `task`           | `track_seed`          | Not a whole number                                                                                 |

The zero-monitor refusal is recoverable: call `refresh_monitors_tool()` once the host enumerates monitors again, then
resend the `camera_mapping` payload.

Validation and application are both gated on the owning component existing. A write to a section whose component is
absent from the active scene is silently ignored and still returns `success: true`, even for values that would otherwise
be rejected. The absent cases are `actor` with no `ActorObject`, `mqtt` with no `MQTTClient`, `display` with no
`DisplayObject`, and `task` with no `Task`. You MUST confirm the matching `state.<section>` is non-null in the returned
snapshot before treating a write as applied.

The camera mapping path carries one GUI-only guard, so a write there can produce a binding the GUI refuses to make.
`FullScreenViewManager.RenderMonitorRow` scans every monitor for the selected camera's `EntityId` and skips the
assignment when that camera is already bound, so the GUI silently keeps the row on its previous camera. The bridge
assigns `cameraEntityId` for every valid monitor / camera pair and then calls `SaveCameras()`, so
`write_task_parameters_tool` can bind one camera to two monitors. The guard also matches the monitor being edited, which
makes re-selecting a row's current camera a harmless no-op in the GUI. Monitors left out of a write payload keep their
current bindings, and the bridge inspects neither them nor the other rows of the payload. You MUST confirm that the
post-write `state.camera_mapping` holds each camera name at most once, unless a duplicate binding is intended. The
`"None"` value is exempt because it clears a monitor, so it may repeat across as many rows as needed. `"None"` is also
the second point where the bridge is more permissive than the GUI. `RenderMonitorRow` refuses to clear a row whose
dropdown merely *parked* on `None` because the assigned camera sits on a deactivated GameObject, while
`ApplyCameraMappingSection` writes `EntityId.None` unconditionally. A `"None"` write therefore clears assignments the
GUI would have preserved.

`mqtt.ip` is the only field the bridge accepts unconditionally (any string, and a non-string value is ignored rather
than rejected). `mqtt.port` is bounded to `[0, 65535]` because the value reaches both the live client and the
`EditorPrefs` entry a fresh session reloads from. `display.current_brightness` / `brightness` / `height_in_vr` must
convert to finite floats but are not range-checked, and nothing downstream clamps or warns. `PerspectiveProjection`
passes `currentBrightness` straight to the display shader, so an out-of-range value writes and takes effect silently.
`task.track_length` must be strictly positive and finite, so zero and negative values are rejected, and that bridge
bound is the only check applied here. A `track_length` too short to cover the template's corridor still writes
successfully and then disables the `Task` at the next Play Mode entry with `Task: trackLength <n> is too short for
template '<name>'.` `/task-generator` owns that runtime contract and `ValidateTrackLengthCoversCorridor`, the
generation-time gate that keeps it unreachable at the generated value. `task.track_seed` must convert to a 32-bit
integer.

The zone-gated rejection of `require_interaction` and `require_wait` is **intentional**, because a successful write
guarantees the flag will actually take effect at runtime. The bridge says so verbatim, in `Cannot set
require_interaction: the active scene has no GuidanceZone, so the control is hidden in the Parameters window and the
flag has no runtime effect.` (and the `require_wait` twin naming `OccupancyZone`). You MUST NOT paper over a rejection
by writing the underlying `Task` field through a different path, because the missing zone means the toggle has nothing
to gate.

---

## Path conventions

This skill does not take filesystem paths directly, and every write targets the active scene the Unity Editor currently
has open. Use `/task-scenes` to switch scenes before issuing Parameters writes:

```text
open_scene_tool(scene_path="Assets/Scenes/<name>.unity")   # /task-scenes
write_task_parameters_tool(task={"track_seed": 42})         # this skill
```

The bridge resolves every Parameters target relative to the active scene's components, so two writes against different
scenes require an `open_scene_tool` call between them.

---

## Interaction with Play Mode

The GUI disables several controls while the Editor is in Play Mode:

| Section          | GUI behavior in Play Mode                                                                                        |
|------------------|------------------------------------------------------------------------------------------------------------------|
| `actor`          | Always editable (the GUI does not grey these out)                                                                |
| `mqtt`           | Entire section greyed out, so input fields do not accept changes and no `EditorPrefs` writes happen from the GUI |
| `display`        | Editable, and `current_brightness` changes are reflected immediately on the live display                         |
| `camera_mapping` | The "Show Full-Screen Views" button is disabled, and field writes still go through                               |
| `task`           | Greyed out, so flip via MQTT (`RequireInteraction` / `RequireWait`) for mid-run changes instead                  |

The bridge does NOT enforce these GUI gates. `write_task_parameters_tool` ignores the Editor Play Mode state and lets
every section through, including `mqtt.*` writes that propagate to `EditorPrefs` and the live `MQTTClient`, and
`task.require_interaction` / `task.require_wait` writes that the GUI would refuse. Treat the table above as a soft
contract. Prefer the MQTT path (`RequireInteraction` / `RequireWait` topics) for guidance toggles once a run is in
progress, and avoid `mqtt.*` writes mid-session because mutating the live broker connection is unsupported.

Use `get_play_state_tool` (`/play-mode`) to check `state == "edit"` before issuing `write_task_parameters_tool` calls
that the GUI would refuse to apply.

---

## Troubleshooting

| Symptom                                                                                     | Cause                                                                                                                                                                  | Resolution                                                                                                                      |
|---------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `state.actor == null` even though an Actor exists                                           | The `ActorObject` is on a deactivated GameObject or lives in a loaded scene other than the active one, and `FindAnyObjectByType<ActorObject>()` skips inactive objects | Confirm the Actor is in the active scene, using `inspect_scene_tool` (`/task-scenes`) to verify                                 |
| `state.task == null`                                                                        | No `Task` component in the active scene (only the empty `ExperimentTemplate` template scene)                                                                           | `create_task_tool(template_name=...)` (`/task-prefabs`) to seed a task                                                          |
| Write rejected: "Invalid controller '...'"                                                  | The controller name is not in `options.actor.controller`                                                                                                               | Re-read `options.actor.controller`, then copy the exact string (it is the GameObject name)                                      |
| Write rejected: "Cannot set require_interaction: the active scene has no GuidanceZone, ..." | The current task prefab has no interaction-mode segments                                                                                                               | The flag is not applicable, so leave it alone or open a scene whose template includes interaction-mode trials                   |
| Write rejected: "Invalid monitor index N; scene has M monitors"                             | Camera mapping payload references a 1-based monitor index outside `[1, M]`                                                                                             | Re-read `state.camera_mapping` to enumerate valid `monitor` indices                                                             |
| Write rejected: "Invalid track_length '...'. Must be a positive, finite number ..."         | The payload carried a zero, negative, or non-finite `track_length`                                                                                                     | Send a positive, finite value, since the whole write was rejected and nothing else in the payload applied either                |
| `state.camera_mapping == []`                                                                | On macOS / Linux the monitor-enumeration helper is missing, and otherwise the host reported zero monitors                                                              | Install the helper (see `/unity-mcp-environment-setup`), confirm the displays are attached, then call `refresh_monitors_tool()` |
| Write rejected: "Cannot write camera_mapping: no monitors were detected on this host. ..."  | The write is refused rather than allowed to erase the saved assignments                                                                                                | Resolve monitor enumeration as above, call `refresh_monitors_tool()`, then resend the payload                                   |
| Writes succeed but the GUI shows old values                                                 | The Parameters window has not repainted, because it caches component references rather than field values                                                               | Click into the Parameters tab to force a repaint, and reopening the window is not required                                      |

Camera mapping is the exception to the repaint caveat: the bridge reuses the open Parameters window's own
`FullScreenViewManager`, so the GUI row and the snapshot never diverge.

---

## Related skills

| Skill                                        | Relationship                                                                                 |
|----------------------------------------------|----------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable                                                     |
| `/task-scenes` (this plugin)                 | Upstream, switches the active scene that this skill reads and writes                         |
| `/scene-setup` (this plugin)                 | Upstream, owns the `MainWindow` GUI and the auto-creation of Actors / Controllers / Displays |
| `/play-mode` (this plugin)                   | Upstream, `get_play_state_tool` gates writes that the GUI greys out at runtime               |
| `/task-prefabs` (this plugin)                | Upstream, generates the task prefab whose `Task` component this skill mutates                |
| `/task-generator` (this plugin)              | Owns the `Task.cs` startup contract that a lowered `task.track_length` violates              |
| `/mqtt-contract` (this plugin)               | Reference for the `RequireInteraction` / `RequireWait` runtime alternative to `task` writes  |
| `/gimbl-framework` (this plugin)             | Reference for `ActorObject`, `DisplayObject`, `MQTTClient`, and `ControllerOutput` semantics |
| `assets:assets-mcp-environment-setup`        | Upstream, owns the slsa MCP server diagnostic                                                |
| `experiment:vr-driver-interface`             | Peer client, rebinds `actor.controller` over these same endpoints mid-session                |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change that calls `write_task_parameters_tool` or
any code path that interprets a `read_task_parameters_tool` snapshot.

```text
Task Parameters Compliance:
- [ ] Unity Editor is running and McpBridge is reachable (else /unity-mcp-environment-setup)
- [ ] read_task_parameters_tool runs before every write to capture the current options list
- [ ] Write payloads contain only field values present in options.<section>.<field>, never names
      echoed back from state.<section>
- [ ] require_interaction / require_wait writes are gated on visibility.task.<field> == true
- [ ] camera_mapping entries use 1-based monitor indices matching state.camera_mapping[*].monitor
- [ ] refresh_monitors_tool is called after any physical monitor change, and state.camera_mapping is
      re-read afterwards because assignments carry across by index
- [ ] Post-write snapshot is inspected to confirm the new state matches the requested change, and a
      follow-up read is issued when the write changed scene structure
- [ ] state.<section> is non-null before a write to that section is treated as applied
- [ ] MQTT writes are avoided while the Editor is in Play Mode (use /play-mode to confirm state)
- [ ] Validation rules are not bypassed by editing the scene file directly to set rejected fields
```
