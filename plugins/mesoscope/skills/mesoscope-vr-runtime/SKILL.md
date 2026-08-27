---
name: mesoscope-vr-runtime
description: >-
  Documents the Mesoscope-VR runtime behavior layer: the MesoscopeVRStates state machine, the MesoscopeVRSystem
  orchestrator, the per-mode runtime logic functions, the two control GUIs and the visualizer, the session data
  lifecycle, and the `sle mesoscope` CLI. Use when adding a training mode or session type, extending the state machine,
  changing a GUI, wiring a new CLI command, or preprocessing, purging, migrating, or deleting a Mesoscope-VR session.
user-invocable: false
---

# Mesoscope-VR runtime

Counterpart to `/mesoscope-vr`, which owns the hardware composition this runtime drives.

Mesoscope-VR is the only registered acquisition system, so this skill doubles as the worked example an agent copies when
building a new one. The platform-general rule behind each choice below lives in `experiment:acquisition-system-runtime`,
the static pattern in `experiment:acquisition-system-design`, and the seam catalog in `experiment:library-extension`.

---

## Scope

**Covers:**
- `MesoscopeVRStates` and the two state axes the orchestrator tracks
- `MesoscopeVRSystem` construction, start and stop ordering, the runtime cycle, and pause and resume
- Per-mode runtime logic functions, including `maintenance_logic`
- `BehaviorVisualizer`, `VisualizerMode`, `RuntimeControlUI`, and `MaintenanceControlUI`
- The session data lifecycle: `preprocess_session_data`, `purge_session`, `migrate_animal_between_projects`
- Session-descriptor consumption, where the runtime reads and completes what assets-plugin tooling authors
- The `sle mesoscope` CLI command surface
- The workflow for adding a training mode across sollertia-shared-assets and sollertia-experiment

**Does not cover:**
- Mesoscope-VR hardware composition, configuration dataclasses, and the system YAML. See `/mesoscope-vr`.
- The platform-general runtime pattern, and the seams a new acquisition system composes. See
  `experiment:acquisition-system-runtime` and `experiment:library-extension`.
- The six shared preprocessing primitives that this lifecycle calls. See `experiment:data-management`.
- The `SurgeryLog` and `WaterLog` processors and the sheet schema. See `experiment:google-sheets-processing`.
- The VR task driver, its MQTT vocabulary, and trial decomposition. See `experiment:vr-driver-interface`.
- Per-firmware-module wrappers and slmc `Module` classes. See `experiment:microcontroller-interface`.
- Descriptor and hardware-state field schemas. See `/mesoscope-vr-session-schema`.
- Generic descriptor tooling, and the `SessionTypes` member with its `DESCRIPTOR_REGISTRY` entry. See
  `assets:session-descriptors` and `assets:library-extension`.
- Task template and experiment configuration authoring. See `assets:task-templates` and
  `assets:experiment-configuration`.

---

## Platform seam mapping

A new acquisition system is built by substituting its own answer in the left column.

| Mesoscope-VR choice                                     | Platform seam it instantiates                                                                                                          |
|---------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `MesoscopeVRStates` plus `change_runtime_state()`       | The two state axes, system state and runtime stage                                                                                     |
| `MesoscopeVRSystem`                                     | A per-system controller, since the platform exposes no runtime base class (`sollertia-experiment/README.md`, "Extending the Platform") |
| The five per-mode logic functions                       | One logic function per acquisition mode                                                                                                |
| Every teardown step wrapped in `run_shutdown_step`      | Teardown isolation, `run_shutdown_step` in `cross_system/shutdown_tools.py`                                                            |
| `RuntimeControlUI` and `MaintenanceControlUI`           | Two daemon-process GUIs, each owning one `SharedMemoryArray`                                                                           |
| `BehaviorVisualizer`                                    | A main-thread visualizer driven by direct cycle calls                                                                                  |
| `MesoscopeVRLogMessageCodes`                            | The system's own log message code space                                                                                                |
| `mark_runtime_initialized()` and `raw_data/nk.bin`      | The initialization marker, owned by `assets:session-data`                                                                              |
| The `sle mesoscope` command group                       | One CLI group per registered acquisition system                                                                                        |
| The fifteen tools of `interfaces/mesoscope_vr_tools.py` | One `<system>_tools.py` module, discovered by filename suffix in `_register_tool_modules()` (`interfaces/mcp_server.py`)               |
| The Mesoscope-VR steps of `preprocess_session_data`     | Per-system steps around the shared preprocessing primitives                                                                            |

