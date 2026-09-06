---
name: vr-driver-interface
description: >-
  Documents the Virtual Reality task driver subsystem: the VRTaskDriver class and its configuration, the MQTT topic
  contract with the Unity game engine, the editor MCP Bridge it drives for scene activation and Play Mode control, the
  per-cycle _VRTaskEvent model, and the cue-sequence trial decomposition. Use when modifying Unity coupling, the editor
  bridge, adding an MQTT topic or VR task event, or wiring a new acquisition system to Unity.
user-invocable: false
---

# VR task driver interface

Documents the host (Python) side of the Virtual Reality task driver subsystem, the acquisition-system-agnostic
`VRTaskDriver` (`vr_task/driver.py`) that couples a Sollertia acquisition runtime to the Unity game engine
implemented in `sollertia-virtual-reality`.

---

## Scope

**Covers:**
- The `vr_task` package export surface and its module constants
- `VRTaskConfiguration` and `load_vr_task_template`, the two setup-time configuration assets
- `VRTaskDriver` connection lifecycle, bridge-driven setup handshake, per-cycle pump, guidance toggles
- `UnityBridgeClient`, the editor MCP Bridge HTTP client that opens scenes, binds the actor's motion controller,
  and controls Play Mode
- The `_VRTaskMQTTTopics` contract, meaning every topic, its payload shape, and its direction
- `VRTaskEventKind` / `_VRTaskEvent`, the typed events `cycle()` surfaces, and the `_VRTaskState` they share
  with the setup handshake
- Cue-sequence trial decomposition (`DecomposedTrials`, `CachedMotifDecomposer`, `decompose_cue_sequence`)
- What a consumer acquisition system must supply to drive the package
- Workflows for adding an MQTT topic or a VR task event

**Does not cover:**
- The Unity-side framework and game objects, owned by `unity:gimbl-framework`
- The Unity-side editor MCP Bridge endpoint (`McpBridge.cs`), scene authoring, and Play-Mode authoring, owned by
  `unity:play-mode` and `unity:task-scenes`
- The Unity-side Task Parameters read / write surface the controller rebind travels over, owned by
  `unity:task-parameters`
- The Unity-side MQTT topic registration and `MQTTTopics` constant set, owned by `unity:mqtt-contract`
- Unity task prefab generation from templates, owned by `unity:task-prefabs`
- `TaskTemplate` authoring (cue catalog, corridor geometry, per-trial cue motifs, trigger types), owned by
  `assets:task-templates`
- The task-templates directory and the rest of the platform working directory, owned by `assets:working-directory`
- The runtime state machine that consumes the driver's events, owned by `mesoscope:mesoscope-vr-runtime`
- The trigger-type-to-trial mapping a system applies to `trial_names`, owned by
  `mesoscope:mesoscope-vr-experiment-schema`
- The platform-general runtime and orchestrator pattern, owned by `/acquisition-system-runtime`

---

## Subsystem role

The VR task driver is the platform-general VR subsystem, parallel to `/microcontroller-interface` (microcontrollers) and
`/zaber-interface` (motors). The module docstring of `vr_task/__init__.py` names it acquisition-system-agnostic, and an
acquisition system's runtime orchestrator composes it.

An orchestrator builds the driver only for the session types that run the linear infinite corridor task.
`SESSION_TYPES_USING_VR_TASK` (`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`) declares that
set, and a session of such a type also carries a `vr_configuration.yaml` task-template snapshot. The orchestrator
holds `None` in place of a driver for every other session type, so each consumer site guards on the driver existing.

Mesoscope-VR is currently the only registered acquisition system, and its orchestrator is the current worked
example. See `mesoscope:mesoscope-vr-runtime`.

The Unity side of the contract, meaning the GIMBL framework, the `MQTTTopics` constant set, and task prefab
generation, lives in the unity plugin. This skill owns the host (Python) side.

---

## Authoritative bases

The `communication:microcontroller-interface` skill referenced below lives in the ataraxis marketplace.

