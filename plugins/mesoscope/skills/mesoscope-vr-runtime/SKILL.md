---
name: mesoscope-vr-runtime
description: >-
  Documents the Mesoscope-VR runtime behavior layer: the MesoscopeVRStates state machine, the
  MesoscopeVRSystem orchestrator, the per-mode runtime logic functions, the BehaviorVisualizer and
  RuntimeControlUI, and the `sle mesoscope` CLI commands. Use when adding a new training mode or
  session type, extending the state machine, modifying visualizers, or wiring new CLI commands.
user-invocable: false
---

# Mesoscope-VR runtime

Documents the Mesoscope-VR runtime behavior layer — the state machine, the orchestrator class, the
per-mode runtime logic functions, the visualizer and control GUI, and the CLI surface. This skill is
the counterpart to `/mesoscope-vr` (hardware composition); together they cover the full
Mesoscope-VR system.

For the platform-general acquisition-system runtime pattern, see `experiment:acquisition-system-runtime`.
For the platform-general static design pattern, see `experiment:acquisition-system-design`. For the
Unity VR task driver the orchestrator uses to couple to the game engine, see
`experiment:vr-driver-interface`.

---

## Scope

**Covers:**
- `MesoscopeVRStates` enumeration and state machine semantics
- `MesoscopeVRSystem` orchestrator class — construction, state transitions, and the runtime cycle
- Per-mode runtime logic functions (`window_checking_logic`, `lick_training_logic`,
  `run_training_logic`, `experiment_logic`, `maintenance_logic`)
- `BehaviorVisualizer` / `VisualizerMode` and the `RuntimeControlUI` control GUI
- Session-descriptor consumption pattern (read by runtime, written by assets-plugin tooling)
- `sle mesoscope` CLI command surface (`configure`, `maintain`, `run`, session-data subcommands)
- Workflow for adding a new training mode (cross-repo: sollertia-shared-assets + sollertia-experiment)

**Does not cover** (delegated):
- Mesoscope-VR hardware composition (binding classes, configuration dataclasses, system YAML) — see `/mesoscope-vr`
- The platform-general runtime pattern — see `experiment:acquisition-system-runtime`
- The Unity VR task driver, its MQTT topic vocabulary, and trial decomposition — see `experiment:vr-driver-interface`
- Per-firmware-module wrappers and slmc Module classes — see `experiment:microcontroller-interface`
- Descriptor and hardware-state field schemas — owned by `/mesoscope-vr-session-schema`
- The generic descriptor tools — owned by `assets:session-descriptors`
- The `SessionTypes` member and its `DESCRIPTOR_REGISTRY` entry — owned by `assets:library-extension`
- Task template and trial-structure authoring — owned by `assets:task-templates`
- Per-experiment configuration — owned by `assets:experiment-configuration`
- Session-data lifecycle (preprocessing, transfer, deletion) — owned by `experiment:data-management`

---

## Runtime layer architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  sle mesoscope CLI (sollertia_experiment/interfaces/mesoscope_vr.py)          │
│  ──────────────────────────────────────────────────────────────              │
│  sle mesoscope run window-checking ──┐                                        │
│  sle mesoscope run lick-training   ──┤                                        │
│  sle mesoscope run run-training    ──┼──► parses CLI options, then calls      │
│  sle mesoscope run experiment      ──┤    the matching per-mode logic fn      │
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
│  lick_training_logic, run_training_logic, and experiment_logic construct a   │
│  MesoscopeVRSystem, drive its state transitions, and run its runtime_cycle() │
│  loop. window_checking_logic and maintenance_logic build their own hardware  │
│  assets directly.                                                            │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │ composes
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  MesoscopeVRSystem orchestrator (mesoscope_vr/system_controller.py)          │
│  ──────────────────────────────────────────────────────────────             │
│  - Composes MicroControllerInterfaces, VideoSystems, ZaberMotors             │
│  - Drives the MesoscopeVRStates system state machine                         │
│  - Owns a VRTaskDriver (experiment sessions only) for Unity coupling         │
│  - Owns BehaviorVisualizer + RuntimeControlUI                                │
│  - runtime_cycle() runs _data_cycle, visualizer update, and _ui_cycle every  │
│    iteration, plus _unity_cycle / _mesoscope_cycle for experiment sessions   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Authoritative bases

