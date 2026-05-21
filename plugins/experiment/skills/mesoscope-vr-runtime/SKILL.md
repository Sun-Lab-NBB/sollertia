---
name: mesoscope-vr-runtime
description: >-
  Documents the Mesoscope-VR runtime behavior layer: the _MesoscopeVRStates state machine, the
  _MesoscopeVRSystem orchestrator, MQTT topics for Unity-VR coupling, the per-mode runtime logic
  functions, the BehaviorVisualizer and its modes, and the sle CLI commands. Use when adding a new
  training mode or session type, extending the state machine, modifying visualizers, or wiring new
  CLI commands.
user-invocable: true
---

# Mesoscope-VR runtime

Documents the Mesoscope-VR runtime behavior layer — the state machine, the orchestrator class, the
per-mode runtime logic functions, the visualizer, and the CLI surface. This skill is the
counterpart to `experiment:mesoscope-vr` (hardware composition); together they cover the full
Mesoscope-VR system.

For the platform-general acquisition-system design pattern, see `experiment:acquisition-system-design`.
For the per-firmware-module wrapper layer, see `experiment:microcontroller-interface`.

---

## Scope

**Covers:**
- `_MesoscopeVRStates` enumeration and state machine semantics
- `_MesoscopeVRSystem` orchestrator class — state transitions and binding-class composition
- Mesoscope-VR-specific MQTT topics for Unity-VR runtime coupling
- Per-mode runtime logic functions (`lick_training_logic`, `run_training_logic`,
  `experiment_logic`, `window_checking_logic`, `maintenance_logic`)
- `BehaviorVisualizer` and `VisualizerMode` enumeration
- Session-descriptor consumption pattern (read by runtime, written by assets-plugin tooling)
- `sle` CLI command surface (`session`, `run`, training/experiment subcommands)
- Workflow for adding a new training mode (cross-repo: sollertia-shared-assets + sollertia-experiment)

**Does not cover** (delegated):
- Mesoscope-VR hardware composition (binding classes, calibration dataclasses, system YAML) — see `experiment:mesoscope-vr`
- The platform-general acquisition-system pattern — see `experiment:acquisition-system-design`
- Per-firmware-module wrappers and slmc Module classes — see `experiment:microcontroller-interface`
- Session descriptors and `SessionTypes` enum — owned by the assets plugin's `/session-descriptors`
- Task template and trial-structure authoring — owned by the assets plugin's `/task-templates`
- Per-experiment configuration — owned by the assets plugin's `/experiment-configuration`
- Session-data lifecycle (preprocessing, transfer, deletion) — owned by `experiment:data-management`

---

## Runtime layer architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  sle CLI (sollertia_experiment/command_line_interfaces/execute.py)           │
│  ───────────────────────────────────────────────────────────────             │
│  sle session lick-training   ──┐                                             │
│  sle session run-training    ──┤                                             │
│  sle session run-experiment  ──┤                                             │
│  sle session check-window    ──┼──► reads session config, instantiates       │
│  sle maintain                ──┘    descriptor, calls per-mode runtime logic │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  Per-mode runtime logic functions                                            │
│  (sollertia_experiment/mesoscope_vr/data_acquisition.py)                     │
│  ─────────────────────────────────────────────────                           │
│  lick_training_logic(session_data, descriptor)                               │
│  run_training_logic(session_data, descriptor)                                │
│  experiment_logic(session_data, descriptor, ...)                             │
│  window_checking_logic(session_data, descriptor)                             │
│  maintenance_logic()                                                         │
│                                                                              │
│  Each constructs a _MesoscopeVRSystem, drives its state transitions, and     │
│  invokes binding-class methods over the session's runtime.                   │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │ composes
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  _MesoscopeVRSystem orchestrator                                             │
│  ────────────────────────────────                                            │
│  - Composes ZaberMotors, MicroControllerInterfaces, VideoSystems             │
│  - Owns the _MesoscopeVRStates state machine                                 │
│  - Owns MQTTCommunication for Unity coupling                                 │
│  - Drives state transitions (change_runtime_state, set_<state>, etc.)        │
│  - Forwards real-time data to BehaviorVisualizer                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Authoritative bases