| Concern                                                              | Authority                                 |
|----------------------------------------------------------------------|-------------------------------------------|
| `MQTTCommunication` mechanics (connect, monitored topics, get_data)  | `communication:microcontroller-interface` |
| Unity-side editor MCP Bridge (scene / Play-Mode tools)               | `unity:play-mode`, `unity:task-scenes`    |
| Unity-side Task Parameters window (actor motion controller)          | `unity:task-parameters`                   |
| Unity-side MQTT topic contract (`MQTTTopics`)                        | `unity:mqtt-contract`                     |
| Unity-side VR framework and game objects                             | `unity:gimbl-framework`                   |
| Unity task prefab and scene generation from templates                | `unity:task-prefabs`                      |
| `TaskTemplate` schema (cue catalog, geometry, motifs, trigger types) | `assets:task-templates`                   |
| Task-templates directory the loader reads                            | `assets:working-directory`                |
| Runtime that consumes this driver                                    | `mesoscope:mesoscope-vr-runtime`          |

The driver builds on `ataraxis_communication_interface.MQTTCommunication` and only documents the Sollertia VR contract
layered on top.

---

## Package exports

The `__all__` list of `vr_task/__init__.py` holds six names: `StimulusCause`, `UnityBridgeClient`,
`VRTaskConfiguration`, `VRTaskDriver`, `VRTaskEventKind`, and `load_vr_task_template`. `_VRTaskEvent`,
`_VRTaskState`, `UnityBridgeError`, `DecomposedTrials`, `CachedMotifDecomposer`, `decompose_cue_sequence`, and
`_VRTaskMQTTTopics` carry no package-level export, so an importer reaches each one through its defining submodule.

---

## Module constants

Every knob the driver tunes is a module constant in `vr_task/driver.py`, outside `VRTaskConfiguration`.

| Constant                            | Value      | Purpose                                                                                                    |
|-------------------------------------|------------|------------------------------------------------------------------------------------------------------------|
| `_LINEAR_CONTROLLER_NAME`           | `"Linear"` | Scene GameObject name of the treadmill controller enforced                                                 |
| `_SETUP_POLLING_DELAY_MS`           | `10`       | Delay between MQTT buffer polls during setup                                                               |
| `_DISPLAY_ANIMATION_STEP_DELAY_MS`  | `100`      | Delay between display-verification position updates                                                        |
| `_DISPLAY_ANIMATION_STEP_UNITS`     | `0.1`      | Unity units per verification animation step                                                                |
| `_DISPLAY_SCREENS_WARMUP_DELAY_MS`  | `2000`     | Screen settle delay applied before animating                                                               |
| `_CUE_SEQUENCE_RESPONSE_TIMEOUT_MS` | `5000`     | Cue-sequence reply wait before a retry prompt                                                              |
| `_SCENE_NAME_RESPONSE_TIMEOUT_MS`   | `5000`     | Scene-name reply wait before a retry prompt                                                                |
| `_PLAY_MODE_READINESS_TIMEOUT_MS`   | `60000`    | Wait for `SessionStart` after Play Mode entry, generous because entry can trigger a Unity script recompile |
| `_SESSION_STOP_TIMEOUT_MS`          | `5000`     | Wait for `SessionStop` after Play Mode exit                                                                |

---

## Configuration

The driver reads `VRTaskConfiguration` (`vr_task/configuration.py`), a frozen slots dataclass:

| Field  | Type  | Default       | Purpose                                                           |
|--------|-------|---------------|-------------------------------------------------------------------|
| `ip`   | `str` | `"127.0.0.1"` | IP address of the MQTT broker used to reach the Unity game engine |
| `port` | `int` | `1883`        | Port number of the MQTT broker                                    |

`VRTaskConfiguration` stores **only** the MQTT broker discovery fields. An acquisition system nests it inside its own
system configuration and hands it to the driver. For the current worked example, see `mesoscope:mesoscope-vr`. The
geometric VR parameters (cue catalog, corridor geometry, cm-per-Unity-unit, per-trial cue motifs, trigger types)
live in the `TaskTemplate` that `load_vr_task_template(unity_scene_name)` resolves at experiment start.