| Concern                                               | Authority                               |
|-------------------------------------------------------|-----------------------------------------|
| Platform-general runtime pattern                      | `experiment:acquisition-system-runtime` |
| Mesoscope-VR hardware composition                     | `/mesoscope-vr`                         |
| Unity VR task driver + MQTT contract                  | `experiment:vr-driver-interface`        |
| Per-firmware-module wrapper API                       | `experiment:microcontroller-interface`  |
| `SessionTypes` member and `DESCRIPTOR_REGISTRY` entry | `assets:library-extension`              |
| Descriptor and hardware-state field schemas           | `/mesoscope-vr-session-schema`          |
| Generic descriptor read and write tooling             | `assets:session-descriptors`            |
| Task template authoring (trial structure)             | `assets:task-templates`                 |
| Experiment configuration authoring                    | `assets:experiment-configuration`       |
| Session-data lifecycle                                | `experiment:data-management`            |

This skill documents how the runtime *consumes* descriptors, session data, and task templates. It
does NOT document how to author them.

---

## State machine

`MesoscopeVRStates` (in `sollertia_experiment/mesoscope_vr/system.py`) is the `IntEnum` that encodes
the runtime's current hardware-control mode. Each state determines the brake state, screen state,
lickport availability, and which sensors are active.

Current states:

| State           | Value | Meaning                                                                                                                  |
|-----------------|-------|--------------------------------------------------------------------------------------------------------------------------|
| `IDLE`          | 0     | Not conducting a session (and the paused state — brake engaged, screens off, only the mesoscope-frame TTL sensor active) |
| `REST`          | 1     | Rest period of an experiment session                                                                                     |
| `RUN`           | 2     | Run period of an experiment session                                                                                      |
| `LICK_TRAINING` | 3     | Lick training session (lickport-only behavior)                                                                           |
| `RUN_TRAINING`  | 4     | Run training session (wheel-only behavior)                                                                               |

The enum exposes `to_dict()` (lowercased member names, underscores→spaces). Its three call sites all
sit in `_generate_hardware_state_snapshot()`, which stores the mapping as `MesoscopeHardwareState`'s
`system_state_codes` field in the session's `hardware_state.yaml`. Downstream behavior processing in
the forging plugin reads that mapping to decode logged system-state codes.

State values 0–4 are taken. New states SHOULD use the next unused value (5) unless a non-contiguous
code carries semantic meaning that justifies the gap.

The system distinguishes two state axes:

- **System state** — a `MesoscopeVRStates` value, set by the hardware-state methods below and logged
  with `MesoscopeVRLogMessageCodes.SYSTEM_STATE` (code 1).
- **Runtime state (stage)** — an integer "stage" code within a session, set by
  `change_runtime_state(new_state)` and logged with `MesoscopeVRLogMessageCodes.RUNTIME_STATE`
  (code 2). The pause/resume machinery restores the pre-pause runtime state after an `IDLE` pause.

---

## `MesoscopeVRSystem` orchestrator

`MesoscopeVRSystem` (in `sollertia_experiment/mesoscope_vr/system_controller.py`) is the central
orchestrator for a single session. External consumers go through the per-mode runtime logic functions
rather than constructing the system class directly.

### Construction

The orchestrator's `__init__` takes `session_data`, `session_descriptor`, and (for experiments)
`experiment_configuration=None`. It resolves the `MesoscopeSystemConfiguration` internally via
`get_system_configuration()` and constructs its own `DataLogger`; neither is passed in. It builds, in
order:

1. `self._trial_state: TrialState` — per-session trial tracking.
2. `self._logger: DataLogger` — the data logger that records messages from every managed source.
3. `MicroControllerInterfaces(data_logger, microcontroller_configuration)`.
4. `VideoSystems(data_logger, camera_configuration, output_directory)`.
5. `ZaberMotors(zaber_positions, zaber_configuration=...assets)`.
6. `VRTaskDriver(configuration=...assets.vr_task, task_template, expected_scene_name)` — **only**
   for `SessionTypes.MESOSCOPE_EXPERIMENT` sessions; `None` otherwise. The task template is loaded
   via `load_vr_task_template()`. See `experiment:vr-driver-interface`.
7. `self._mesoscope: MesoscopeDriver`, built with `configuration=...assets.vr_task` and
   `acquisition=...acquisition` — the MQTT control surface for the ScanImage software running on the
   ScanImagePC. See `/mesoscope-vr`.
8. `self._ui: RuntimeControlUI` — the runtime control GUI (reads the valve/gas-puff trackers).
9. `self._visualizer: BehaviorVisualizer`.

After construction the orchestrator owns `self._microcontrollers`, `self._cameras`,
`self._zaber_motors`, `self._vr_task` (`VRTaskDriver | None`), `self._mesoscope`, `self._ui`,
`self._visualizer`, and `self._trial_state`.

The construction/teardown ordering constraints (hardware-subsystem order, keepalive enforcement) come
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
and logs `MesoscopeVRLogMessageCodes.SYSTEM_STATE`. `change_runtime_state(new_state)` separately
advances the within-session runtime stage. State transitions are idempotent.

### Runtime cycle

`runtime_cycle()` is the per-iteration heartbeat the logic functions call in a loop. Every iteration
runs, in order:

- `_data_cycle()`, which drains microcontroller data, updates trackers, and pushes motion/lick to the
  VR driver and the visualizer.
- `self._visualizer.update()`, which redraws the behavior visualizer.
- `_ui_cycle()`, which services the `RuntimeControlUI` (pause/resume, threshold modifiers, manual
  reward).

`SessionTypes.MESOSCOPE_EXPERIMENT` sessions run two additional cycles at the end of each iteration.
Window-checking and maintenance runtimes build their hardware assets directly and stay outside the
cycle entirely. Window-checking runs a linear sequence of blocking operator prompts, and maintenance
runs its own control loop driven by the maintenance GUI. Lick-training and run-training runtimes run
the cycle and skip both of these additional cycles:

- `_unity_cycle()`, which consumes at most one `VRTaskEvent` from the `VRTaskDriver` and dispatches it. For
  `STIMULUS_TRIGGERED` it resolves the trial outcome (success / guided / failure) from the event's `delivered` and
  `cause` and reports it to the visualizer, labeling a trial guided only when guidance actually fired.
- `_mesoscope_cycle()`, which services mesoscope frame-acquisition bookkeeping.

`start()` and `stop()` bring the session up and tear it down. For
`SessionTypes.MESOSCOPE_EXPERIMENT` sessions, `start()` calls `self._mesoscope.connect()` and then
`setup_mesoscope(session_data=..., mesoscope_data=..., mesoscope_driver=...)` to bring the ScanImage
software up to acquisition readiness. When `stop()` is invoked on a system whose `start()` was
interrupted before completing initialization (`not self._started`), it delegates to
`_emergency_shutdown()`, which tears down whatever assets `start()` managed to bring up. That teardown
executes each action through `run_shutdown_step(description, step)` (a public function in
`acquisition_components.py`) so an error or interrupt in one step cannot prevent the remaining
teardown steps from running. `_pause_runtime()` / `_resume_runtime()` implement the IDLE-pause
behavior, and `_terminate_runtime()` handles end-of-session shutdown.

### Trial-state tracking