---

## State machine

`MesoscopeVRStates` (`mesoscope_vr/system.py`) is the `IntEnum` encoding the runtime's hardware-control mode. Each state
fixes the brake state, the screen state, and which sensors are monitoring.

| State           | Value | Meaning                                                |
|-----------------|-------|--------------------------------------------------------|
| `IDLE`          | 0     | Not conducting a session, and the state a pause enters |
| `REST`          | 1     | Rest period of an experiment session                   |
| `RUN`           | 2     | Run period of an experiment session                    |
| `LICK_TRAINING` | 3     | Lick training session                                  |
| `RUN_TRAINING`  | 4     | Run training session                                   |

`to_dict()` lowercases each member name and replaces underscores with spaces (`mesoscope_vr/system.py`). Its three call
sites all sit in `_generate_hardware_state_snapshot()`, which stores the mapping as the `system_state_codes` field of
`MesoscopeHardwareState` in the session's `hardware_state.yaml` (`mesoscope_vr/system_controller.py`). Downstream
behavior processing reads it to decode the logged system-state codes. Values 0 through 4 are taken, and a new state
SHOULD take the next unused value, 5. The system tracks two state axes. **System state** is a `MesoscopeVRStates` value
set by the hardware-state methods and logged through `_change_system_state()` with the `SYSTEM_STATE` code
(`mesoscope_vr/system_controller.py`). **Runtime state**, the within-session stage, is an integer set by
`change_runtime_state(new_state)` and logged with the `RUNTIME_STATE` code. A code outside 0 to 255 raises `ValueError`,
because the value is serialized as `uint8`, and a non-`IDLE` code is cached so a resume restores it. The training modes
stamp the stage code `_GUIDED_RUNTIME_STATE_CODE = 255`, deliberately outside the system-state range.

---

## `MesoscopeVRSystem` orchestrator

`MesoscopeVRSystem` (`mesoscope_vr/system_controller.py`) orchestrates a single session. External consumers go through
the per-mode logic functions rather than constructing the class directly.

Module constants: `_MINIMUM_CPU_COUNT = 10`, derived as three cores for the microcontrollers, one for the data logger,
four for the video systems, one for the central process, and one for the GUI. `_GUIDED_RUNTIME_STATE_CODE` and
`_MAXIMUM_RUNTIME_STATE_CODE` are both 255. `_MESOSCOPE_START_TIMEOUT_MS = 15000` and `_EXPECTED_FRAME_PULSES = 10`
together decide whether frame acquisition began. Class statics set the mesoscope frame-checking window
`_mesoscope_frame_delay = 300` ms, the `_speed_calculation_window = 50` ms, and the logging `_source_id = 1`.

### Construction

`__init__(session_data, session_descriptor, experiment_configuration=None)` resolves the system configuration through
`get_system_configuration()` and constructs its own `DataLogger`, so neither is passed in. It proceeds in this order:

1. Guards the host CPU count against `_MINIMUM_CPU_COUNT`, raising `RuntimeError` when it falls short.
2. Caches the descriptor on the public `descriptor` attribute and dumps it to `raw_data.session_descriptor_path`, so an
   unexpected termination still leaves a preprocessable session.
3. Builds `MesoscopeData` and the `_is_mesoscope_experiment` predicate the cycle consults each pass.
4. For experiment sessions only, writes a `MesoscopePositions` precursor into the session `raw_data`, seeded from the
   animal's persistent snapshot when one exists.
5. Initializes the state and tracker attributes, including `TrialState` and the resolved-stimulus counter.
6. `DataLogger(output_directory=raw_data_path, instance_name=BEHAVIOR_LOGGER_NAME, thread_count=10)`, then
   `MicroControllerInterfaces(...)` with `_wheel_encoder`, `_lick`, and `_valve` bound as hot-path aliases, then
   `VideoSystems(..., output_directory=raw_data.camera_data_path)`.
7. Warns that ZaberLauncher must already be running, blocks on Enter, loads `ZaberPositions` when the snapshot exists,
   then builds `ZaberMotors`.
8. For experiment sessions only, loads the task template through `load_vr_task_template()` and builds `VRTaskDriver`,
   while every other session type holds `None`.