### `load_vr_task_template`

`load_vr_task_template` (`vr_task/configuration.py`) resolves the task-templates directory that the
`slsa configure templates` CLI persists, then expects `<unity_scene_name>.yaml` inside it. A missing directory or an
unset directory setting raises `FileNotFoundError` from the `get_task_templates_directory` resolver
(`sollertia-shared-assets/src/sollertia_shared_assets/configuration/configuration_utilities.py`), and a missing template
file raises `FileNotFoundError` through `console.error` listing the sorted available stems.

The loader validates existence only. `TaskTemplate.__post_init__` owns every content check, covering cue-code and
cue-name uniqueness, the trial-name pattern, cue references, transitions, trigger types, zone bounds, and cue-sequence
uniqueness (`sollertia-shared-assets/src/sollertia_shared_assets/configuration/vr_configuration.py`). The directory
itself is platform state owned by `assets:working-directory`, and template authoring by `assets:task-templates`.

Scene activation and Play Mode travel over the editor MCP Bridge (see the next section), whose endpoint is a fixed
loopback constant (`127.0.0.1:8090`) in `bridge.py`, deliberately outside `VRTaskConfiguration`. The bridge binds the
loopback interface only, so the acquisition host must run on the same machine as the Unity Editor.

---

## MQTT topic contract

`_VRTaskMQTTTopics` (a `StrEnum` in `vr_task/driver.py`) mirrors the flat PascalCase `MQTTTopics` constant set
published by `sollertia-virtual-reality` (see `unity:mqtt-contract`). Both sides MUST agree on these strings exactly.

| Topic enum             | Wire string          | Direction       | Payload                                                                 |
|------------------------|----------------------|-----------------|-------------------------------------------------------------------------|
| `SESSION_START`        | `SessionStart`       | Unity → runtime | empty trigger (Unity MQTT client started)                               |
| `SESSION_STOP`         | `SessionStop`        | Unity → runtime | empty trigger (Unity application quit)                                  |
| `MOTION`               | `Motion`             | runtime → Unity | `TreadmillMessage` `{movement: float}` (Unity-unit delta)               |
| `INTERACTION`          | `Interaction`        | runtime → Unity | empty trigger                                                           |
| `STIMULUS`             | `Stimulus`           | Unity → runtime | `StimulusMessage` `{trialName: string, delivered: bool, cause: string}` |
| `DELAY`                | `Delay`              | Unity → runtime | `TriggerDelayMessage` `{delayMilliseconds: uint}`                       |
| `CUE_SEQUENCE_TRIGGER` | `CueSequenceTrigger` | runtime → Unity | empty trigger (request flattened cue sequence)                          |
| `CUE_SEQUENCE`         | `CueSequence`        | Unity → runtime | `SequenceMessage` `{cueSequence: byte[]}`                               |
| `SCENE_NAME_TRIGGER`   | `SceneNameTrigger`   | runtime → Unity | empty trigger (request active scene name)                               |
| `SCENE_NAME`           | `SceneName`          | Unity → runtime | `SceneNameMessage` `{name: string}`                                     |
| `REQUIRE_INTERACTION`  | `RequireInteraction` | runtime → Unity | `BoolMessage` `{value: bool}` (inverse of reinforcing guidance)         |
| `REQUIRE_WAIT`         | `RequireWait`        | runtime → Unity | `BoolMessage` `{value: bool}` (inverse of aversive guidance)            |

The `monitored_topics` tuple in `VRTaskDriver.__init__` fixes the monitored (subscribed) set at exactly six topics,
`CUE_SEQUENCE`, `SESSION_STOP`, `SESSION_START`, `SCENE_NAME`, `STIMULUS`, and `DELAY`.

Unity's `StimulusMessage` carries all three fields on every send. The host driver parses them defensively,
resolving an absent `delivered` to `True` and any `cause` value other than the explicit `guidance` marker to
`BEHAVIOR`.