| Concern                                           | Authority                                              |
|---------------------------------------------------|--------------------------------------------------------|
| Mesoscope-VR hardware composition                 | `experiment:mesoscope-vr`                              |
| Per-firmware-module wrapper API                   | `experiment:microcontroller-interface`                 |
| `SessionTypes` enum                               | assets plugin `/session-descriptors`                   |
| Session descriptor dataclass authoring            | assets plugin `/session-descriptors`                   |
| Task template authoring (trial structure)         | assets plugin `/task-templates`                        |
| Experiment configuration authoring                | assets plugin `/experiment-configuration`              |
| Session-data lifecycle                            | `experiment:data-management`                           |

This skill documents how the runtime *consumes* descriptors, session data, and task templates. It
does NOT document how to author them.

---

## State machine

`_MesoscopeVRStates` (in `sollertia_experiment/mesoscope_vr/data_acquisition.py`) is the enum
that encodes the runtime's current hardware-control mode. Each state determines the brake state,
screen state, lickport state, and (during experiments) the trial-guidance mode.

Current states:

| State           | Value | Meaning                                                                          |
|-----------------|-------|----------------------------------------------------------------------------------|
| `IDLE`          | 0     | Not conducting a session (between-session default)                               |
| `REST`          | 1     | Rest period of an experiment session (brake engaged, screens off)                |
| `RUN`           | 2     | Run period of an experiment session (brake disengaged, screens on)               |
| `LICK_TRAINING` | 3     | Lick training session (lickport-only behavior)                                    |
| `RUN_TRAINING`  | 4     | Run training session (wheel-only behavior)                                        |

The enum exposes `to_dict()` for the visualizer (which displays the current state's human-readable
name in the title bar).

State values 0-4 are taken. New states SHOULD use the next unused value (5) unless a
non-contiguous code carries semantic meaning that justifies the gap.

---

## `_MesoscopeVRSystem` orchestrator

`_MesoscopeVRSystem` (line 762 of `data_acquisition.py`) is the central orchestrator. It is named
with a leading underscore because it is internal to the runtime — external consumers go through
the per-mode runtime logic functions, not the system class directly.

### Construction

The orchestrator's `__init__` builds the binding classes in the order documented in
`experiment:acquisition-system-design`'s [Construction order](../acquisition-system-design/SKILL.md#construction-order):

1. `DataLogger.start()` (called by the runtime logic function before constructing the orchestrator)
2. `MQTTCommunication(...)` for Unity coupling
3. `MicroControllerInterfaces(data_logger, microcontroller_configuration)`
4. `VideoSystems(data_logger, camera_configuration, output_directory)`
5. `ZaberMotors(zaber_positions, zaber_configuration)`

After construction the orchestrator owns:

- `self._microcontrollers: MicroControllerInterfaces`
- `self._cameras: VideoSystems`
- `self._zaber: ZaberMotors`
- `self._mqtt: MQTTCommunication`
- `self._trial_state: _TrialState` (per-session trial tracking)
- `self._unity_state: _UnityState` (per-session Unity-coupling state)
- `self._visualizer: BehaviorVisualizer | None` (created lazily by runtime logic functions)

### State transitions

The orchestrator exposes per-state methods that drive the hardware into the target state:

```python
def set_idle_state(self) -> None: ...
def set_rest_state(self) -> None: ...
def set_run_state(self) -> None: ...
def set_lick_training_state(self) -> None: ...
def set_run_training_state(self) -> None: ...
```

Each method:

1. Updates `self._current_state` to the new `_MesoscopeVRStates` value.
2. Sends the appropriate commands to the binding classes (`self._microcontrollers.brake.set_state(...)`,
   `self._microcontrollers.screens.set_state(...)`, etc.).
3. Logs the state change via DataLogger (event code `_MesoscopeVRLogMessageCodes.SYSTEM_STATE`).
4. Updates the visualizer's mode (`self._visualizer.mode = ...`).

State transitions are idempotent — setting the current state is a no-op.

### MQTT topics

