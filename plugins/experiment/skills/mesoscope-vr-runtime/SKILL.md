---
name: mesoscope-vr-runtime
description: >-
  Documents the Mesoscope-VR runtime behavior layer: the MesoscopeVRStates state machine, the
  _MesoscopeVRSystem orchestrator, the per-mode runtime logic functions, the BehaviorVisualizer and
  RuntimeControlUI, and the `sle mesoscope` CLI commands. Use when adding a new training mode or
  session type, extending the state machine, modifying visualizers, or wiring new CLI commands.
user-invocable: false
---

# Mesoscope-VR runtime

Documents the Mesoscope-VR runtime behavior layer — the state machine, the orchestrator class, the
per-mode runtime logic functions, the visualizer and control GUI, and the CLI surface. This skill is
the counterpart to `experiment:mesoscope-vr` (hardware composition); together they cover the full
Mesoscope-VR system.

For the platform-general acquisition-system runtime pattern, see `experiment:acquisition-system-runtime`.
For the platform-general static design pattern, see `experiment:acquisition-system-design`. For the
Unity VR task driver the orchestrator uses to couple to the game engine, see
`experiment:vr-driver-interface`.

---

## Scope

**Covers:**
- `MesoscopeVRStates` enumeration and state machine semantics
- `_MesoscopeVRSystem` orchestrator class — construction, state transitions, and the runtime cycle
- Per-mode runtime logic functions (`window_checking_logic`, `lick_training_logic`,
  `run_training_logic`, `experiment_logic`, `maintenance_logic`)
- `BehaviorVisualizer` / `VisualizerMode` and the `RuntimeControlUI` control GUI
- Session-descriptor consumption pattern (read by runtime, written by assets-plugin tooling)
- `sle mesoscope` CLI command surface (`configure`, `maintain`, `run`, session-data subcommands)
- Workflow for adding a new training mode (cross-repo: sollertia-shared-assets + sollertia-experiment)

**Does not cover** (delegated):
- Mesoscope-VR hardware composition (binding classes, calibration dataclasses, system YAML) — see `experiment:mesoscope-vr`
- The platform-general runtime pattern — see `experiment:acquisition-system-runtime`
- The Unity VR task driver, its MQTT topic vocabulary, and trial decomposition — see `experiment:vr-driver-interface`
- Per-firmware-module wrappers and slmc Module classes — see `experiment:microcontroller-interface`
- Session descriptors and `SessionTypes` enum — owned by the assets plugin's `/session-descriptors`
- Task template and trial-structure authoring — owned by the assets plugin's `/task-templates`
- Per-experiment configuration — owned by the assets plugin's `/experiment-configuration`
- Session-data lifecycle (preprocessing, transfer, deletion) — owned by `experiment:data-management`

---

## Runtime layer architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  sle mesoscope CLI (sollertia_experiment/interfaces/mesoscope_vr.py)          │
│  ──────────────────────────────────────────────────────────────              │
│  sle mesoscope run window-checking ──┐                                        │
│  sle mesoscope run lick-training   ──┤                                        │
│  sle mesoscope run run-training    ──┼──► builds SessionData + descriptor,    │
│  sle mesoscope run experiment      ──┤    then calls the per-mode logic fn    │
│  sle mesoscope maintain            ──┘                                        │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  Per-mode runtime logic functions                                            │
│  (sollertia_experiment/mesoscope_vr/data_acquisition.py)                     │
│  ─────────────────────────────────────────────────                           │
│  window_checking_logic(...)   lick_training_logic(...)                       │
│  run_training_logic(...)      experiment_logic(...)   maintenance_logic()    │
│                                                                              │
│  Each constructs a _MesoscopeVRSystem, drives its state transitions, and     │
│  runs its runtime_cycle() loop over the session's lifetime.                  │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │ composes
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  _MesoscopeVRSystem orchestrator (mesoscope_vr/system_controller.py)         │
│  ──────────────────────────────────────────────────────────────             │
│  - Composes MicroControllerInterfaces, VideoSystems, ZaberMotors             │
│  - Drives the MesoscopeVRStates system state machine                         │
│  - Owns a VRTaskDriver (experiment sessions only) for Unity coupling         │
│  - Owns BehaviorVisualizer + RuntimeControlUI                                │
│  - runtime_cycle() fans out to _data_cycle / _unity_cycle / _ui_cycle /      │
│    _mesoscope_cycle each iteration                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Authoritative bases