9. `MesoscopeDriver` from `assets.vr_task` and `acquisition`, built for every session and connected only for experiment
   sessions, then `RuntimeControlUI` and `BehaviorVisualizer()`.

### State transitions

The orchestrator exposes one hardware-state method per system state, each ending in `_change_system_state()`:

```python
def idle(self)       -> None: ...   # IDLE, and calls change_runtime_state(IDLE) first
def rest(self)       -> None: ...   # REST
def run(self)        -> None: ...   # RUN
def lick_train(self) -> None: ...   # LICK_TRAINING, stage code 255 first
def run_train(self)  -> None: ...   # RUN_TRAINING, stage code 255 first
```

Each method is convergent and unconditional. It issues every actuator and monitoring command for its target state and
re-logs the state on re-entry, rather than checking whether the system already sits there. The short-circuit lives one
layer down, in the per-actuator wrappers, where a setter such as `BrakeInterface.set_state` returns early when the
request matches its own cached state (`cross_system/module_interfaces.py`).

### Start and stop ordering

`start()` runs thirteen steps. It starts the logger and logs the onset timestamp at `acquisition_time=0`, starts the
microcontrollers, enters `idle()`, and writes the hardware-state and system-configuration snapshots. For VR sessions it
applies the encoder Unity scale, connects, brackets `vr_task.setup()` with the screens on and off, seeds the trial
structures, and resets the distance and trial counters. It then starts both cameras and runs `setup_zaber_motors()`, and
for experiment sessions connects the mesoscope driver and runs `setup_mesoscope()`. It resolves the `VisualizerMode` and
the reinforcing, aversive, and mesoscope flags, starts the control GUI, and pushes the initial guidance state to Unity.
It opens the visualizer, which MUST follow the cameras and the GUI to avoid Qt backend collisions, then runs
`_checkpoint()` and returns early with `_started = True` when the operator aborted there. Finally it starts frame saving
on both cameras and, for experiment sessions, enables frame monitoring, settles one second, and calls
`_start_mesoscope()`.

`stop()` delegates to `_emergency_shutdown()` and returns when `start()` never completed. Otherwise it clears `_started`
and runs each teardown step through `run_shutdown_step(description, step)`, which is defined in
`cross_system/shutdown_tools.py`, re-exported from `cross_system`, and imported from there by the orchestrator. That
helper catches `(Exception, KeyboardInterrupt)` and echoes an ERROR, so a failing step never skips the ones after it.
The order is `idle()`, the control GUI, the visualizer, the VR task, and the cameras. It continues with the mesoscope
stop that carries the frame-monitoring disable and `rename_mesoscope_directory`, the Zaber snapshot, the session
descriptor, the mesoscope position snapshot, the ScanImagePC disconnect, `reset_zaber_motors`, the microcontrollers, and
the logger. The three artifact-writing steps deliberately follow the hardware steps, so a failed hardware teardown still
leaves the snapshots and the descriptor on disk. Unless the `nk.bin` marker survives, `stop()` ends by asking the
operator to choose among `preprocess`, `skip preprocessing`, and `purge session`. `_emergency_shutdown()` never prompts
and runs the visualizer, the GUI, the VR task, the cameras, the mesoscope, a plain Zaber `disconnect()` with no
interactive parking, the microcontrollers, and the logger.

### Runtime cycle

`runtime_cycle()` is the per-iteration heartbeat the logic functions call in a loop. Every iteration runs
`_data_cycle()`, `self._visualizer.update()`, and `_ui_cycle()`, returns early once `terminated` is set, and, for
experiment sessions only, then runs `_unity_cycle()` and `_mesoscope_cycle()`. It returns after one pass while the
runtime is running, and loops in place while the runtime is paused, which suspends the outer logic loop without
unwinding it.

- `_data_cycle()` reads the traveled distance and recomputes the running speed over the window that **actually** elapsed
  rather than the nominal 50 ms, because a blocking call inside the cycle stretches the real window. For VR sessions it
  pushes position to Unity, advances the distance-driven trial counter, arms per-type recovery guidance, forwards lick
  increments, and splits newly dispensed water between the paused and delivered totals.
- `_ui_cycle()` reads the pause flag once and routes it to `_pause_runtime` or `_resume_runtime`, then services the
  exit, reward, and gas-puff signals. It consumes `generate_reference_signal` every pass while acting on it only once
  the mesoscope has terminated, and syncs the guidance flags to the VR driver.