Mesoscope-VR uses MQTT to coordinate with the Unity VR game engine. The topic vocabulary is
defined as `_MesoscopeVRMQTTTopics` (StrEnum):

| Category               | Topics                                                                                                          |
|------------------------|-----------------------------------------------------------------------------------------------------------------|
| Session lifecycle      | `Gimbl/Session/Start` (`UNITY_STARTUP`), `Gimbl/Session/Stop` (`UNITY_TERMINATION`)                              |
| Cue sequence           | `CueSequence/` (`CUE_SEQUENCE`), `CueSequenceTrigger/` (`CUE_SEQUENCE_REQUEST`)                                  |
| Trial guidance         | `RequireLick/True/`, `RequireLick/False/`, `RequireWait/True/`, `RequireWait/False/`, `VisibleMarker/True/`, `VisibleMarker/False/` |
| Scene introspection    | `SceneNameTrigger/` (`UNITY_SCENE_REQUEST`), `SceneName/` (`UNITY_SCENE`)                                        |
| Animal events          | `Gimbl/Stimulus/` (`STIMULUS`), `LickPort/` (`LICK_EVENT`), `LinearTreadmill/Data` (`ENCODER_DATA`)              |
| Aversive triggers      | `Gimbl/TriggerDelay/` (`TRIGGER_DELAY`)                                                                          |

The orchestrator subscribes to inbound topics (cue sequences, scene names, stimulus events) and
publishes outbound topics (session start/stop, guidance state, motion updates). Topic semantics
are owned by the Unity-side Gimbl framework; see `unity:gimbl-framework` and `unity:mqtt-contract`
for the Unity side of the contract.

### Trial-state tracking

`_TrialState` (line 625 of `data_acquisition.py`) consolidates all per-trial tracking attributes
used during experiment runtimes. It tracks both reinforcing (water reward) and aversive (gas puff)
trial types, including:

- Completed trial count and cumulative distance per trial
- Per-type guided-trial counts (`reinforcing_guided_trials`, `aversive_guided_trials`)
- Per-type recovery thresholds (engage guidance after N consecutive failures)
- Per-type "rewarded" flags for the in-flight trial

Adding new trial-tracking dimensions (e.g., a new stimulus modality) requires extending
`_TrialState` and updating the orchestrator's stimulus handling in the experiment runtime logic.

### Log message codes

`_MesoscopeVRLogMessageCodes` (IntEnum) defines event codes the orchestrator emits to the
DataLogger:

| Code | Name                          | Meaning                                                       |
|------|-------------------------------|---------------------------------------------------------------|
| 1    | `SYSTEM_STATE`                | System has changed configuration state                        |
| 2    | `RUNTIME_STATE`               | Acquired session has changed runtime state                    |
| 3    | `REINFORCING_GUIDANCE_STATE`  | Reinforcing-trial guidance state changed                       |
| 4    | `AVERSIVE_GUIDANCE_STATE`     | Aversive-trial guidance state changed                          |
| 5    | `DISTANCE_SNAPSHOT`           | Distance snapshot taken (e.g., on cue sequence change)        |

New events get the next unused code (currently 6). These codes are consumed by downstream behavior
processing in the forging plugin's behavior pipeline.

---

## Per-mode runtime logic functions

Each runtime mode has a top-level function in `data_acquisition.py` that:

1. Validates inputs and prepares output directories.
2. Starts `DataLogger`.
3. Constructs `_MesoscopeVRSystem`.
4. Loads the session-specific descriptor (read from the per-session YAML written by the assets plugin).
5. Drives state transitions, trial loops, and CLI prompts.
6. Tears down in the reverse order on completion or error.

Current functions:

| Function                  | Location | Purpose                                                                              |
|---------------------------|----------|--------------------------------------------------------------------------------------|
| `lick_training_logic`     | line 2773 | Lickport-only training (animal learns to operate the lickport for water rewards)   |
| `run_training_logic`      | line 3068 | Wheel-only training (animal learns to run for water rewards)                       |
| `experiment_logic`        | line 3508 | Full Mesoscope-VR experiment session with trial structure                          |
| `window_checking_logic`   | line 2599 | Cranial window maintenance session (no behavior)                                    |
| `maintenance_logic`       | line 3782 | Hardware maintenance (valve calibration, motor positioning, brake testing)         |