| Concern                                           | Authority                                              |
|---------------------------------------------------|--------------------------------------------------------|
| Platform-general runtime pattern                  | `experiment:acquisition-system-runtime`                |
| Mesoscope-VR hardware composition                 | `experiment:mesoscope-vr`                              |
| Unity VR task driver + MQTT contract              | `experiment:vr-driver-interface`                       |
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

`MesoscopeVRStates` (in `sollertia_experiment/mesoscope_vr/system.py`) is the `IntEnum` that encodes
the runtime's current hardware-control mode. Each state determines the brake state, screen state,
lickport availability, and which sensors are active.

Current states:

| State           | Value | Meaning                                                                          |
|-----------------|-------|----------------------------------------------------------------------------------|
| `IDLE`          | 0     | Not conducting a session (and the paused state — brake engaged, screens off, only the mesoscope-frame TTL sensor active) |
| `REST`          | 1     | Rest period of an experiment session                                             |
| `RUN`           | 2     | Run period of an experiment session                                              |
| `LICK_TRAINING` | 3     | Lick training session (lickport-only behavior)                                   |
| `RUN_TRAINING`  | 4     | Run training session (wheel-only behavior)                                       |

The enum exposes `to_dict()` (lowercased, underscores→spaces) for the visualizer title bar.

State values 0–4 are taken. New states SHOULD use the next unused value (5) unless a non-contiguous
code carries semantic meaning that justifies the gap.

The system distinguishes two state axes:

- **System state** — a `MesoscopeVRStates` value, set by the hardware-state methods below and logged
  with `_MesoscopeVRLogMessageCodes.SYSTEM_STATE` (code 1).
- **Runtime state (stage)** — an integer "stage" code within a session, set by
  `change_runtime_state(new_state)` and logged with `_MesoscopeVRLogMessageCodes.RUNTIME_STATE`
  (code 2). The pause/resume machinery restores the pre-pause runtime state after an `IDLE` pause.

---

## `_MesoscopeVRSystem` orchestrator

`_MesoscopeVRSystem` (in `sollertia_experiment/mesoscope_vr/system_controller.py`) is the central
orchestrator for a single session. It is named with a leading underscore because it is internal to
the runtime — external consumers go through the per-mode runtime logic functions, not the system
class directly.

### Construction

The orchestrator's `__init__` receives the started `DataLogger`, the loaded
`MesoscopeSystemConfiguration`, the `SessionData`, and (for experiments) the
`MesoscopeExperimentConfiguration`. It builds, in order:

1. `self._trial_state: _TrialState` — per-session trial tracking.
2. `MicroControllerInterfaces(data_logger, microcontroller_configuration)`.
3. `VideoSystems(data_logger, camera_configuration, output_directory)`.
4. `ZaberMotors(zaber_positions, zaber_configuration=...assets)`.
5. `VRTaskDriver(configuration=...assets.vr_task, task_template, expected_scene_name)` — **only**
   for `SessionTypes.MESOSCOPE_EXPERIMENT` sessions; `None` otherwise. The task template is loaded
   via `load_vr_task_template()`. See `experiment:vr-driver-interface`.
6. `self._ui: RuntimeControlUI` — the runtime control GUI (reads the valve/gas-puff trackers).
7. `self._visualizer: BehaviorVisualizer`.