- `_mesoscope_cycle()` returns while `_mesoscope_timer.elapsed` is under 300 ms or the mesoscope already terminated, and
  otherwise refreshes the frame count when pulses advance, or logs acquisition off, pauses the runtime, stops the
  mesoscope, and re-enables the reference button.

Window-checking and maintenance runtimes stay outside the cycle, the first running a linear sequence of blocking
operator prompts and the second running its own loop driven by the maintenance GUI.

### Pause, resume, and terminate

`_pause_runtime()` mirrors the pause into the GUI, returns without restarting the pause clock when the runtime is
already paused, stamps `_pause_start_time`, calls `idle()`, and sets `_paused`. `_resume_runtime()` re-arms Unity
through `resume_after_unity_restart()` when Unity terminated, and recovers and restarts the mesoscope when the mesoscope
terminated, re-engaging the pause on a `RuntimeError` instead of propagating it. It then accumulates `paused_time` in
seconds, restores the cached runtime stage, and re-enters the pre-pause system state. `_terminate_runtime()` gates
`_terminated` behind a terminal `request_confirmation(default=False)`, so a misclick on the GUI terminate button cannot
end a session on its own. `_checkpoint()` is the pre-start loop that runs while the GUI reports a paused runtime,
servicing reward delivery, water-valve open and close, gas-valve open, close, and puff, reference regeneration, guidance
sync, and exit. On exit it closes the valve, folds the pre-start water into `_paused_water_volume`, calls
`set_setup_complete()`, and disables the reference button.

### Trial-state tracking

`TrialState` (`mesoscope_vr/acquisition_components.py`) tracks reinforcing (water reward) and aversive (gas puff) trials
separately, holding the completed trial count, the per-trial cumulative distances, per-type guided-trial and
consecutive-failure counters, per-type recovery thresholds, the in-flight outcome flags, and the `trial_structures`
mapping. `trial_completed(traveled_distance)` reports `False` once every decomposed trial is consumed, and
`advance_trial()` returns the updated per-type failure count. `setup_reinforcing_guidance()` and
`setup_aversive_guidance()` configure guidance, and `_refresh_trial_state_from_vr_decomposition()` rebuilds the
per-trial parameter arrays from the ordered trial names the `VRTaskDriver` produces by decomposing the active Unity cue
sequence. `_build_trial_parameter_arrays(trial_names)` raises `ValueError` when a decomposed name matches no configured
trial structure. Adding a trial-tracking dimension means extending `TrialState` and updating the stimulus handling in
`_unity_cycle()`.

### Log message codes

`MesoscopeVRLogMessageCodes` (`mesoscope_vr/acquisition_components.py`) defines the event codes the orchestrator emits
to the `DataLogger`:

| Code | Name                          | Meaning                                                          |
|------|-------------------------------|------------------------------------------------------------------|
| 1    | `SYSTEM_STATE`                | The system changed its hardware-control (system) state           |
| 2    | `RUNTIME_STATE`               | The acquired session changed its runtime state (stage)           |
| 3    | `REINFORCING_GUIDANCE_STATE`  | The reinforcing-trial guidance state changed                     |
| 4    | `AVERSIVE_GUIDANCE_STATE`     | The aversive-trial guidance state changed                        |
| 5    | `DISTANCE_SNAPSHOT`           | Total traveled distance at Unity-signaled runtime termination    |
| 6    | `MESOSCOPE_ACQUISITION_STATE` | Whether the ScanImagePC is expected to be writing session frames |

A new event gets the next unused code, 7. Downstream behavior processing in the forging plugin consumes these codes.

---

## Unity coupling

For experiment sessions the orchestrator couples to Unity through a `VRTaskDriver` held on `_vr_task`. It pushes motion
and lick events to the driver and consumes at most one typed `VRTaskEvent` per `_unity_cycle()`. The driver owns the
MQTT broker connection, the topic vocabulary, scene and cue verification, cue-sequence trial decomposition, and the
editor bridge that opens the scene and controls Play Mode, so `start()` only brackets `setup()` with the VR-screen
enable and disable.

| Event kind                | Dispatch                                                                                                                 |
|---------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `STIMULUS_TRIGGERED`      | Resolves the trial position from `_resolved_stimulus_count` and dispatches the outcome, as broken out below              |
| `TRIGGER_DELAY_REQUESTED` | `brake.send_pulse(duration_ms=event.delay_ms)` when the delay is positive                                                |
| `UNITY_TERMINATED`        | Enters an emergency pause, echoes an error, and logs a `DISTANCE_SNAPSHOT` packet carrying the float64 traveled distance |