---

## Unity editor MCP Bridge

Alongside the MQTT data channel, the driver controls the Unity Editor over the **editor MCP Bridge**, the HTTP
listener that `McpBridge.cs` starts automatically inside the Unity Editor (Unity side: `unity:play-mode`,
`unity:task-scenes`, `unity:task-parameters`). `UnityBridgeClient` (`vr_task/bridge.py`) is the host-side client,
and it is **mandatory and always on**, because the driver carries no enable/disable field and no manual prompt
fallback. The endpoint is a pair of loopback module constants, `_BRIDGE_HOST` (`127.0.0.1`) and `_BRIDGE_PORT` (`8090`),
alongside the 5.0 second per-request `_BRIDGE_REQUEST_TIMEOUT_S`, all in `vr_task/bridge.py`, so the acquisition host
MUST be the Unity Editor's own machine.

Wire protocol: POST JSON `{"tool": ..., "args": {...}}`, and the reply always carries `success: bool` with an `error`
string on a failure. `UnityBridgeClient` maps a transport failure, a malformed or non-object JSON body, and a
`success: false` reply to `UnityBridgeError`, which the driver layer catches to retry, surface, or probe
reachability.

| Method                                    | Bridge tool             | Purpose                                                        |
|-------------------------------------------|-------------------------|----------------------------------------------------------------|
| `_list_scenes()`                          | `list_scenes`           | All project scene paths and the active scene path              |
| `open_scene(scene_path, unsaved_changes)` | `open_scene`            | Open a scene (the driver takes the default `save` policy)      |
| `get_active_controller()`                 | `read_task_parameters`  | The motion controller bound to the active scene's actor        |
| `set_active_controller(controller_name)`  | `write_task_parameters` | Bind a motion controller to the actor, returns the bound name  |
| `enter_play_mode()`                       | `enter_play_mode`       | Enter Play Mode (arm), async, returns the post-request state   |
| `exit_play_mode()`                        | `exit_play_mode`        | Exit Play Mode (disarm)                                        |
| `get_play_state()`                        | `get_play_state`        | The editor play state and active scene name                    |
| `resolve_scene_path(scene_name)`          | (via `list_scenes`)     | Resolve `expected_scene_name` to a project scene path by stem  |
| `is_reachable()`                          | (via `get_play_state`)  | True when the bridge responds, never raises                    |
| `describe_status()`                       | (via `get_play_state`)  | One-line reachability / scene / state summary                  |
| `close()`                                 | (none)                  | Close the underlying HTTP client, called during `disconnect()` |

`enter_play_mode()` only presses the editor's play button. The MQTT `SessionStart` message remains the authoritative
"Unity is armed and connected" signal, so `setup()` still waits for it after arming. A dedicated reachability check is
surfaced to pre-flight through the `sle get unity` CLI command (`get_unity_bridge` in `interfaces/get.py`) and the
`check_unity_bridge_tool` MCP tool (`interfaces/get_tools.py`), both of which build a bare `UnityBridgeClient` (see
`/system-health-check`, `/acquisition-system-setup`).

---

## Event model

`cycle()` consumes **at most one** MQTT message per call and returns a typed `_VRTaskEvent`, whose `kind` is a
`VRTaskEventKind` member and whose remaining fields are populated per kind. `_VRTaskState`, the driver's `state`
property, is the single source of truth shared between the setup handshake and per-cycle events.

For the event-kind table, the `_VRTaskEvent` field set, and the `_VRTaskState` field table, see
[`references/event-model.md`](references/event-model.md).

---

## VRTaskDriver API

Construction, performed only for a session type that drives Unity:

```python
VRTaskDriver(
    configuration,                  # VRTaskConfiguration, nested in the system configuration
    *,
    task_template,                  # TaskTemplate from load_vr_task_template(unity_scene_name)
    expected_scene_name,            # str, enforced during the setup handshake
)
```

`VRTaskDriver.__init__` performs no network I/O (`vr_task/driver.py`), so `connect()` is a separate call.