After construction the orchestrator owns `self._microcontrollers`, `self._cameras`, `self._zaber`,
`self._vr_task` (`VRTaskDriver | None`), `self._ui`, `self._visualizer`, and `self._trial_state`.

The construction/teardown ordering constraints (hardware-lane order, keepalive enforcement) come
from `experiment:acquisition-system-runtime`.

### State transitions

The orchestrator exposes one hardware-state method per system state. Each drives the binding classes
into the target configuration (brake, screens, lickport, sensor enable/disable), updates the
visualizer mode, and logs the change:

```python
def idle(self)       -> None: ...   # MesoscopeVRStates.IDLE
def rest(self)       -> None: ...   # MesoscopeVRStates.REST
def run(self)        -> None: ...   # MesoscopeVRStates.RUN
def lick_train(self) -> None: ...   # MesoscopeVRStates.LICK_TRAINING
def run_train(self)  -> None: ...   # MesoscopeVRStates.RUN_TRAINING
```

Internally these call `_change_system_state(new_state)`, which applies the hardware configuration
and logs `_MesoscopeVRLogMessageCodes.SYSTEM_STATE`. `change_runtime_state(new_state)` separately
advances the within-session runtime stage. State transitions are idempotent.

### Runtime cycle

`runtime_cycle()` is the per-iteration heartbeat the logic functions call in a loop. It fans out to:

- `_data_cycle()` — drains microcontroller data, updates trackers, pushes motion/lick to the VR
  driver and the visualizer.
- `_unity_cycle()` — consumes at most one `VRTaskEvent` from the `VRTaskDriver` and dispatches it.
- `_ui_cycle()` — services the `RuntimeControlUI` (pause/resume, threshold modifiers, manual reward).
- `_mesoscope_cycle()` — services mesoscope frame-acquisition bookkeeping.

`start()` and `stop()` bring the session up and tear it down. `_pause_runtime()` / `_resume_runtime()`
implement the IDLE-pause behavior, and `_terminate_runtime()` handles end-of-session shutdown.

### Trial-state tracking

`_TrialState` (in `sollertia_experiment/mesoscope_vr/acquisition_components.py`) consolidates the
per-trial tracking attributes used during experiment runtimes. It tracks both reinforcing (water
reward) and aversive (gas puff) trial types, including completed trial count, cumulative distance,
per-type guided-trial counts, per-type recovery thresholds (engage guidance after N consecutive
failures), and per-type in-flight "rewarded" flags.

The orchestrator's `setup_reinforcing_guidance()` / `setup_aversive_guidance()` configure guidance,
and `_refresh_trial_state_from_vr_decomposition()` rebuilds the trial parameter arrays from the
ordered trial names the `VRTaskDriver` produces by decomposing the active Unity cue sequence.

Adding new trial-tracking dimensions requires extending `_TrialState` and updating the
orchestrator's stimulus handling in `experiment_logic` / `runtime_cycle`.

### Log message codes

`_MesoscopeVRLogMessageCodes` (`IntEnum`, in `acquisition_components.py`) defines the event codes the
orchestrator emits to the DataLogger:

| Code | Name                          | Meaning                                                       |
|------|-------------------------------|---------------------------------------------------------------|
| 1    | `SYSTEM_STATE`                | System has changed hardware-control (system) state            |
| 2    | `RUNTIME_STATE`               | Acquired session has changed runtime state (stage)            |
| 3    | `REINFORCING_GUIDANCE_STATE`  | Reinforcing-trial guidance state changed                      |
| 4    | `AVERSIVE_GUIDANCE_STATE`     | Aversive-trial guidance state changed                         |
| 5    | `DISTANCE_SNAPSHOT`           | Distance snapshot taken (e.g., on cue sequence change)        |

New events get the next unused code (currently 6). These codes are consumed by downstream behavior
processing in the forging plugin's behavior pipeline.

---

## Unity coupling

