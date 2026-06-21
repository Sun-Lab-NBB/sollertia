---
name: vr-driver-interface
description: >-
  Documents the Virtual Reality task driver subsystem: the VRTaskDriver class and its
  configuration, the MQTT topic contract with the Unity game engine, the editor MCP Bridge it drives
  for scene activation and Play Mode control, the per-cycle VRTaskEvent model, and the cue-sequence
  trial decomposition. Use when modifying Unity coupling, the editor bridge, adding an MQTT topic or
  VR task event, or wiring a new acquisition system to Unity.
user-invocable: false
---

# VR task driver interface

Documents the Virtual Reality task driver — the host-side hardware subsystem that couples a Sollertia
acquisition runtime to the Unity game engine implemented in `sollertia-virtual-reality`. This is the
platform-general VR subsystem, parallel to `/microcontroller-interface` (microcontrollers) and
`/zaber-interface` (motors): the `VRTaskDriver` in
`sollertia_experiment/vr_task/driver.py` is hardware-agnostic and composed by an acquisition
system's runtime orchestrator (currently Mesoscope-VR's `MesoscopeVRSystem`).

The VR task driver is a standard subsystem of every acquisition system, but it is built only for the
session types that run the linear infinite corridor task: experiment sessions. The runtime orchestrator
constructs it only for those session types (for Mesoscope-VR's `MesoscopeVRSystem`, only
`MESOSCOPE_EXPERIMENT` sessions); `self._vr_task` is `None` for training and window-checking sessions,
which run no corridor task (see `/acquisition-system-runtime`).

The Unity side of the contract — the GIMBL framework, the `MQTTTopics` constant set, and task prefab
generation — lives in the unity plugin. This skill owns the **host (Python) side**.

---

## Scope

**Covers:**
- `VRTaskConfiguration` — the MQTT broker discovery fields the driver reads
- `VRTaskDriver` — connection lifecycle, bridge-driven setup handshake, per-cycle pump, guidance toggles
- `UnityBridgeClient` — the editor MCP Bridge HTTP client the driver uses to open scenes and control Play Mode
- The `_VRTaskMQTTTopics` contract — every topic, its payload shape, and its direction
- `VRTaskEventKind` / `VRTaskEvent` — the typed events `cycle()` surfaces to the runtime
- `VRTaskState` — the single source of truth shared between setup and per-cycle events
- Cue-sequence trial decomposition (`DecomposedTrials`, `CachedMotifDecomposer`, `decompose_cue_sequence`)
- How the runtime orchestrator composes and drives the VRTaskDriver
- Workflows for adding an MQTT topic or a VR task event

**Does not cover** (delegated):
- The Unity-side framework and game objects — see `unity:gimbl-framework`
- The Unity-side editor MCP Bridge endpoint (`McpBridge.cs`) and scene / Play-Mode authoring — see
  `unity:play-mode`, `unity:scene-setup`
- The Unity-side MQTT topic registration / `MQTTTopics` constant set — see `unity:mqtt-contract`
- Unity task prefab / scene generation from templates — see `unity:task-prefabs`, `unity:task-scenes`
- `TaskTemplate` authoring (cue catalog, corridor geometry, per-trial cue motifs, trigger types) —
  owned by `assets:task-templates`
- The runtime state machine that consumes the driver's events — see `mesoscope:mesoscope-vr-runtime`
- The platform-general runtime/orchestrator pattern — see `/acquisition-system-runtime`

---

## Authoritative bases

| Concern                                                              | Authority                                          |
|----------------------------------------------------------------------|----------------------------------------------------|
| `MQTTCommunication` mechanics (connect, monitored topics, get_data)  | `ataraxis@communication:microcontroller-interface` |
| Unity-side editor MCP Bridge (scene / Play-Mode tools)               | `unity:play-mode`, `unity:scene-setup`             |
| Unity-side MQTT topic contract (`MQTTTopics`)                        | `unity:mqtt-contract`                              |
| Unity-side VR framework and game objects                             | `unity:gimbl-framework`                            |
| `TaskTemplate` schema (cue catalog, geometry, motifs, trigger types) | `assets:task-templates`                            |
| Runtime that consumes this driver                                    | `mesoscope:mesoscope-vr-runtime`                   |

The driver builds on `ataraxis_communication_interface.MQTTCommunication` and only documents the
Sollertia VR contract layered on top.

---

## Configuration

The driver reads `VRTaskConfiguration` (`sollertia_experiment/vr_task/configuration.py`):