The `STIMULUS_TRIGGERED` dispatch runs five steps in order:

- Resolves the trial position from `_resolved_stimulus_count`.
- Discards an event past the decomposed trial count, with a warning.
- Delivers the puff or the reward when `delivered` is set.
- Decrements the per-type guided counter.
- Reports the outcome to the visualizer.

The trial position comes from the resolved-stimulus counter rather than the distance-driven counter, because the data
cycle advances the distance counter before the Unity cycle dequeues the event the finished trial produced. The
`STIMULUS_TRIGGERED` branch of `_unity_cycle()` counts an aversive trial a success when the puff was **not** delivered,
and a reinforcing trial a success when the reward was delivered. On `UNITY_TERMINATED` the resume path calls
`resume_after_unity_restart()`, which re-arms Unity through the bridge and re-fetches the cue sequence, so the operator
does not press the play button. The driver, its event model, the MQTT contract, the editor bridge, and trial
decomposition are documented in `experiment:vr-driver-interface`, and the Unity side in `unity:gimbl-framework`,
`unity:mqtt-contract`, `unity:play-mode`, `unity:task-scenes`, and `unity:scene-setup`.

---

## Detailed surfaces

[`references/runtime-surface.md`](references/runtime-surface.md) carries the enumerations this file summarizes. Those
are the per-mode logic function sequence with its function table and descriptor consumption pattern, and the
`sle mesoscope` command table with every option surface. The same file holds the shared-memory index maps, prototype
defaults, and per-control tables of both GUIs and the visualizer.

---

## Session data lifecycle

`preprocess_session_data(session_data)` (`mesoscope_vr/data_preprocessing.py`) is the Mesoscope-VR orchestration around
the shared primitives that `experiment:data-management` owns. A session whose `nk.bin` marker survives never finished
initialization, so it is purged instead of preprocessed, and each destination listed in
`MesoscopeData.unconfigured_destinations` produces one WARNING.

| Order | Step                                                              | Owner                       |
|-------|-------------------------------------------------------------------|-----------------------------|
| 1     | `rename_mesoscope_directory(mesoscope_data)`                      | Mesoscope-VR                |
| 2     | `assemble_session_logs(session_data, processes=...)`              | `cross_system`              |
| 3     | `rename_session_videos(session_data)`                             | `cross_system`              |
| 4     | `_launch_face_tracking(...)`, experiment sessions only, async     | Mesoscope-VR                |
| 5     | `_pull_mesoscope_data(...)`                                       | Mesoscope-VR                |
| 6     | `_preprocess_mesoscope_directory(...)`                            | Mesoscope-VR                |
| 7     | `_preprocess_google_sheet_data(...)`                              | Mesoscope-VR Sheets wrapper |
| 8     | `_purge_window_checking_behavior_data(...)`, window checking only | Mesoscope-VR                |
| 9     | `_join_face_tracking(...)`                                        | Mesoscope-VR                |
| 10    | `push_session_data(session_data, destinations, threads=15)`       | `cross_system`              |

Steps 5 through 8 run inside a `try` whose `except BaseException` calls `_terminate_face_tracking(...)` and re-raises,
so an abort never abandons the child holding the GPU. Constants: `_PREPROCESSING_WORKER_COUNT` is
`resolve_worker_count(reserved_cores=1)`, `_STORAGE_TRANSFER_THREAD_COUNT = 15`,
`_FACE_TRACKING_TERMINATION_TIMEOUT = 30.0` seconds, and `_INFERENCE_LOG_TAIL_CHARACTERS = 2000`.

### Mesoscope-VR-only steps

- `rename_mesoscope_directory` renames the shared `mesoscope_data` directory to the session-specific path, only when the
  session path is absent and the shared path holds files, then recreates an empty shared directory.
- `_launch_face_tracking` returns `None` when either `conda_environment` or `dlc_project_path` is unset, or when the
  face-camera video is missing. Otherwise it runs `conda run -n <env> slvt infer` with `--config-path`, `--videos`,
  `--shuffle`, `--device cuda`, `--gpus 0`, `--batch-size`, `--chunks`, `--compile-model`, `--no-progress`, and `--crop`
  when configured. It writes predictions beside the video in raw `camera_data` and redirects output to a temporary log
  file rather than a pipe, because a full pipe buffer would deadlock the long-running child.