| Method / property                      | Purpose                                                                                                                                                                                           |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `connect()` / `disconnect()`           | Open / close the MQTT connection. `disconnect()` additionally exits Play Mode to stop the active Unity scene and closes the bridge HTTP client (each step best-effort)                            |
| `setup()`                              | Bridge-driven start-of-session handshake. Its final step zeroes `state.position` and rebuilds the trial decomposition. See the Setup handshake note below.                                        |
| `push_position(absolute_position)`     | Forward the animal's position to Unity as a movement delta (only emits on change)                                                                                                                 |
| `push_lick_event()`                    | Publish the generic `Interaction` trigger for the rig's interaction sensor                                                                                                                        |
| `set_reinforcing_guidance(*, enabled)` | Toggle reinforcing guidance (publishes `RequireInteraction` = `not enabled`)                                                                                                                      |
| `set_aversive_guidance(*, enabled)`    | Toggle aversive guidance (publishes `RequireWait` = `not enabled`)                                                                                                                                |
| `cycle() -> _VRTaskEvent`              | Consume the next pending Unity message and return it as a typed event                                                                                                                             |
| `resume_after_unity_restart()`         | Re-arm Unity via the bridge, re-publish both tracked guidance modes, re-fetch and re-decompose the cue sequence, zeroing `state.position` and replacing the decomposition, and clear `terminated` |
| `state` (property)                     | The current `_VRTaskState`                                                                                                                                                                        |
| `cue_sequence_distances` (property)    | Cumulative distance (cm) to complete each decomposed trial                                                                                                                                        |
| `trial_names` (property)               | The name of each decomposed trial, in sequence order                                                                                                                                              |

Every instance attribute is underscore-private, so the `state`, `cue_sequence_distances`, and `trial_names`
properties are the whole read surface (`vr_task/driver.py`).

> **Setup handshake.** `setup()` runs six bridge-driven steps in order (`VRTaskDriver.setup` in `vr_task/driver.py`).
> It requires the bridge reachable, opens the expected scene and binds the `Linear` treadmill controller, and arms
> Unity (`enter_play_mode` plus a wait for MQTT `SessionStart`). It then cross-checks the active scene name over MQTT,
> verifies the VR display, and fetches the cue sequence. Scene activation reads the actor's bound motion controller and
> rebinds it to `Linear` only when a different controller is bound, then raises `UnityBridgeError` if the actor still
> reports another controller. The only operator interaction on the failure-free path is the display check, where Unity
> animates continuously until the operator presses Enter. The driver then reads the play state, stops Play Mode when
> the scene is playing, and re-arms so the session starts from a fresh Virtual Reality origin. The caller MUST enable
> the VR screens before the call and disable them after.

> **Guidance inversion.** Unity's `RequireInteraction` / `RequireWait` flags are the inverse of guidance: a
> `True` value forces the animal to perform the behavior unaided, which is the unguided case, so enabling
> guidance publishes `value=False`. Preserve this inversion when changing guidance handling.

---

## Trial decomposition

`vr_task/trial_decomposition.py` turns Unity's flat wall-cue sequence into a trial sequence the acquisition system
can act on, using the per-trial cue motifs from the `TaskTemplate`. `decompose_cue_sequence` produces
`DecomposedTrials`, whose `trial_names` are the join key a system's experiment configuration uses to look up
per-trial structures.

For the decomposition API, the `DecomposedTrials` field table, the three raise cases, and the jit boundary, see
[`references/event-model.md`](references/event-model.md).

---

## What a consumer must supply

The package is reusable as-is, so a new acquisition system writes only the items below. Sources:
`VRTaskDriver.__init__`, `setup`, `push_lick_event`, `resume_after_unity_restart`, `_require_bridge`, `_activate_scene`,
and `_refresh_cue_sequence` in `vr_task/driver.py`, `load_vr_task_template` in `vr_task/configuration.py`, and
`run_shutdown_step` in `cross_system/shutdown_tools.py`.