For experiment sessions the orchestrator couples to the Unity game engine through a `VRTaskDriver`
(`self._vr_task`). The orchestrator pushes motion and lick events to the driver and consumes typed
`VRTaskEvent`s from it each `_unity_cycle()`; the driver owns the MQTT broker connection, the topic
vocabulary (`_VRTaskMQTTTopics`), scene/cue verification, and cue-sequence trial decomposition.

The driver, its event model, the MQTT topic contract, and the trial-decomposition layer are
documented in `experiment:vr-driver-interface`. The Unity side of the contract is documented in
`unity:gimbl-framework` and `unity:mqtt-contract`.

---

## Per-mode runtime logic functions

Each runtime mode has a top-level function in `sollertia_experiment/mesoscope_vr/data_acquisition.py`
that:

1. Validates inputs and prepares output directories.
2. Starts `DataLogger`.
3. Constructs `_MesoscopeVRSystem`.
4. Loads the session-specific descriptor (read from the per-session YAML written by the assets plugin).
5. Drives state transitions and the `runtime_cycle()` loop.
6. Tears down in the reverse order on completion or error.

Current functions:

| Function                  | Purpose                                                                              |
|---------------------------|--------------------------------------------------------------------------------------|
| `window_checking_logic`   | Cranial window maintenance session (no behavior)                                     |
| `lick_training_logic`     | Lickport-only training (animal learns to operate the lickport for water rewards)     |
| `run_training_logic`      | Wheel-only training (animal learns to run for water rewards)                          |
| `experiment_logic`        | Full Mesoscope-VR experiment session with VR trial structure                         |
| `maintenance_logic`       | Hardware maintenance (valve calibration, motor positioning, brake testing)           |