- `_join_face_tracking` waits for the child, then raises `RuntimeError` when the exit code is non-zero or no `.h5`
  prediction file sits beside the video, which aborts the transfer and retains the local copy for a retry. The transient
  log is removed on success and retained on failure, and the failure message carries its tail.
- `_pull_mesoscope_data` raises `RuntimeError` unless `MotionEstimator.me`, `fov.roi`, and `zstack.tiff` are all
  present, strips `*.bin` markers, creates `raw_data/raw_mesoscope_frames` only after that verification, and then
  transfers with `remove_source=True`.
- `_preprocess_mesoscope_directory` re-verifies the same three files, seeds the animal's persistent ScanImagePC
  `fov.roi` and `MotionEstimator.me` when absent, copies all three into the session `mesoscope_data` directory, and
  emits `frame_invariant_metadata.json`, `frame_variant_metadata.npz`, and `cindra_parameters.json` alongside the
  LERC-recompressed frame stacks.
- `_preprocess_google_sheet_data` returns early with a WARNING when neither sheet id is set, otherwise resolves
  `get_credentials(CredentialsTypes.GOOGLE)`, validates the session type against `MESOSCOPE_VR_SESSIONS`, and loads the
  descriptor through `DESCRIPTOR_REGISTRY`. Window-checking sessions call `update_surgery_quality` with the descriptor
  value clamped into 0 to 3, every other session type writes the water log entry from the animal weight and the summed
  training and experimenter-given volumes, and both handles close in a `finally`.

### Purge and migration

`purge_session(session_data)` builds its candidate set from the local session parent, every configured storage
destination's session path, and the ScanImagePC session-specific path, then delegates to
`delete_session_directories(..., require_confirmation=not nk_path.exists())`. A declined confirmation returns without
further change, and a completed deletion also clears residual files from the shared ScanImagePC `mesoscope_data`
directory.

`migrate_animal_between_projects(animal, source_project, target_project)` raises `FileNotFoundError` when the target
project is absent, then picks one of two strategies. With no configured storage destination it relocates each locally
stored session on premises. With at least one, the first configured destination becomes the source of truth, and each
session is pulled from that destination, re-preprocessed, and purged against it. Both strategies then relocate the
ScanImagePC persistent directory at `mesoscope_directory/<project>/<animal>` and the VRPC persistent directory, and
delete the redundant `<root>/<source_project>/<animal>` directory under the mesoscope mount, the data root, and every
configured storage root.

`sle mesoscope delete` runs the data-root containment check and then delegates to `purge_session` (the `delete` command
in `interfaces/mesoscope_vr.py`). It therefore prompts for an interactive confirmation on any session whose `nk.bin`
marker is already cleared, and deletes without a prompt only for a session that never finished initializing. The MCP
`delete_session_tool` refuses to act without an explicit `confirm_deletion`, returning an `Error:` string when it is
`None` and an abandonment notice when it is `"no"` (`interfaces/mesoscope_vr_tools.py`). You MUST route an
agent-initiated deletion through the MCP tool and warn the user before passing `"yes"`.

---

## Runtime GUIs and the visualizer

`RuntimeControlUI` (`mesoscope_vr/runtime_ui.py`) is the session GUI, running in a daemon process and backed by one
20-slot `SharedMemoryArray` it owns plus the two valve trackers it reads and does not own. Its prototype sets
`PAUSE_STATE = 1`, so every runtime starts paused and enters the pre-start checkpoint. A signal-style slot is cleared by
the property that reads it, under the array lock, while a value-style slot is not.

`BehaviorVisualizer` (`mesoscope_vr/visualizer.py`) renders lick, valve, puff, running-speed, and trial-performance data
with matplotlib on a `QtAgg` backend, driven by a `_BlitManager` for partial redraws. It runs in the main thread of the
runtime control process and updates through direct calls from `runtime_cycle()`, with no IPC.

`RUN_TRAINING_THRESHOLD_LIMITS`, a frozen `RunTrainingThresholdLimits` in `mesoscope_vr/system.py`, fixes the
run-training speed bounds at 0.1 to 5.0 cm/s and the duration bounds at 0.05 to 5.0 s. The run-training logic clamps the
effective thresholds to these bounds and the GUI constrains its spin boxes to them, so an out-of-range request is
silently clamped rather than rejected.