| Requirement                                    | Detail                                                                                                                                                |
|------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| A `VRTaskConfiguration`                        | Broker `ip` and `port` reachable by both the runtime and Unity                                                                                        |
| A `TaskTemplate`                               | Usually through `load_vr_task_template`, which needs the templates directory set                                                                      |
| The expected Unity scene name                  | Sourced from the system's experiment configuration (`unity_scene_name`)                                                                               |
| A running Unity Editor, same machine           | The bridge is loopback-only and `setup()` blocks until it is reachable                                                                                |
| A scene carrying a `"Linear"` actor controller | `_activate_scene` raises `UnityBridgeError` otherwise                                                                                                 |
| A position source in Unity units               | The caller converts encoder counts using the template's `cm_per_unity_unit`                                                                           |
| A zeroable position source                     | `setup()` and `resume_after_unity_restart()` zero `state.position`, so the caller re-zeroes its own tracker and re-reads `trial_names`                |
| An interaction-sensor mapping                  | `push_lick_event` publishes the generic `Interaction` topic, the rig names the sensor                                                                 |
| All actuator dispatch                          | Reward, puff, brake, and emergency pause are consumer code reacting to `_VRTaskEvent`                                                                 |
| Per-trial hardware parameters                  | Joined back by `trial_names` on the consumer side                                                                                                     |
| Display power sequencing                       | The caller enables the VR screens before `setup()` and disables them after                                                                            |
| An interactive terminal operator               | `setup()` blocks on `wait_for_enter` prompts for bridge retry, Play-Mode arming retry, scene-name retry, display verification, and cue-sequence retry |
| The shutdown-isolation helper                  | `disconnect()` runs each teardown step through `run_shutdown_step`                                                                                    |

---

## Orchestrator integration

The runtime orchestrator owns the driver lifecycle. The current worked example is `mesoscope:mesoscope-vr-runtime`.

1. `__init__` constructs the `VRTaskDriver` from the nested `VRTaskConfiguration` plus the loaded `TaskTemplate`,
   only for the session type that runs the corridor task, and holds `None` otherwise. The Mesoscope-VR orchestrator
   gates on a hard-coded `session_type == SessionTypes.MESOSCOPE_EXPERIMENT` rather than reading
   `SESSION_TYPES_USING_VR_TASK`, so a new VR session type must be wired into the orchestrator gate as well as the
   registry.
2. `start()` calls `connect()`, enables the VR screens, calls `setup()`, then disables the screens. Because `setup()`
   zeroes `state.position` and rebuilds the decomposition, the orchestrator then zeroes its own distance tracker and
   re-reads `trial_names` and `cue_sequence_distances`.
3. Each runtime iteration: the data cycle calls `push_position()` and `push_lick_event()`, and the Unity cycle
   calls `cycle()` and dispatches the returned `_VRTaskEvent` (an actuator on `STIMULUS_TRIGGERED`, a brake
   pulse on `TRIGGER_DELAY_REQUESTED`, an emergency pause on `UNITY_TERMINATED`).
4. Guidance is set through `set_reinforcing_guidance()` and `set_aversive_guidance()` as trial state evolves.
5. On resume from an emergency pause, `resume_after_unity_restart()` re-arms Unity through the bridge, re-publishes both
   guidance modes, and re-fetches the cue sequence, so the operator does not press the play button. The guidance
   re-publication is required because a fresh Play Mode session reloads the interaction and wait requirements from the
   task prefab. That call zeroes `state.position` and replaces the decomposition the same way `setup()` does, so the
   orchestrator again zeroes its own distance tracker and re-reads `trial_names` and `cue_sequence_distances`.
6. `stop()` calls `disconnect()`.

---

## Known edge cases

- The `Delay` branch of `cycle()` decodes its payload without the `None` guard the `Stimulus` branch carries (both
  branches live in `VRTaskDriver.cycle`, `vr_task/driver.py`).