Each function takes a `SessionData` (file-paths metadata) and a session-specific descriptor (the
mode's runtime parameters loaded from YAML). The descriptor is the per-session parameter envelope —
see [Session descriptor consumption](#session-descriptor-consumption).

### Session descriptor consumption

Descriptors live in `sollertia-shared-assets`
(`sollertia_shared_assets/data_classes/runtime_data.py`). The runtime treats them as read-only:

- The CLI command instantiates the descriptor with default values.
- Per-CLI-flag overrides are applied to the in-memory descriptor.
- The descriptor is written to the session's directory before the runtime begins
  (see `assets:session-descriptors`).
- The runtime logic function reads the descriptor from disk at session start and uses it to
  parameterize the state machine.

The descriptor is NEVER mutated by the runtime — runtime-discovered values (e.g., total water
delivered, total trials completed) are recorded in the DataLogger archive, not back into the
descriptor file. The descriptor is the "session plan"; the DataLogger archive is the "session
result."

---

## BehaviorVisualizer

`BehaviorVisualizer` (in `sollertia_experiment/mesoscope_vr/visualizers.py`) renders real-time
behavior data using matplotlib with the QT backend. It is constructed lazily by the runtime logic
functions and updated at a fixed cadence as the session progresses.

### VisualizerMode

The visualizer's display mode is set by `VisualizerMode` (IntEnum):

| Mode            | Value | Panels displayed                                       |
|-----------------|-------|--------------------------------------------------------|
| `LICK_TRAINING` | 0     | Lick sensor signal, valve open events                  |
| `RUN_TRAINING`  | 1     | Lick, valve, and running-speed plots                   |
| `EXPERIMENT`    | 2     | All of the above plus the trial-performance panel      |

Adding a new visualizer mode is a code change to `visualizers.py` and is typically required when
adding a new runtime mode with display needs that differ from the existing three modes.

### Real-time data sources

The visualizer reads from the binding classes' `SharedMemoryArray`-backed property accessors:

- `self._microcontrollers.lick.lick_count` — cumulative lick count
- `self._microcontrollers.valve.delivered_volume` — cumulative reward volume
- `self._microcontrollers.wheel_encoder.absolute_position` — current Unity position
- `self._microcontrollers.wheel_encoder.traveled_distance` — cumulative traveled distance

These properties are safe to read from the main process because they back onto SharedMemoryArrays
written by the communication subprocesses. See `experiment:microcontroller-interface`'s wrapper
lifecycle documentation for the SharedMemoryArray pattern.

---

## CLI command surface

The user-facing entry points live in
`sollertia_experiment/command_line_interfaces/execute.py`. They are registered as `sle session`
subcommands plus the top-level `sle run` and `sle maintain` commands:

| Command                                | Calls                       | Notes                                                        |
|----------------------------------------|-----------------------------|--------------------------------------------------------------|
| `sle session lick-training`            | `lick_training_logic`       | Defaults match `LickTrainingDescriptor` defaults              |
| `sle session run-training`             | `run_training_logic`        | Defaults match `RunTrainingDescriptor` defaults               |
| `sle session run-experiment`           | `experiment_logic`          | Takes an `--experiment` flag for the task template name      |
| `sle session check-window`             | `window_checking_logic`     | Window-checking maintenance mode                              |
| `sle maintain`                         | `maintenance_logic`         | Top-level command; not a `session` subcommand                |
| `sle run`                              | (entry point)               | Top-level entry for non-session commands                     |

Each CLI command:

1. Builds a `SessionData` instance from the CLI arguments (user, project, animal, weight, etc.).
2. Builds the appropriate descriptor with CLI overrides applied to defaults.
3. Writes the descriptor to disk (via assets-plugin tooling).
4. Calls the per-mode runtime logic function.

The CLI is the only public surface for starting a session. Direct calls to the runtime logic
functions from notebooks or scripts are not supported.

---

## Workflow: adding a new runtime mode

Adding a new mode is a coordinated cross-repository change. Follow the steps below in order. The
steps that touch repositories outside `sollertia-experiment` are delegated to other skills via
explicit handoffs.

### Step 1: Author the session descriptor (assets plugin)

Hand off to `assets:session-descriptors` to:

1. Add a new `SessionTypes` member in
   `sollertia_shared_assets/data_classes/session_data.py`.
2. Create the new descriptor dataclass in
   `sollertia_shared_assets/data_classes/runtime_data.py`. Follow the existing
   `LickTrainingDescriptor` / `RunTrainingDescriptor` patterns.
3. Register the descriptor in the preprocessing map.
4. Export the new class from the package's `__init__.py`.
5. Bump `sollertia-shared-assets` version.

After Step 1, the descriptor is part of the shared-assets vocabulary but not yet used by any
runtime.

### Step 2: Extend the state machine (this repo)

If the new mode requires a hardware state distinct from existing states:

1. Add a new member to `_MesoscopeVRStates` in `data_acquisition.py` with the next unused integer
   value.
2. Add a `set_<new_state>_state()` method to `_MesoscopeVRSystem` that drives the hardware into
   the new state (brake/screens/lickport/etc. configuration).
3. Ensure the new method logs the state change via DataLogger using
   `_MesoscopeVRLogMessageCodes.SYSTEM_STATE`.

If the mode reuses an existing state, skip this step.

### Step 3: Add a visualizer mode (this repo)

If the new mode's display needs differ from the existing visualizer modes:

1. Add a new member to `VisualizerMode` in `visualizers.py` with the next unused integer value.
2. Update `BehaviorVisualizer` to handle the new mode in its plot-construction and update logic.

If the existing visualizer modes cover the new mode, skip this step.

### Step 4: Author the runtime logic function (this repo)

In `data_acquisition.py`, add a new top-level function:

```python
def <new_mode>_logic(session_data: SessionData, descriptor: <NewMode>Descriptor) -> None:
    """Runs the <new mode> session.

    Args:
        session_data: The session metadata and file hierarchy.
        descriptor: The session-specific runtime parameters.
    """
    # 1. Start DataLogger.
    # 2. Construct _MesoscopeVRSystem.
    # 3. Drive state transitions and trial/event loops.
    # 4. Update the visualizer.
    # 5. Tear down on completion or KeyboardInterrupt.
```

Mirror the structure of an existing function (e.g., `lick_training_logic` for a non-trial mode,
`experiment_logic` for a trial-structured mode).

### Step 5: Add the CLI command (this repo)

In `command_line_interfaces/execute.py`, add a new function decorated with `@click.command()` or
`@session.command()` (for `sle session <subcommand>` syntax). The function:

1. Accepts `click.Context` plus per-CLI-flag arguments.
2. Constructs a `SessionData` and the new descriptor with overrides applied.
3. Writes the descriptor to the session directory.
4. Calls the new runtime logic function.

Mirror the structure of an existing CLI command (e.g., `lick_training` or `run_training`).

### Step 6: Export and version-bump (this repo)

1. Re-export the new logic function and descriptor from the package's `__init__.py` if other
   consumers need them.
2. Bump `sollertia-experiment` version in `pyproject.toml`.
3. Update the `sollertia-shared-assets` dependency pin in `pyproject.toml` to match the new
   minimum version (the version that includes the new descriptor).

### Step 7: Update this skill

Add the new mode to the [State machine](#state-machine), [Per-mode runtime logic
functions](#per-mode-runtime-logic-functions), and [CLI command surface](#cli-command-surface)
tables. If the mode introduced a new visualizer mode, update the
[VisualizerMode](#visualizermode) table.

---

## Workflow: adding a new MQTT topic

Mesoscope-VR's MQTT vocabulary is owned jointly by this repo (the runtime side) and the Unity
project (`sollertia-unity-tasks`). Adding a topic requires changes on both sides.

1. Add the new topic to `_MesoscopeVRMQTTTopics` in `data_acquisition.py`. Follow the existing
   `<noun>/<value>/` pattern.
2. Update the orchestrator's MQTT message routing to handle the new topic (subscribe + dispatch).
3. Coordinate with the Unity side — see `unity:mqtt-contract` for the Unity-side topic
   registration.
4. Bump `sollertia-experiment` version.

Topics that only flow one-way (e.g., outbound notifications to Unity that the Unity side
ignores) are still part of the contract and MUST be documented.

---

## Maintenance contract

This skill is updated when:

- A new state is added to `_MesoscopeVRStates`.
- A new runtime logic function is added.
- A new CLI command is added.
- A new `VisualizerMode` is added.
- A new MQTT topic is added to `_MesoscopeVRMQTTTopics`.
- A new log message code is added to `_MesoscopeVRLogMessageCodes`.
- The orchestrator's construction order or lifecycle changes.

This skill is NOT updated when:

- A descriptor's field surface changes (owned by `assets:session-descriptors`).
- The `SessionTypes` enum changes (owned by `assets:session-descriptors`).
- Hardware composition changes (owned by `experiment:mesoscope-vr`).
- Per-firmware-module wrapper APIs change (owned by `experiment:microcontroller-interface`).

When in doubt, re-read the relevant source file
(`sollertia_experiment/mesoscope_vr/data_acquisition.py`,
`sollertia_experiment/mesoscope_vr/visualizers.py`,
`sollertia_experiment/command_line_interfaces/execute.py`) and reconcile this skill against
ground truth.

---

## Related skills

| Skill                                              | Relationship                                                                       |
|----------------------------------------------------|------------------------------------------------------------------------------------|
| `experiment:mesoscope-vr`                          | Hardware composition for the binding classes the runtime composes.                  |
| `experiment:acquisition-system-design`             | Platform-general pattern. Provides construction/shutdown order constraints.        |
| `experiment:microcontroller-interface`             | Per-module wrapper API the orchestrator and visualizer consume.                    |
| assets plugin `/session-descriptors`               | Authors descriptors and the SessionTypes enum the runtime consumes.                |
| assets plugin `/task-templates`                    | Authors task templates the experiment runtime loads.                                |
| assets plugin `/experiment-configuration`          | Authors experiment configurations the runtime loads.                                |
| `experiment:data-management`                       | Downstream session-data lifecycle (preprocess, transfer, delete).                   |
| `experiment:session-snapshots`                     | Zaber position snapshots written at session end.                                    |
| `unity:gimbl-framework`                            | Unity-side framework for the VR game engine.                                        |
| `unity:mqtt-contract`                              | Unity-side MQTT topic registration.                                                 |
| `unity:task-prefabs`                               | Unity-side task prefab generation from task templates.                              |

---

## Verification checklist

```text
When adding a new runtime mode:

Cross-repo handoffs:
- [ ] Session descriptor authored via assets plugin /session-descriptors
- [ ] SessionTypes enum extended via assets plugin /session-descriptors
- [ ] sollertia-shared-assets version bumped

This repo (sollertia-experiment):
- [ ] _MesoscopeVRStates extended (if a new state is needed)
- [ ] set_<state>_state() method added to _MesoscopeVRSystem (if a new state)
- [ ] State change logged via _MesoscopeVRLogMessageCodes.SYSTEM_STATE
- [ ] VisualizerMode extended (if new display needs)
- [ ] BehaviorVisualizer updated to handle new mode
- [ ] Runtime logic function added to data_acquisition.py
- [ ] CLI command registered in command_line_interfaces/execute.py
- [ ] New function and descriptor re-exported from __init__.py (if external consumers need them)
- [ ] sollertia-experiment version bumped
- [ ] sollertia-shared-assets dependency pin updated to match new minimum version

Documentation:
- [ ] State table updated in this skill
- [ ] Runtime logic functions table updated in this skill
- [ ] CLI command surface table updated in this skill
- [ ] VisualizerMode table updated in this skill (if applicable)
- [ ] Log message codes table updated in this skill (if applicable)

Testing:
- [ ] Hardware verified via experiment:acquisition-system-setup before running session
- [ ] At least one dry-run executed with the new CLI command
- [ ] Session-data layout matches expectations (assets plugin /session-data verification)
```