The orchestrator and the visualizer read four `SharedMemoryArray`-backed properties off the binding classes,
`lick.lick_count`, `valve.delivered_volume`, `wheel_encoder.absolute_position`, and `wheel_encoder.traveled_distance`.
Each read is safe from the main process because a communication subprocess owns the write side. See
`experiment:microcontroller-interface` for the wrapper lifecycle behind that pattern.

### Maintenance runtime

`maintenance_logic()` (`mesoscope_vr/data_acquisition.py`) runs no session. It asks whether to position the Zaber
motors, then works inside a `tempfile.TemporaryDirectory(prefix="sl_maintenance_")` whose contents are discarded, builds
a `DataLogger(instance_name="temporary")`, and stands up only `WaterValveInterface`, `GasPuffValveInterface`, and
`BrakeInterface` on a standalone `MicroControllerInterface(controller_id=101, buffer_size=8192, name="actor")`. When the
operator opts in, it homes the motors and moves them to `maintenance_position()` behind a warning to remove the
objective, swivel out the screens, and confirm the animal is not mounted. It then starts `MaintenanceControlUI` and
loops on the GUI signals until `exit_signal`, erroring out when the GUI process dies without setting it. Its `finally`
block parks the motors and stops the controller, the logger, and the GUI, with each step isolated.

`MaintenanceControlUI` (`mesoscope_vr/maintenance_ui.py`) owns a 14-slot `SharedMemoryArray` and exposes seventeen API
members. Valve calibration, valve referencing, brake testing, and motor positioning are experimenter-operated through
this GUI. You MUST NOT drive them on the operator's behalf, and you MUST NOT issue any mesoscope hardware command.

---

## Workflow: adding a new runtime mode

Adding a mode is a coordinated cross-repository change. Follow the steps in order. Steps that touch repositories outside
sollertia-experiment are delegated through explicit handoffs.

**Step 0, read the seam catalog.** Hand off to `experiment:library-extension` for the configuration-registry,
`cross_system`, MCP, and CLI seams a new mode or a new system touches, and for the absent runtime base class.

**Step 1, author the session descriptor.** Hand off to `assets:library-extension`, which owns the full touch list for
adding a `SessionTypes` member. The one Mesoscope-VR-specific point is that the new member MUST be claimed by
`AcquisitionSystems.MESOSCOPE_VR` in `SYSTEM_SESSION_TYPES` to be runnable here. The descriptor's field surface is
documented in `/mesoscope-vr-session-schema`.

**Step 2, extend the state machine.** When the mode needs a hardware state distinct from the existing five, add a
`MesoscopeVRStates` member in `mesoscope_vr/system.py` with the next unused value. Then add a convergent hardware-state
method to `MesoscopeVRSystem` that drives every actuator and monitoring flag and ends in `_change_system_state()`.

**Step 3, add a visualizer mode.** When the mode's display needs differ from the existing three, add a `VisualizerMode`
member in `mesoscope_vr/visualizer.py` and update the plot-construction and update logic in `BehaviorVisualizer`.

**Step 4, author the runtime logic function.** In `mesoscope_vr/data_acquisition.py`, add a top-level
`<new_mode>_logic(...)` that mints the `SessionData` and the descriptor, constructs `MesoscopeVRSystem`, calls
`mark_runtime_initialized()` once the hardware is up, drives the state transitions and the `runtime_cycle()` loop, and
tears down in a `finally` block. Mirror `lick_training_logic` for a mode without trials and `experiment_logic` for a
trial-structured mode.

**Step 5, add the CLI command.** In `interfaces/mesoscope_vr.py`, add a `@run.command(...)` for a session or a
`@mesoscope.command(...)` for a standalone utility. A `@run.command` declares its own defaultless options and forwards
them together with the `_SharedSessionParameters` values the `run` group shares.

**Step 6, export, bump, and document.** Re-export the logic function from `mesoscope_vr/__init__.py` when external
consumers need it, bump `sollertia-experiment` in `pyproject.toml`, then raise the `sollertia-shared-assets` pin to the
version carrying the new descriptor. Update the state and log message code tables here and the function, CLI, and GUI
tables in [`references/runtime-surface.md`](references/runtime-surface.md).

---