`TrialState` (in `sollertia_experiment/mesoscope_vr/acquisition_components.py`) consolidates the
per-trial tracking attributes used during experiment runtimes. It tracks both reinforcing (water
reward) and aversive (gas puff) trial types, including completed trial count, cumulative distance,
per-type guided-trial counts, per-type recovery thresholds (engage guidance after N consecutive
failures), and per-type in-flight "rewarded" flags.

The orchestrator's `setup_reinforcing_guidance()` / `setup_aversive_guidance()` configure guidance,
and `_refresh_trial_state_from_vr_decomposition()` rebuilds the trial parameter arrays from the
ordered trial names the `VRTaskDriver` produces by decomposing the active Unity cue sequence.

Adding new trial-tracking dimensions requires extending `TrialState` and updating the
orchestrator's stimulus handling in `experiment_logic` / `runtime_cycle`.

### Log message codes

`MesoscopeVRLogMessageCodes` (`IntEnum`, in `acquisition_components.py`) defines the event codes the
orchestrator emits to the DataLogger:

| Code | Name                         | Meaning                                                       |
|------|------------------------------|---------------------------------------------------------------|
| 1    | `SYSTEM_STATE`               | System has changed hardware-control (system) state            |
| 2    | `RUNTIME_STATE`              | Acquired session has changed runtime state (stage)            |
| 3    | `REINFORCING_GUIDANCE_STATE` | Reinforcing-trial guidance state changed                      |
| 4    | `AVERSIVE_GUIDANCE_STATE`    | Aversive-trial guidance state changed                         |
| 5    | `DISTANCE_SNAPSHOT`          | Total traveled distance at Unity-signaled runtime termination |

New events get the next unused code (currently 6). These codes are consumed by downstream behavior
processing in the forging plugin's behavior pipeline.

---

## Unity coupling

For experiment sessions the orchestrator couples to the Unity game engine through a `VRTaskDriver`
(`self._vr_task`). The orchestrator pushes motion and lick events to the driver and consumes typed
`VRTaskEvent`s from it each `_unity_cycle()`; the driver owns the MQTT broker connection, the topic
vocabulary (`_VRTaskMQTTTopics`), scene/cue verification, cue-sequence trial decomposition, and the editor
MCP Bridge it uses to open the scene and control Play Mode. The driver's `setup()` opens the scene and arms
Unity through the bridge (no manual play-button prompt), so `start()` only brackets it with the VR-screen
enable/disable.

On a `UNITY_TERMINATED` event the orchestrator enters an emergency pause; on resume, `_resume_runtime()`
calls `resume_after_unity_restart()`, which re-arms Unity through the bridge and re-fetches the cue sequence
— the operator does not press the play button.

The driver, its event model, the MQTT topic contract, the editor bridge, and the trial-decomposition layer
are documented in `experiment:vr-driver-interface`. The Unity side of the contract is documented in
`unity:gimbl-framework`, `unity:mqtt-contract`, `unity:play-mode`, `unity:task-scenes`, and
`unity:scene-setup`.

---

## Per-mode runtime logic functions

The sequence every per-mode logic function follows, the table of the current functions, and the session-descriptor
consumption pattern are documented in [`references/runtime-surface.md`](references/runtime-surface.md).

---

## BehaviorVisualizer and RuntimeControlUI

`BehaviorVisualizer` (in `sollertia_experiment/mesoscope_vr/visualizer.py`) renders real-time
behavior data using matplotlib with a Qt backend, driven by a `_BlitManager` for fast redraws. It is
constructed by the orchestrator and `open()`ed in the mode selected for the session.

`RuntimeControlUI` (in `sollertia_experiment/mesoscope_vr/runtime_ui.py`) is the interactive control
GUI surfaced during a session — pause/resume, run-training threshold modifiers, and manual reward
delivery. The maintenance GUI lives separately in `maintenance_ui.py`.