- `push_position` tests the truthiness of the delta, so an exactly-zero delta publishes nothing (`vr_task/driver.py`).
- `_stop_unity` drains `SessionStop` within 5 seconds, so `cycle()` never misreads a commanded exit as an unexpected
  Unity termination (`vr_task/driver.py`).
- `_arm_unity` restarts Play Mode when Unity already reports `"playing"`, because an earlier `SessionStart` never
  repeats and a fresh entry yields a clean VR origin (`vr_task/driver.py`).
- Unity's `Task.cs` disables itself mid-run on the corridor-key and segment-exhaustion bailouts, and `SessionStop`
  publishes only from `MQTTClient.OnApplicationQuit`, so the driver reads a silent stall instead of `UNITY_TERMINATED`.
  The diagnostic is the Unity Console error, which `read_console_tool` (`unity:unity-mcp-environment-setup`) returns
  with `level="error"`, and `unity:task-generator` catalogues every bailout.

---

## Extension

The VR task package is acquisition-system-agnostic, as its module docstring states (`vr_task/__init__.py`), so a new
acquisition system composes it rather than modifying it. `/library-extension` catalogues the seams that wiring
touches, covering the configuration registry, the shared `cross_system` primitives, and the MCP and CLI surfaces.

---

## Workflow: adding an MQTT topic

Adding a topic is a coordinated change with the Unity project (`sollertia-virtual-reality`).

1. Add the topic to `_VRTaskMQTTTopics` in `driver.py`, mirroring the exact wire string Unity uses.
2. If inbound and surfaced to the runtime, add it to the `monitored_topics` tuple in `VRTaskDriver.__init__`.
3. Wire it: an outbound topic gets a `push_*` or `set_*` method that sends the payload, and an inbound dispatchable
   topic gets a branch in `cycle()` returning a new `_VRTaskEvent`.
4. Coordinate the Unity-side registration, owned by `unity:mqtt-contract`.
5. Bump `sollertia-experiment` version.

---

## Workflow: adding a VR task event