## Maintenance contract

Update this skill when a `MesoscopeVRStates` member, a hardware-state method, a runtime logic function, a
`VisualizerMode` member, a `MesoscopeVRLogMessageCodes` member, a `sle mesoscope` command or option, or a GUI
shared-memory slot changes. Update it as well when the orchestrator's construction order, runtime cycle, teardown order,
or preprocessing step order changes. Leave it untouched for anything the Scope section delegates, because each of those
concerns is versioned by its owning skill.

When in doubt, re-read the source (`mesoscope_vr/system_controller.py`, `mesoscope_vr/system.py`,
`mesoscope_vr/acquisition_components.py`, `mesoscope_vr/data_acquisition.py`, `mesoscope_vr/data_preprocessing.py`,
`mesoscope_vr/visualizer.py`, `mesoscope_vr/runtime_ui.py`, `mesoscope_vr/maintenance_ui.py`,
`interfaces/mesoscope_vr.py`) and reconcile this skill against it.

---

## Related skills

| Skill                                   | Relationship                                                                          |
|-----------------------------------------|---------------------------------------------------------------------------------------|
| `experiment:acquisition-system-runtime` | Platform-general runtime pattern this system instantiates                             |
| `experiment:library-extension`          | Catalogs the sollertia-experiment seams a new system's runtime composes               |
| `/mesoscope-vr`                         | Hardware composition for the binding classes the runtime drives                       |
| `experiment:acquisition-system-design`  | Platform-general static composition pattern                                           |
| `experiment:vr-driver-interface`        | The `VRTaskDriver` the orchestrator uses for Unity coupling and trial decomposition   |
| `experiment:microcontroller-interface`  | Per-module wrapper API the orchestrator and the visualizer consume                    |
| `experiment:data-management`            | The shared preprocessing primitives this lifecycle calls into                         |
| `experiment:google-sheets-processing`   | The `SurgeryLog` and `WaterLog` processors the Sheets step invokes                    |
| `assets:session-data`                   | `SessionData.create`, the `nk.bin` marker, and the session hierarchy                  |
| `assets:session-descriptors`            | Generic descriptor read and write tooling for the descriptors the runtime writes      |
| `assets:library-extension`              | Adds the `SessionTypes` member and its `DESCRIPTOR_REGISTRY` entry a new mode needs   |
| `assets:task-templates`                 | Authors the task templates the experiment runtime loads                               |
| `assets:experiment-configuration`       | Authors the experiment configurations the runtime loads                               |
| `/mesoscope-vr-snapshots`               | Zaber and mesoscope position snapshots recorded near the session's end                |
| `/mesoscope-vr-session-schema`          | Field-level schema for the descriptors and hardware state this runtime populates      |
| `/mesoscope-vr-experiment-schema`       | Field-level schema for the experiment configuration and trial types this runtime runs |
| `forging:batch-processing`              | Runs the runtime pipeline that decodes the message codes this runtime logs            |
| `unity:gimbl-framework`                 | Unity-side framework for the VR game engine                                           |
| `unity:mqtt-contract`                   | Unity-side MQTT topic registration                                                    |
| `unity:task-prefabs`                    | Unity-side task prefab generation from task templates                                 |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Cross-repo handoffs for a new runtime mode:
- [ ] experiment:library-extension consulted for the seam touch list
- [ ] assets:library-extension checklist completed for the new SessionTypes member, and its version bumped
- [ ] Descriptor field surface documented via /mesoscope-vr-session-schema

This repo (sollertia-experiment):
- [ ] MesoscopeVRStates extended and a convergent hardware-state method routed through _change_system_state
- [ ] VisualizerMode extended and BehaviorVisualizer updated, if the display needs differ
- [ ] Logic function added to data_acquisition.py, calling mark_runtime_initialized()
- [ ] Every teardown step routed through run_shutdown_step
- [ ] CLI command registered in interfaces/mesoscope_vr.py and the logic function re-exported if needed
- [ ] sollertia-experiment version bumped and the sollertia-shared-assets pin raised

Documentation and operation:
- [ ] State, log message code, and platform seam mapping tables updated in this skill
- [ ] Logic function, CLI, and GUI tables updated in references/runtime-surface.md
- [ ] Hardware verified via experiment:acquisition-system-setup before running a session
- [ ] Calibration, referencing, and motor positioning left to the experimenter at the maintenance GUI
```