Each session-running function takes a `SessionData` (file-paths metadata) and a session-specific
descriptor (the mode's runtime parameters loaded from YAML). `maintenance_logic()` takes no session.

### Session descriptor consumption

Descriptors are owned by `sollertia-shared-assets` and authored via `assets:session-descriptors`.
The runtime treats them as read-only:

- The CLI command instantiates the descriptor with default values and applies per-flag overrides.
- The descriptor is written to the session's directory before the runtime begins.
- The runtime logic function reads the descriptor from disk at session start and uses it to
  parameterize the state machine.

The descriptor is NEVER mutated by the runtime — runtime-discovered values (total water delivered,
total trials completed) are recorded in the DataLogger archive, not back into the descriptor file.
The descriptor is the "session plan"; the DataLogger archive is the "session result."

---

## BehaviorVisualizer and RuntimeControlUI

`BehaviorVisualizer` (in `sollertia_experiment/mesoscope_vr/visualizer.py`) renders real-time
behavior data using matplotlib with a Qt backend, driven by a `_BlitManager` for fast redraws. It is
constructed by the orchestrator and `open()`ed in the mode selected for the session.

`RuntimeControlUI` (in `sollertia_experiment/mesoscope_vr/runtime_ui.py`) is the interactive control
GUI surfaced during a session — pause/resume, run-training threshold modifiers, and manual reward
delivery. The maintenance GUI lives separately in `maintenance_ui.py`.

### VisualizerMode

The visualizer's display mode is set by `VisualizerMode` (`IntEnum`, in `visualizer.py`):

| Mode            | Value | Panels displayed                                       |
|-----------------|-------|--------------------------------------------------------|
| `LICK_TRAINING` | 0     | Lick sensor signal, valve open events                  |
| `RUN_TRAINING`  | 1     | Lick, valve, and running-speed plots                   |
| `EXPERIMENT`    | 2     | All of the above plus the trial-performance panel      |

Adding a new visualizer mode is a code change to `visualizer.py`, typically required when adding a
new runtime mode whose display needs differ from the existing three.

### Real-time data sources

The visualizer and orchestrator read from the binding classes' `SharedMemoryArray`-backed property
accessors:

- `self._microcontrollers.lick.lick_count` — cumulative lick count
- `self._microcontrollers.valve.delivered_volume` — cumulative reward volume
- `self._microcontrollers.wheel_encoder.absolute_position` — current Unity position
- `self._microcontrollers.wheel_encoder.traveled_distance` — cumulative traveled distance

These properties are safe to read from the main process because they back onto SharedMemoryArrays
written by the communication subprocesses. See `experiment:microcontroller-interface`'s wrapper
lifecycle documentation for the SharedMemoryArray pattern.

---

## CLI command surface

The user-facing entry points live in `sollertia_experiment/interfaces/mesoscope_vr.py`, registered
under the `sle mesoscope` command group (itself registered on the top-level `sle` group in
`interfaces/entry_points.py`):

| Command                              | Calls                       | Notes                                                  |
|--------------------------------------|-----------------------------|--------------------------------------------------------|
| `sle mesoscope configure`            | `create_system_configuration_file` | Writes a template system configuration YAML     |
| `sle mesoscope maintain`             | `maintenance_logic`         | Hardware maintenance GUI (no session)                  |
| `sle mesoscope run window-checking`  | `window_checking_logic`     | Cranial-window maintenance mode                        |
| `sle mesoscope run lick-training`    | `lick_training_logic`       | Defaults match the lick-training descriptor            |
| `sle mesoscope run run-training`     | `run_training_logic`        | Absolute speed/duration threshold targets via flags    |
| `sle mesoscope run experiment`       | `experiment_logic`          | Takes `--experiment` for the experiment configuration  |
| `sle mesoscope preprocess`           | (session data lifecycle)    | See `experiment:data-management`                       |
| `sle mesoscope delete`               | (session data lifecycle)    | See `experiment:data-management`                       |
| `sle mesoscope migrate`              | (session data lifecycle)    | See `experiment:data-management`                       |

`sle mesoscope run` is a command group; the `--user`, `--project`, `--animal`, and `--animal-weight`
options are supplied on `run` and shared by every session subcommand. Each session subcommand builds
a `SessionData`, builds the descriptor with CLI overrides, writes the descriptor to disk, and calls
the per-mode logic function. The CLI is the only public surface for starting a session.

---

## Workflow: adding a new runtime mode

Adding a new mode is a coordinated cross-repository change. Follow the steps in order; steps that
touch repositories outside `sollertia-experiment` are delegated via explicit handoffs.

### Step 1: Author the session descriptor (assets plugin)

Hand off to `assets:session-descriptors` to add the new `SessionTypes` member, create the descriptor
dataclass, register it in the preprocessing map, export it, and bump the `sollertia-shared-assets`
version.

### Step 2: Extend the state machine (this repo)

If the new mode requires a hardware state distinct from existing states:

1. Add a new member to `MesoscopeVRStates` in `mesoscope_vr/system.py` with the next unused value.
2. Add a hardware-state method (`idle`/`rest`/`run`/`lick_train`/`run_train`-style) to
   `_MesoscopeVRSystem` in `system_controller.py` that drives the hardware into the new state and
   calls `_change_system_state()`.

If the mode reuses an existing state, skip this step.

### Step 3: Add a visualizer mode (this repo)

If the new mode's display needs differ from the existing modes, add a `VisualizerMode` member in
`visualizer.py` and update `BehaviorVisualizer`'s plot-construction and update logic.

### Step 4: Author the runtime logic function (this repo)

In `mesoscope_vr/data_acquisition.py`, add a top-level `<new_mode>_logic(session_data, descriptor)`
function that starts the `DataLogger`, constructs `_MesoscopeVRSystem`, drives the state transitions
and `runtime_cycle()` loop, and tears down on completion or `KeyboardInterrupt`. Mirror an existing
function (`lick_training_logic` for a non-trial mode, `experiment_logic` for a trial-structured mode).

### Step 5: Add the CLI command (this repo)

In `interfaces/mesoscope_vr.py`, add a new `@run.command(...)` (for a session) or `@mesoscope.command(...)`
subcommand that builds a `SessionData` and descriptor with overrides, writes the descriptor, and
calls the new logic function. Mirror an existing command.

### Step 6: Export and version-bump (this repo)

Re-export the new logic function/descriptor from the package `__init__.py` if external consumers need
them, bump `sollertia-experiment` in `pyproject.toml`, and update the `sollertia-shared-assets`
dependency pin to the version that includes the new descriptor.

### Step 7: Update this skill

Add the new mode to the [State machine](#state-machine), [Per-mode runtime logic
functions](#per-mode-runtime-logic-functions), and [CLI command surface](#cli-command-surface)
tables, and the [VisualizerMode](#visualizermode) table if a new mode was added.

---

## Maintenance contract

This skill is updated when:

- A new state is added to `MesoscopeVRStates`.
- A new runtime logic function or hardware-state method is added.
- A new `sle mesoscope` command is added.
- A new `VisualizerMode` is added.
- A new log message code is added to `_MesoscopeVRLogMessageCodes`.
- The orchestrator's construction order, runtime cycle, or lifecycle changes.

This skill is NOT updated when:

- A descriptor's field surface or the `SessionTypes` enum changes (owned by `assets:session-descriptors`).
- Hardware composition changes (owned by `experiment:mesoscope-vr`).
- Per-firmware-module wrapper APIs change (owned by `experiment:microcontroller-interface`).
- The Unity VR driver, MQTT topics, or trial decomposition change (owned by `experiment:vr-driver-interface`).

When in doubt, re-read the relevant source file (`mesoscope_vr/system_controller.py`,
`mesoscope_vr/system.py`, `mesoscope_vr/acquisition_components.py`, `mesoscope_vr/visualizer.py`,
`mesoscope_vr/runtime_ui.py`, `mesoscope_vr/data_acquisition.py`, `interfaces/mesoscope_vr.py`) and
reconcile this skill against ground truth.

---

## Related skills

| Skill                                              | Relationship                                                                       |
|----------------------------------------------------|------------------------------------------------------------------------------------|
| `experiment:acquisition-system-runtime`            | Platform-general runtime pattern this system instantiates.                          |
| `experiment:mesoscope-vr`                          | Hardware composition for the binding classes the runtime composes.                  |
| `experiment:acquisition-system-design`             | Platform-general static composition pattern.                                        |
| `experiment:vr-driver-interface`                   | The `VRTaskDriver` the orchestrator uses for Unity coupling; MQTT + trial decomposition. |
| `experiment:microcontroller-interface`             | Per-module wrapper API the orchestrator and visualizer consume.                     |
| assets plugin `/session-descriptors`               | Authors descriptors and the `SessionTypes` enum the runtime consumes.               |
| assets plugin `/task-templates`                    | Authors task templates the experiment runtime loads.                                |
| assets plugin `/experiment-configuration`          | Authors experiment configurations the runtime loads.                                |
| `experiment:data-management`                       | Downstream session-data lifecycle (preprocess, transfer, delete).                   |
| `experiment:session-snapshots`                     | Zaber/mesoscope position snapshots captured at session start.                       |
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
- [ ] MesoscopeVRStates extended in system.py (if a new state is needed)
- [ ] Hardware-state method added to _MesoscopeVRSystem and routed through _change_system_state (if a new state)
- [ ] VisualizerMode extended in visualizer.py (if new display needs)
- [ ] BehaviorVisualizer updated to handle the new mode
- [ ] Runtime logic function added to data_acquisition.py
- [ ] CLI command registered in interfaces/mesoscope_vr.py
- [ ] New function and descriptor re-exported from __init__.py (if external consumers need them)
- [ ] sollertia-experiment version bumped
- [ ] sollertia-shared-assets dependency pin updated to match the new minimum version

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