The module-level `RUN_TRAINING_THRESHOLD_LIMITS` (a frozen `RunTrainingThresholdLimits` in `mesoscope_vr/system.py`)
fixes the run-training speed bounds at 0.1-5.0 cm/s and the duration bounds at 0.05-5.0 s. The run-training logic
clamps the effective thresholds to these bounds and the control GUI constrains its spin boxes to them, so a requested
value outside the range is silently clamped rather than rejected.

### VisualizerMode

The visualizer's display mode is set by `VisualizerMode` (`IntEnum`, in `visualizer.py`):

| Mode            | Value | Panels displayed                                  |
|-----------------|-------|---------------------------------------------------|
| `LICK_TRAINING` | 0     | Lick sensor signal, valve open events             |
| `RUN_TRAINING`  | 1     | Lick, valve, and running-speed plots              |
| `EXPERIMENT`    | 2     | All of the above plus the trial-performance panel |

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

The `sle mesoscope` command table and the option surfaces of the `configure` and `run` command groups are documented
in [`references/runtime-surface.md`](references/runtime-surface.md).

---

## Workflow: adding a new runtime mode

Adding a new mode is a coordinated cross-repository change. Follow the steps in order; steps that
touch repositories outside `sollertia-experiment` are delegated via explicit handoffs.

### Step 1: Author the session descriptor (assets plugin)

Hand off to `assets:library-extension` to add the new `SessionTypes` member, create the descriptor dataclass, register
it under that member in `DESCRIPTOR_REGISTRY` in `registries.py`, export it, and bump the `sollertia-shared-assets`
version. The new descriptor MUST declare `incomplete: bool = True`, or `import sollertia_shared_assets` fails the
import-time `_assert_descriptor_contract` check.

The new member MUST also be claimed by `AcquisitionSystems.MESOSCOPE_VR` in `SYSTEM_SESSION_TYPES`, or the library
fails at import with `SYSTEM_SESSION_TYPES does not claim <NAMES>...`. When the mode runs a Unity VR task, the member
MUST additionally be added to `SESSION_TYPES_USING_VR_TASK`, since that set gates the `vr_configuration.yaml`
task-template snapshot.

The descriptor's field surface is documented in `/mesoscope-vr-session-schema`.

### Step 2: Extend the state machine (this repo)

If the new mode requires a hardware state distinct from existing states:

1. Add a new member to `MesoscopeVRStates` in `mesoscope_vr/system.py` with the next unused value.
2. Add a hardware-state method (`idle`/`rest`/`run`/`lick_train`/`run_train`-style) to
   `MesoscopeVRSystem` in `system_controller.py` that drives the hardware into the new state and
   calls `_change_system_state()`.

If the mode reuses an existing state, skip this step.

### Step 3: Add a visualizer mode (this repo)

If the new mode's display needs differ from the existing modes, add a `VisualizerMode` member in
`visualizer.py` and update `BehaviorVisualizer`'s plot-construction and update logic.

### Step 4: Author the runtime logic function (this repo)

In `mesoscope_vr/data_acquisition.py`, add a top-level `<new_mode>_logic(...)` function that builds
the `SessionData` and descriptor, constructs `MesoscopeVRSystem` (which owns its `DataLogger`), drives
the state transitions and `runtime_cycle()` loop, and tears down on completion or `KeyboardInterrupt`.
Mirror an existing
function (`lick_training_logic` for a non-trial mode, `experiment_logic` for a trial-structured mode).

### Step 5: Add the CLI command (this repo)

In `interfaces/mesoscope_vr.py`, add a new `@run.command(...)` subcommand for a session, or a
`@mesoscope.command(...)` subcommand for a standalone utility. A `@run.command` declares its per-flag Click
options and forwards them, together with the `ctx.obj` values the `run` group shares, to the new logic
function. A `@mesoscope.command` forwards only its own options. The logic function owns the `SessionData`
and descriptor construction from Step 4. Mirror an existing command.

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
- A new log message code is added to `MesoscopeVRLogMessageCodes`.
- The orchestrator's construction order, runtime cycle, or lifecycle changes.