| Field  | Type  | Default       | Purpose                                                           |
|--------|-------|---------------|-------------------------------------------------------------------|
| `ip`   | `str` | `"127.0.0.1"` | IP address of the MQTT broker used to reach the Unity game engine |
| `port` | `int` | `1883`        | Port number of the MQTT broker                                    |

`VRTaskConfiguration` stores **only** the MQTT broker discovery fields. For Mesoscope-VR it is nested
under `MesoscopeSystemConfiguration.assets.vr_task` (see `mesoscope:mesoscope-vr`). The geometric VR
parameters (cue catalog, corridor geometry, cm-per-Unity-unit, per-trial cue motifs, trigger types)
are NOT stored here — they live in the `TaskTemplate` resolved at experiment start by
`load_vr_task_template(unity_scene_name)`, which reads from the shared VR task templates directory.

Scene activation and Play Mode are driven over the editor MCP Bridge (see the next section), whose
endpoint is a fixed loopback constant (`127.0.0.1:8090`) in `bridge.py` — deliberately NOT a
`VRTaskConfiguration` field. The bridge binds the loopback interface only, so the acquisition host
must run on the same machine as the Unity Editor.

---

## MQTT topic contract

`_VRTaskMQTTTopics` (`StrEnum` in `driver.py`) mirrors the flat PascalCase `MQTTTopics` constant set
published by `sollertia-virtual-reality` (see `unity:mqtt-contract`). Both sides MUST agree on these
strings exactly.

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

The driver subscribes to the inbound subset it surfaces or resolves internally
(`CUE_SEQUENCE`, `SESSION_STOP`, `SESSION_START`, `SCENE_NAME`, `STIMULUS`, `DELAY`) when constructing
its `MQTTCommunication`.

---

## Unity Editor MCP Bridge

Alongside the MQTT data channel, the driver drives the Unity Editor over the **editor MCP Bridge** — the
HTTP listener `McpBridge.cs` starts automatically inside the Unity Editor (Unity side: `unity:play-mode`,
`unity:scene-setup`). `UnityBridgeClient` (`sollertia_experiment/vr_task/bridge.py`) is the host-side
client. It is **mandatory and always on**: there is no enable/disable configuration field, and the driver
does NOT fall back to manual scene / Play-Mode prompts. The bridge binds the loopback interface only, so
the acquisition host MUST be the same machine as the Unity Editor; the endpoint is a fixed module constant
(`127.0.0.1:8090`), not configuration.

Wire protocol: POST JSON `{"tool": ..., "args": {...}}`; the reply always carries `success: bool`, and a
failure carries an `error` string. `UnityBridgeClient` maps a transport failure or a `success: false`
reply to `UnityBridgeError`, which the driver layer catches to retry, surface, or probe reachability.

| Method                     | Bridge tool            | Purpose                                                        |
|----------------------------|------------------------|----------------------------------------------------------------|
| `list_scenes()`            | `list_scenes`          | All project scene paths and the active scene path              |
| `open_scene(path, policy)` | `open_scene`           | Open a scene (driver passes the `save` unsaved-changes policy) |
| `enter_play_mode()`        | `enter_play_mode`      | Enter Play Mode (arm); async, returns the post-request state   |
| `exit_play_mode()`         | `exit_play_mode`       | Exit Play Mode (disarm)                                        |
| `get_play_state()`         | `get_play_state`       | The editor play state and active scene name                    |
| `resolve_scene_path(name)` | (via `list_scenes`)    | Resolve `expected_scene_name` to a project scene path by stem  |
| `is_reachable()`           | (via `get_play_state`) | True when the bridge responds; never raises                    |
| `describe_status()`        | (via `get_play_state`) | One-line reachability / scene / state summary                  |

`enter_play_mode()` only presses the editor's play button; the MQTT `SessionStart` message remains the
authoritative "Unity is armed and connected" signal, so `setup()` still waits for it after arming. A
dedicated reachability check is surfaced to pre-flight via the `sle get unity` CLI command and the
`check_unity_bridge_tool` MCP tool (see `/system-health-check`,
`/acquisition-system-setup`).

---

## Event model

`cycle()` consumes **at most one** MQTT message per call and returns a typed `VRTaskEvent`. The
asynchronous Unity messages it surfaces are enumerated by `VRTaskEventKind` (`IntEnum`):

| Kind                      | Value | Source topic                        | Meaning / caller action                                              |
|---------------------------|-------|-------------------------------------|----------------------------------------------------------------------|
| `NONE`                    | 0     | (buffer empty or a handshake topic) | No dispatchable event this cycle                                     |
| `STIMULUS_TRIGGERED`      | 1     | `STIMULUS`                          | Trial resolved; `delivered` drives reward/puff, `cause` marks guided |
| `TRIGGER_DELAY_REQUESTED` | 2     | `DELAY`                             | Unity requests a brake pulse of `delay_ms` milliseconds              |
| `UNITY_TERMINATED`        | 3     | `SESSION_STOP`                      | Unity runtime ended; the system must enter an emergency pause        |