1. Add a member to `VRTaskEventKind`.
2. Add any payload fields to `_VRTaskEvent`, keeping it a frozen dataclass and defaulting new fields.
3. Branch on the source topic in `cycle()` to construct and return the new event.
4. Handle the new event kind in the orchestrator's Unity cycle (see `mesoscope:mesoscope-vr-runtime`).
5. Update the [Event model](#event-model) table here and bump `sollertia-experiment` version.

---

## Maintenance contract

Update this skill when:

- `_VRTaskMQTTTopics` gains or loses a topic, or a payload shape changes.
- `VRTaskEventKind`, `_VRTaskEvent`, or `_VRTaskState` change.
- The `VRTaskDriver` public method surface or the `vr_task` package `__all__` changes.
- The `UnityBridgeClient` surface or the editor-bridge HTTP tool set changes.
- The trial-decomposition data model (`DecomposedTrials`) changes.

The VR task is a single contract spread across three libraries: the host driver (this skill), the Unity game engine, and
the shared-assets data model. The MQTT wire strings, the active scene name, the cue and trial-structure data, and the
trigger-type vocabulary are SHARED state. Changing any of them is a cross-library change that MUST land on every node in
its row below, in the same change set. Never edit one side alone, because Unity and the host silently desynchronize at
runtime.

| Contract dimension           | experiment (host)                             | unity (game engine)                                                              | assets (data model)                                    |
|------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------|--------------------------------------------------------|
| MQTT wire strings / payloads | `_VRTaskMQTTTopics` (this skill)              | `unity:mqtt-contract`, `unity:gimbl-framework` (`MQTTTopics.cs`)                 | (none)                                                 |
| Editor bridge HTTP tools     | `UnityBridgeClient` (this skill)              | `unity:play-mode`, `unity:task-scenes`, `unity:task-parameters` (`McpBridge.cs`) | (none)                                                 |
| Active scene name            | `expected_scene_name` + `SceneName` handshake | `unity:task-scenes`, `unity:task-prefabs`                                        | `assets:experiment-configuration` (`unity_scene_name`) |
| Cue catalog / trial motifs   | `decompose_cue_sequence` (this skill)         | `unity:task-generator`, `unity:task-prefabs`                                     | `assets:task-templates` (`TaskTemplate`)               |
| TriggerType / trigger zones  | `trial_names` join key (this skill)           | `unity:zone-prefabs`, `unity:task-generator`                                     | `assets:library-extension`, `assets:task-templates`    |
| Unity-side runtime bailouts  | Known edge cases (this skill)                 | `unity:task-generator` (`Task.cs` self-disable paths)                            | (none)                                                 |

When in doubt, re-read `vr_task/driver.py`, `vr_task/configuration.py`, and `vr_task/trial_decomposition.py` and
reconcile this skill against ground truth.

---

## Related skills

The `communication:microcontroller-interface` entry below resolves through the ataraxis marketplace.

| Skill                                      | Relationship                                                                 |
|--------------------------------------------|------------------------------------------------------------------------------|
| `/cli-reference`                           | Owns the `sle get unity` command surface that probes this bridge             |
| `mesoscope:mesoscope-vr-runtime`           | Owns the orchestrator that composes and drives this driver                   |
| `mesoscope:mesoscope-vr`                   | Owns the worked example's system configuration nesting `VRTaskConfiguration` |
| `mesoscope:mesoscope-vr-experiment-schema` | Owns the worked example's trigger-type-to-trial-class mapping                |
| `/acquisition-system-runtime`              | Platform-general runtime pattern this subsystem plugs into                   |
| `/library-extension`                       | Catalogues the seams that wire a new acquisition system to this package      |
| `/system-health-check`                     | Pre-flight `check_unity_bridge_tool` that enforces the Unity Editor is open  |
| `communication:microcontroller-interface`  | `MQTTCommunication` mechanics the driver builds on                           |
| `unity:play-mode`                          | Unity-side editor bridge Play-Mode control the driver drives                 |
| `unity:task-scenes`                        | Unity-side owner of the scene listing and opening tools the driver calls     |
| `unity:task-parameters`                    | Unity-side Task Parameters surface the controller rebind reads and writes    |
| `unity:mqtt-contract`                      | Unity side of the MQTT topic contract (`MQTTTopics`)                         |
| `unity:gimbl-framework`                    | Unity-side VR framework and game objects                                     |
| `unity:task-prefabs`                       | Unity task prefab generation from templates                                  |
| `assets:task-templates`                    | Authors the mandatory corridor task asset (`TaskTemplate`) decomposed here   |
| `assets:working-directory`                 | Owns the task-templates directory `load_vr_task_template` resolves           |
| `assets:library-extension`                 | Owns the `TriggerType` enum in `sollertia-shared-assets`                     |
| `assets:experiment-configuration`          | Owns `unity_scene_name` (verified by `setup()`) and the per-trial parameters |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Contract fidelity:
- [ ] _VRTaskMQTTTopics wire strings match the Unity MQTTTopics constant set exactly (unity:mqtt-contract)
- [ ] Bridge tool names and reply keys match McpBridge.cs (unity:play-mode, unity:task-scenes, unity:task-parameters)
- [ ] monitored_topics updated for any new inbound surfaced topic
- [ ] Event model and topic contract tables in this skill updated
- [ ] Module constants table matches the module constants in vr_task/driver.py
- [ ] Scene activation binds the Linear treadmill controller and errors when the actor reports a different one
- [ ] Bridge stays mandatory and loopback-only (no enable/disable config, no manual-prompt fallback)
- [ ] cycle() branches and VRTaskEventKind/_VRTaskEvent updated for any new dispatchable event
- [ ] Guidance inversion preserved (RequireInteraction/RequireWait publish `not enabled`)
- [ ] Trial decomposition (DecomposedTrials) updated if the per-trial data model changed

Cross-repository coordination:
- [ ] Orchestrator Unity cycle handles any new event kind (mesoscope:mesoscope-vr-runtime)
- [ ] sollertia-experiment version bumped, Unity side coordinated for any wire-string change
```