This skill is NOT updated when:

- A descriptor's or the hardware state's field surface changes (owned by `/mesoscope-vr-session-schema`).
- The `SessionTypes` enum or its `DESCRIPTOR_REGISTRY` entry changes (owned by `assets:library-extension`).
- The generic descriptor read and write tooling changes (owned by `assets:session-descriptors`).
- Hardware composition changes (owned by `/mesoscope-vr`).
- Per-firmware-module wrapper APIs change (owned by `experiment:microcontroller-interface`).
- The Unity VR driver, MQTT topics, or trial decomposition change (owned by `experiment:vr-driver-interface`).

When in doubt, re-read the relevant source file (`mesoscope_vr/system_controller.py`,
`mesoscope_vr/system.py`, `mesoscope_vr/acquisition_components.py`, `mesoscope_vr/visualizer.py`,
`mesoscope_vr/runtime_ui.py`, `mesoscope_vr/data_acquisition.py`, `interfaces/mesoscope_vr.py`) and
reconcile this skill against ground truth.

---

## Related skills

| Skill                                   | Relationship                                                                             |
|-----------------------------------------|------------------------------------------------------------------------------------------|
| `experiment:acquisition-system-runtime` | Platform-general runtime pattern this system instantiates.                               |
| `/mesoscope-vr`                         | Hardware composition for the binding classes the runtime composes.                       |
| `experiment:acquisition-system-design`  | Platform-general static composition pattern.                                             |
| `experiment:vr-driver-interface`        | The `VRTaskDriver` the orchestrator uses for Unity coupling; MQTT + trial decomposition. |
| `experiment:microcontroller-interface`  | Per-module wrapper API the orchestrator and visualizer consume.                          |
| `assets:session-descriptors`            | Generic descriptor read and write tooling for the descriptors the runtime writes.        |
| `assets:library-extension`              | Adds the `SessionTypes` member and its `DESCRIPTOR_REGISTRY` entry a new mode needs.     |
| `assets:task-templates`                 | Authors task templates the experiment runtime loads.                                     |
| `assets:experiment-configuration`       | Authors experiment configurations the runtime loads.                                     |
| `experiment:data-management`            | Downstream session-data lifecycle (preprocess, transfer, delete).                        |
| `/mesoscope-vr-snapshots`               | Zaber and mesoscope position snapshots recorded near the session's end.                  |
| `/mesoscope-vr-session-schema`          | Field-level schema for the descriptors + hardware state this runtime populates           |
| `/mesoscope-vr-experiment-schema`       | Field-level schema for the experiment config + trial types this runtime executes         |
| `unity:gimbl-framework`                 | Unity-side framework for the VR game engine.                                             |
| `unity:mqtt-contract`                   | Unity-side MQTT topic registration.                                                      |
| `unity:task-prefabs`                    | Unity-side task prefab generation from task templates.                                   |

---

## Verification checklist

```text
When adding a new runtime mode:

Cross-repo handoffs:
- [ ] SessionTypes member added and registered in DESCRIPTOR_REGISTRY via assets:library-extension
- [ ] Descriptor dataclass authored with an 'incomplete: bool = True' field (import fails without it)
- [ ] New member claimed by AcquisitionSystems.MESOSCOPE_VR in SYSTEM_SESSION_TYPES (import fails otherwise)
- [ ] New member added to SESSION_TYPES_USING_VR_TASK if the mode runs a Unity VR task
- [ ] Descriptor field surface documented via /mesoscope-vr-session-schema
- [ ] sollertia-shared-assets version bumped

This repo (sollertia-experiment):
- [ ] MesoscopeVRStates extended in system.py (if a new state is needed)
- [ ] Hardware-state method added to MesoscopeVRSystem and routed through _change_system_state (if a new state)
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
- [ ] Session-data layout matches expectations (assets:session-data verification)
```