`VRTaskEvent` (frozen dataclass) carries `kind: VRTaskEventKind`, `delay_ms: int = 0` (populated only for
`TRIGGER_DELAY_REQUESTED`), and — populated only for `STIMULUS_TRIGGERED` and parsed from the `Stimulus` payload —
`trial_name: str`, `delivered: bool = True`, and `cause: StimulusCause = BEHAVIOR` (a `StrEnum` of `BEHAVIOR`/`GUIDANCE`
exported from the `vr_task` package). The runtime resolves the per-trial outcome from `delivered` and `cause`. Handshake
topics consumed during `cycle()` (`SESSION_START`, `SCENE_NAME`, `CUE_SEQUENCE`) resolve to `NONE` — they are handled
by the setup sequence, not dispatched.

`VRTaskState` (dataclass, the driver's `state` property) is the single source of truth shared between
the setup handshake and per-cycle events:

| Field                          | Type             | Purpose                                        |
|--------------------------------|------------------|------------------------------------------------|
| `position`                     | `np.float64`     | Current absolute animal position (Unity units) |
| `cue_sequence`                 | `NDArray[uint8]` | The session's flattened wall-cue sequence      |
| `terminated`                   | `bool`           | Whether Unity has unexpectedly terminated      |
| `reinforcing_guidance_enabled` | `bool`           | Reinforcing-trial guidance state               |
| `aversive_guidance_enabled`    | `bool`           | Aversive-trial guidance state                  |

---

## VRTaskDriver API

Construction (built only for experiment sessions that drive Unity):

```python
VRTaskDriver(
    configuration,                  # VRTaskConfiguration (assets.vr_task)
    *,
    task_template,                  # TaskTemplate from load_vr_task_template(unity_scene_name)
    expected_scene_name,            # str; enforced during the setup handshake
)
```

| Method / property                      | Purpose                                                                           |
|----------------------------------------|-----------------------------------------------------------------------------------|
| `connect()` / `disconnect()`           | Open / close the MQTT connection (`disconnect()` also closes the bridge client)   |
| `setup()`                              | Bridge-driven start-of-session handshake; see the Setup handshake note below.     |
| `push_position(absolute_position)`     | Forward the animal's position to Unity as a movement delta (only emits on change) |
| `push_lick_event()`                    | Notify Unity that the animal licked                                               |
| `set_reinforcing_guidance(*, enabled)` | Toggle reinforcing guidance (publishes `RequireInteraction` = `not enabled`)      |
| `set_aversive_guidance(*, enabled)`    | Toggle aversive guidance (publishes `RequireWait` = `not enabled`)                |
| `cycle() -> VRTaskEvent`               | Consume the next pending Unity message and return it as a typed event             |
| `resume_after_unity_restart()`         | Re-arm Unity via the bridge, re-fetch the cue sequence, and clear `terminated`    |
| `state` (property)                     | The current `VRTaskState`                                                         |
| `cue_sequence_distances` (property)    | Cumulative distance (cm) to complete each decomposed trial                        |
| `trial_names` (property)               | The name of each decomposed trial, in sequence order                              |

> **Setup handshake.** `setup()` is bridge-driven: require bridge reachable → open the expected scene →
> arm Unity (`enter_play_mode` + wait for MQTT `SessionStart`) → cross-check the active scene name over
> MQTT → VR display verification → cue-sequence fetch. There are no "press play" prompts. The only operator
> interaction is the display check: Unity animates continuously until the operator presses Enter, the driver
> stops Play Mode, and the operator confirms the render (yes advances and re-arms; no lets them adjust, then
> re-arms and repeats). The caller MUST enable the VR screens before the call and disable them after.

> **Guidance inversion.** Unity's `RequireInteraction` / `RequireWait` flags are the inverse of guidance: a
> `True` value forces the animal to perform the behavior unaided (the unguided case), so enabling
> guidance publishes `value=False`. Preserve this inversion when changing guidance handling.

---

## Trial decomposition

`sollertia_experiment/vr_task/trial_decomposition.py` turns Unity's flat wall-cue sequence into a
trial sequence the acquisition system can act on, using the per-trial cue motifs from the
`TaskTemplate`.

- `decompose_cue_sequence(...)` — matches the flat cue array against the template's per-trial motifs
  and produces `DecomposedTrials`.
- `CachedMotifDecomposer` — caches the flattened motif data (numba-accelerated) between successive
  decomposition runs so re-decomposition after a Unity restart is cheap.
- `DecomposedTrials` (frozen dataclass) — aligned per-trial sequences (index `i` = the i-th trial):

  | Field                  | Type               | Purpose                                                                                   |
  |------------------------|--------------------|-------------------------------------------------------------------------------------------|
  | `cumulative_distances` | `NDArray[float64]` | Cumulative distance (cm) to reach the end of each trial                                   |
  | `trial_names`          | `tuple[str, ...]`  | Join key the runtime uses to look up per-trial parameters in its experiment configuration |

The driver's `trial_names` are the join key the system's experiment configuration uses to look up per-trial
structures; for the TriggerType taxonomy see `assets:experiment-configuration` / `assets:library-extension`,
and for how a system maps trigger types to runtime trial classes see
`mesoscope:mesoscope-vr-experiment-schema` (the `TriggerType` enum lives in `sollertia-shared-assets`).

e.g. Mesoscope-VR maps `INTERACTION`->reward and `OCCUPANCY_DISARM`->gas-puff; for the full per-trigger
mapping see `mesoscope:mesoscope-vr-experiment-schema`.

The orchestrator reads the driver's `trial_names` — joining them against its experiment configuration's trial
structures to build the per-trial outcome arrays (e.g. Mesoscope-VR's reward/puff arrays — see
`mesoscope:mesoscope-vr-experiment-schema`) — along with `cue_sequence_distances` and `state.cue_sequence`.
All trial modes share one MQTT/wire contract: every mode publishes the same `Stimulus` event and
adds no topics (see the [MQTT topic contract](#mqtt-topic-contract)); the Unity-side dispatch, prefab reuse,
and mode-aware template geometry are owned by `unity:zone-prefabs`, `unity:task-generator`, and
`assets:task-templates`.

---

## Orchestrator integration

The runtime orchestrator owns the driver lifecycle (see `mesoscope:mesoscope-vr-runtime`):

1. `__init__` constructs the `VRTaskDriver` from `assets.vr_task` + the loaded `TaskTemplate`, **only**
   for `SessionTypes.MESOSCOPE_EXPERIMENT` sessions (`self._vr_task` is `None` otherwise).
2. `start()` calls `connect()`, enables the VR screens, calls `setup()`, then disables the screens.
3. Each runtime iteration: `_data_cycle()` calls `push_position()` / `push_lick_event()`; `_unity_cycle()`
   calls `cycle()` and dispatches the returned `VRTaskEvent` (reward on `STIMULUS_TRIGGERED`, brake
   pulse on `TRIGGER_DELAY_REQUESTED`, emergency pause on `UNITY_TERMINATED`).
4. Guidance is set via `set_reinforcing_guidance()` / `set_aversive_guidance()` as trial state evolves.
5. On resume from an emergency pause, `resume_after_unity_restart()` re-arms Unity through the bridge and
   re-fetches the cue sequence — the operator does not press the play button.
6. `stop()` calls `disconnect()`.

---

## Workflow: adding an MQTT topic

Adding a topic is a coordinated change with the Unity project (`sollertia-virtual-reality`).

1. Add the topic to `_VRTaskMQTTTopics` in `driver.py`, mirroring the exact wire string Unity uses.
2. If inbound and surfaced to the runtime, add it to the `monitored_topics` tuple in `VRTaskDriver.__init__`.
3. Wire it: for an outbound topic, add a `push_*` / `set_*` method that `send_data`s the payload; for
   an inbound dispatchable topic, branch on it in `cycle()` and return a new `VRTaskEvent`.
4. Coordinate the Unity-side registration — see `unity:mqtt-contract`.
5. Bump `sollertia-experiment` version.

---

## Workflow: adding a VR task event

1. Add a member to `VRTaskEventKind`.
2. Add any payload fields to `VRTaskEvent` (keep it a frozen dataclass; default new fields).
3. Branch on the source topic in `cycle()` to construct and return the new event.
4. Handle the new event kind in the orchestrator's `_unity_cycle()` (see `mesoscope:mesoscope-vr-runtime`).
5. Update the [Event model](#event-model) table here and bump `sollertia-experiment` version.

---

## Maintenance contract

Update this skill when:

- `_VRTaskMQTTTopics` gains/loses a topic or a payload shape changes.
- `VRTaskEventKind` / `VRTaskEvent` / `VRTaskState` change.
- The `VRTaskDriver` public method surface changes.
- The `UnityBridgeClient` surface or the editor-bridge HTTP tool set changes.
- The trial-decomposition data model (`DecomposedTrials`) changes.

The VR task is a single contract spread across three libraries — the host driver (this skill), the Unity
game engine, and the shared-assets data model. The MQTT wire strings, the active scene name, the cue /
trial-structure data, and the trigger-type vocabulary are SHARED state: changing any of them is a
cross-library change that MUST land on every node in its row below, in the same change set. Never edit one
side alone — Unity and the host silently desynchronize at runtime.

| Contract dimension           | experiment (host)                             | unity (game engine)                                              | assets (data model)                                    |
|------------------------------|-----------------------------------------------|------------------------------------------------------------------|--------------------------------------------------------|
| MQTT wire strings / payloads | `_VRTaskMQTTTopics` (this skill)              | `unity:mqtt-contract`, `unity:gimbl-framework` (`MQTTTopics.cs`) | —                                                      |
| Editor bridge HTTP tools     | `UnityBridgeClient` (this skill)              | `unity:play-mode`, `unity:scene-setup` (`McpBridge.cs`)          | —                                                      |
| Active scene name            | `expected_scene_name` + `SceneName` handshake | `unity:task-scenes`, `unity:task-prefabs`                        | `assets:experiment-configuration` (`unity_scene_name`) |
| Cue catalog / trial motifs   | `decompose_cue_sequence` (this skill)         | `unity:task-generator`, `unity:task-prefabs`                     | `assets:task-templates` (`TaskTemplate`)               |
| TriggerType / trigger zones  | `trial_names` join key (this skill)           | `unity:zone-prefabs`, `unity:task-generator`                     | `assets:library-extension`, `assets:task-templates`    |

A wire-string, scene-name, cue-schema, or trigger-type change is a contract break that MUST be reconciled
across every node in the matching row above — on both/all sides at once.

When in doubt, re-read `sollertia_experiment/vr_task/driver.py`,
`sollertia_experiment/vr_task/configuration.py`, and
`sollertia_experiment/vr_task/trial_decomposition.py` and reconcile this skill against ground truth.

---

## Related skills

| Skill                                              | Relationship                                                                 |
|----------------------------------------------------|------------------------------------------------------------------------------|
| `mesoscope:mesoscope-vr-runtime`                   | Owns the orchestrator that composes and drives this driver                   |
| `mesoscope:mesoscope-vr`                           | Defines `assets.vr_task` (`VRTaskConfiguration`) in the system config        |
| `/acquisition-system-runtime`                      | Platform-general runtime pattern this subsystem plugs into                   |
| `ataraxis@communication:microcontroller-interface` | `MQTTCommunication` mechanics the driver builds on                           |
| `/system-health-check`                             | Pre-flight `check_unity_bridge_tool` that enforces the Unity Editor is open  |
| `unity:play-mode`                                  | Unity-side editor bridge Play-Mode control the driver drives                 |
| `unity:scene-setup`                                | Unity-side editor bridge scene activation the driver drives                  |
| `unity:mqtt-contract`                              | Unity side of the MQTT topic contract (`MQTTTopics`)                         |
| `unity:gimbl-framework`                            | Unity-side VR framework and game objects                                     |
| `unity:task-prefabs`                               | Unity task prefab generation from templates                                  |
| `assets:task-templates`                            | Authors the mandatory corridor task asset (`TaskTemplate`) decomposed here   |
| `assets:library-extension`                         | Owns the `TriggerType` enum (in `sollertia-shared-assets`)                   |
| `assets:experiment-configuration`                  | Owns `unity_scene_name` (verified by `setup()`) and the per-trial parameters |

---

## Verification checklist

```text
When modifying the VR task driver:
- [ ] _VRTaskMQTTTopics wire strings match the Unity MQTTTopics constant set exactly (unity:mqtt-contract)
- [ ] UnityBridgeClient tool names and response keys match McpBridge.cs (unity:play-mode, unity:scene-setup)
- [ ] Bridge stays mandatory and loopback-only (no enable/disable config, no manual-prompt fallback)
- [ ] monitored_topics updated for any new inbound surfaced topic
- [ ] cycle() branches and VRTaskEventKind/VRTaskEvent updated for any new dispatchable event
- [ ] Guidance inversion preserved (RequireInteraction/RequireWait publish `not enabled`)
- [ ] Orchestrator _unity_cycle() handles any new event kind (mesoscope:mesoscope-vr-runtime)
- [ ] Trial decomposition (DecomposedTrials) updated if the per-trial data model changed
- [ ] Event model / topic contract tables in this skill updated
- [ ] sollertia-experiment version bumped; Unity side coordinated for any wire-string change
```
