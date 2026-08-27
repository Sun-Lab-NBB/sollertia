# Mesoscope-VR runtime surface

Enumerates the per-mode runtime logic functions, the `sle mesoscope` CLI commands that invoke them, and the
shared-memory surfaces of the two runtime GUIs. See [`../SKILL.md`](../SKILL.md) for the state machine, the
orchestrator, the session data lifecycle, and the workflow for adding a new runtime mode.

---

## Per-mode runtime logic functions

Each runtime mode has a top-level function in `mesoscope_vr/data_acquisition.py` that:

1. Resolves the system configuration through `get_system_configuration()`, verifies the project and the animal's project
   membership through `_verify_project_configured()` and `_verify_animal_project_membership()`, reads the version data
   through `get_version_data()`, and mints the session through
   `SessionData.create(..., acquisition_system=AcquisitionSystems.MESOSCOPE_VR)`.
2. Builds the session-specific descriptor from default values, then the previous same-type session's parameters when one
   exists, then the per-flag CLI overrides, in that order. `experiment_logic` additionally loads the
   `MesoscopeExperimentConfiguration` from `raw_data.experiment_configuration_path` through its `from_yaml()` builder.
3. Builds the hardware assets the mode needs. `lick_training_logic`, `run_training_logic`, and `experiment_logic`
   construct `MesoscopeVRSystem`, which owns and starts its own `DataLogger`. `window_checking_logic` and
   `maintenance_logic` construct their own `DataLogger` and hardware assets directly, with no orchestrator.
4. Marks the session as initialized. Every session-running function, once its hardware assets are up and the session is
   ready to acquire, calls `session_data.mark_runtime_initialized()`, which removes the `nk.bin` marker from the
   session's `raw_data` directory. This is a different signal from the descriptor's `incomplete` flag, which the runtime
   sets at session end. See `assets:session-data` for the marker's semantics. `maintenance_logic` runs no session and
   never performs this step.
5. Runs the session's control loop. The three `MesoscopeVRSystem` modes drive state transitions and call
   `runtime_cycle()` each iteration, folding `system.paused_time` into the mode's own timing budget.
   `lick_training_logic` subtracts it from each reward delay and zeroes it after every delay cycle, while
   `run_training_logic` and `experiment_logic` add it to the budget so a pause extends the session, with only
   `experiment_logic` zeroing it between experiment states.
6. Tears down in a `finally` block that isolates each step and purges a session whose marker survives.

Current functions:

| Function                | Purpose                                                                          |
|-------------------------|----------------------------------------------------------------------------------|
| `window_checking_logic` | Cranial window quality session, no behavior data and no orchestrator             |
| `lick_training_logic`   | Lickport-only training (animal learns to operate the lickport for water rewards) |
| `run_training_logic`    | Wheel-only training (animal learns to run for water rewards)                     |
| `experiment_logic`      | Full Mesoscope-VR experiment session with VR trial structure                     |
| `maintenance_logic`     | Hardware maintenance GUI (valve calibration, referencing, brake, motor position) |

Every session-running function takes the experimenter, project, and animal identifiers and builds the `SessionData` and
the descriptor internally. `lick_training_logic`, `run_training_logic`, and `experiment_logic` additionally take the
animal weight and the per-flag parameter overrides, and `experiment_logic` also takes `experiment_name`.
`window_checking_logic` takes exactly the three identifiers `experimenter`, `project_name`, and `animal_id`, all in
`mesoscope_vr/data_acquisition.py`. `maintenance_logic()` takes no arguments.

All three `MesoscopeVRSystem` logic functions convert an operator abort at the pre-start checkpoint into a
`RecursionError` so control jumps straight to the teardown block. `run_training_logic` inherits the previous session's
`final_run_speed_threshold_cm_s` and `final_run_duration_threshold_s` as its new initial thresholds and clamps every
effective threshold to `RUN_TRAINING_THRESHOLD_LIMITS`. `experiment_logic` pre-validates that every experiment state's
`system_state_code` is `REST` or `RUN` before starting the hardware. All three functions live in
`mesoscope_vr/data_acquisition.py`.

### Session descriptor consumption

The descriptor dataclasses are owned by `sollertia-shared-assets`. Their field surface is documented in
`/mesoscope-vr-session-schema`, and the generic descriptor read and write tools in `assets:session-descriptors`.

The runtime both consumes and completes the descriptor:

- At construction the orchestrator caches the partially configured descriptor to the session's `raw_data` directory, so
  the session can be preprocessed even if the runtime terminates unexpectedly (`MesoscopeVRSystem.__init__` in
  `mesoscope_vr/system_controller.py`).
- During runtime it records runtime-discovered values into the descriptor in place. Those are the dispensed and
  pause-dispensed water volumes, the pre-filled experimenter-delivered water volume, the `incomplete` flag, and, for
  run-training sessions, the final speed and duration thresholds the operator sets through the control GUI
  (`update_visualizer_thresholds()` and `_generate_session_descriptor()` in `mesoscope_vr/system_controller.py`).
- At session end `_generate_session_descriptor()` calls `finalize_session_descriptor(...)`, which collects the
  experimenter notes through a blocking terminal prompt and, for window-checking sessions, a 0 to 3 cranial-window
  quality rating. It stores both on the descriptor, writes the completed descriptor to the session's `raw_data`
  directory, and copies it to the animal's persistent directory, where the next session of the same type reads it back
  (`mesoscope_vr/acquisition_components.py`).

---

## CLI command surface

The `sle mesoscope` command group, every one of its Click nodes, and all thirty-three of its options are documented by
`/mesoscope-vr-cli-reference`. This file covers only what each command does once it starts, which is the per-mode
logic above and the GUI surfaces below.

---

## Runtime control GUI surface

`RuntimeControlUI` (`mesoscope_vr/runtime_ui.py`) owns one `SharedMemoryArray` named `"runtime_control_ui"`, shaped
`(20,)` and typed `np.int32`, built in its `__init__`. It also holds two read-only trackers it does **not** own, the
`WaterValveInterface` valve tracker and the `GasPuffValveInterface` puff tracker, so `shutdown()` destroys its own array
and deliberately leaves the trackers connected. The window runs in a daemon `multiprocessing.Process` that `start()`
spawns, and costs one CPU core.

| Index | Member                         | Index | Member                       |
|-------|--------------------------------|-------|------------------------------|
| 0     | `TERMINATION`                  | 10    | `AVERSIVE_GUIDANCE_ENABLED`  |
| 1     | `EXIT_SIGNAL`                  | 11    | `GAS_VALVE_OPEN`             |
| 2     | `REWARD_SIGNAL`                | 12    | `GAS_VALVE_CLOSE`            |
| 3     | `SPEED_MODIFIER`               | 13    | `GAS_VALVE_PUFF`             |
| 4     | `DURATION_MODIFIER`            | 14    | `GAS_VALVE_PUFF_DURATION`    |
| 5     | `PAUSE_STATE`                  | 15    | `SETUP_COMPLETE`             |
| 6     | `OPEN_VALVE`                   | 16    | `RUNTIME_SPEED_THRESHOLD`    |
| 7     | `CLOSE_VALVE`                  | 17    | `RUNTIME_DURATION_THRESHOLD` |
| 8     | `REWARD_VOLUME`                | 18    | `GENERATE_REFERENCE`         |
| 9     | `REINFORCING_GUIDANCE_ENABLED` | 19    | `REFERENCE_BUTTON_ENABLED`   |

The `__init__` prototype sets `PAUSE_STATE = 1`, so **every runtime starts paused**, both guidance flags to `0`,
`REWARD_VOLUME = 5`, `GAS_VALVE_PUFF_DURATION = 100`, and `REFERENCE_BUTTON_ENABLED = 1`. `RUNTIME_SPEED_THRESHOLD` is
stored in hundredths of a centimeter per second and `RUNTIME_DURATION_THRESHOLD` in milliseconds, through
`_SPEED_THRESHOLD_SCALE = 100` and `_DURATION_THRESHOLD_SCALE = 1000`. One modifier increment is
`_MODIFIER_STEP = 0.01`, the window polls external state every `_UI_REFRESH_INTERVAL_MS = 100`, and the exit-request
feedback reverts after `_EXIT_FEEDBACK_DELAY_MS = 2000`.

A signal-style slot is cleared by the property that reads it, under the array lock. Those are `exit_signal`,
`reward_signal`, `generate_reference_signal`, `open_valve`, `close_valve`, `gas_valve_open_signal`,
`gas_valve_close_signal`, and `gas_valve_puff_signal`. A value-style slot is not cleared by its reader. Those are
`speed_modifier`, `duration_modifier`, `pause_runtime`, `reward_volume`, `enable_reinforcing_guidance`,
`enable_aversive_guidance`, and `gas_valve_puff_duration`.

`start(mode=VisualizerMode.EXPERIMENT, *, has_reinforcing_trials=True, has_aversive_trials=True, has_mesoscope=False)`
spawns the window and rolls the spawned process back when the connect fails. `_ControlUIWindow.__init__` titles the
window `"Mesoscope-VR Control Panel"` and fixes it at 450 pixels wide, with its height grown per enabled control group.

| Control                                                                                       | Visible when                      | Writes                                     |
|-----------------------------------------------------------------------------------------------|-----------------------------------|--------------------------------------------|
| `✖ Terminate Runtime`                                                                         | always                            | `EXIT_SIGNAL`, with 2 s transient feedback |
| `▶️ Resume Runtime` / `⏸️ Pause Runtime`                                                      | always                            | `PAUSE_STATE`                              |
| Reinforcing guidance toggle                                                                   | EXPERIMENT and reinforcing trials | `REINFORCING_GUIDANCE_ENABLED`             |
| Aversive guidance toggle                                                                      | EXPERIMENT and aversive trials    | `AVERSIVE_GUIDANCE_ENABLED`                |
| `🔬 Regenerate Reference`                                                                      | `has_mesoscope`                   | `GENERATE_REFERENCE`                       |
| `Runtime Status:` label                                                                       | always                            | display only                               |
| `🔓 Open` / `🔒 Close` water valve                                                              | always, disabled once setup ends  | `OPEN_VALVE` / `CLOSE_VALVE`               |
| `● Reward`                                                                                    | always                            | `REWARD_SIGNAL`                            |
| Reward volume spinbox, 1 to 20 µL, default 5                                                  | always                            | `REWARD_VOLUME`                            |
| `Valve:` status label                                                                         | always                            | display only                               |
| Gas puff group: open and close (disabled once setup ends), puff, 10 to 350 ms spinbox, status | EXPERIMENT and aversive trials    | the four `GAS_VALVE_*` slots               |
| Speed threshold spinbox, 0.1 to 5.0 cm/s, step 0.01                                           | RUN_TRAINING only                 | `SPEED_MODIFIER`                           |
| Duration threshold spinbox, 0.05 to 5.0 s, step 0.01                                          | RUN_TRAINING only                 | `DURATION_MODIFIER`                        |

The water-valve and gas-valve `🔓 Open` and `🔒 Close` buttons are all permanently disabled once the runtime calls
`set_setup_complete()` at the end of the pre-start checkpoint, which reaches the buttons through
`_ControlUIWindow._disable_valve_open_close_buttons()`. A 100 ms `QTimer` runs `_check_external_state`, which honors
`TERMINATION` and mirrors external pause, guidance, setup-complete, and reference-enabled changes. It also re-syncs both
run-training spinboxes through `_sync_run_training_spinbox` unless the spinbox has focus, and clears the reward and puff
status labels by watching the **cumulative** tracker totals rather than the live open state.

`runtime_ui.py` also exports the three terminal prompt helpers the teardown path uses, `collect_experimenter_notes`,
`collect_surgery_quality`, and `collect_experimenter_given_water_volume`.

---

## Maintenance control GUI surface

`MaintenanceControlUI` (`mesoscope_vr/maintenance_ui.py`) owns one `SharedMemoryArray` named `"maintenance_control_ui"`,
shaped `(14,)` and typed `np.uint32`, plus the same two externally owned trackers, all built in its `__init__`. It runs
in a daemon process and costs one CPU core.

| Index | Member            | Index | Member                       |
|-------|-------------------|-------|------------------------------|
| 0     | `TERMINATION`     | 7     | `BRAKE_UNLOCK`               |
| 1     | `VALVE_OPEN`      | 8     | `REWARD_VOLUME`              |
| 2     | `VALVE_CLOSE`     | 9     | `CALIBRATION_PULSE_DURATION` |
| 3     | `VALVE_REWARD`    | 10    | `GAS_VALVE_OPEN`             |
| 4     | `VALVE_REFERENCE` | 11    | `GAS_VALVE_CLOSE`            |
| 5     | `VALVE_CALIBRATE` | 12    | `GAS_VALVE_PULSE`            |
| 6     | `BRAKE_LOCK`      | 13    | `GAS_VALVE_PULSE_DURATION`   |

Spinbox ranges and defaults are 1 to 20 µL default 5 for the reward volume, 1 to 200 ms default 30 for the calibration
pulse, and 10 to 350 ms default 100 for the gas puff. The monitor `QTimer` runs every 100 ms. Those come from the
`_REWARD_VOLUME_RANGE`, `_DEFAULT_REWARD_VOLUME`, `_CALIBRATION_PULSE_DURATION_RANGE`,
`_DEFAULT_CALIBRATION_PULSE_DURATION`, `_GAS_PUFF_DURATION_RANGE`, `_DEFAULT_GAS_PUFF_DURATION`, and
`_STATE_MONITOR_INTERVAL` module constants. The valve tracker carries the `_ValveTrackerIndex` members
`TOTAL_VOLUME = 0` and `CALIBRATION_STATE = 1`, where the calibration state reads 0 while calibrating and 1 once
calibrated.

The seventeen-member API is `start()`, `shutdown()`, `is_alive`, the non-clearing `exit_signal`, the clearing signals
`valve_open_signal`, `valve_close_signal`, `valve_reward_signal`, `valve_reference_signal`, `valve_calibrate_signal`,
`brake_lock_signal`, `brake_unlock_signal`, `gas_valve_open_signal`, `gas_valve_close_signal`, `gas_valve_pulse_signal`,
and the value properties `reward_volume`, `calibration_pulse_duration`, and `gas_valve_pulse_duration`.

`_MaintenanceUIWindow.__init__` titles the window `"Mesoscope-VR Maintenance Panel"` and fixes it at 550 by 750 pixels.
Its `_setup_ui()` groups Reward Valve Control, Valve Calibration with a `🔄 Reference (200 x 5 μL)` button and a
`📊 Calibrate` button, Brake Control, Gas Puff Valve Control, and `✖ Terminate Maintenance`. `_check_external_state`
detects reward completion from the cumulative dispensed volume, calibration and referencing completion from a
seen-active-then-idle transition on `CALIBRATION_STATE`, and puff completion from the cumulative puff count.

Every control on this panel is operated by the experimenter at the rig. You MUST NOT drive valve calibration,
referencing, brake actuation, or motor positioning on the operator's behalf.

---

## Behavior visualizer surface

`BehaviorVisualizer` (`mesoscope_vr/visualizer.py`) runs in the **main thread** of the runtime control process, with no
IPC, and is driven by direct calls from `runtime_cycle()`. The module forces the matplotlib backend to `QtAgg` with a
top-level `mpl.use("QtAgg")` call, the `__init__` sliding window spans 10 seconds at a 16 ms step, and `update()` routes
redraws through a `_BlitManager` that repaints only the data lines.

| Mode            | Value | Panels                                                                       |
|-----------------|-------|------------------------------------------------------------------------------|
| `LICK_TRAINING` | 0     | Lick sensor and valve plots                                                  |
| `RUN_TRAINING`  | 1     | Lick, valve, and running speed plots                                         |
| `EXPERIMENT`    | 2     | The above plus the trial panel, and the puff plot when aversive trials exist |

`open(mode=VisualizerMode.EXPERIMENT, *, has_reinforcing_trials=True, has_aversive_trials=True)` builds the
mode-dependent subplot layout and returns early when the window is already open, and an unrecognized mode is treated as
`EXPERIMENT`. The trial panel holds `_TRIAL_HISTORY_SIZE = 20` slots whose outcome codes are
`_TRIAL_OUTCOME_EMPTY = -1`, `_TRIAL_OUTCOME_FAILURE = 0`, `_TRIAL_OUTCOME_SUCCESS = 1`, and
`_TRIAL_OUTCOME_GUIDED = 2`. Axis bounds are `_SPEED_AXIS_YLIM = (-2.0, 42.0)` cm/s and
`_BINARY_AXIS_YLIM = (-0.05, 1.05)`.

| Method                                                                | Role                                                                         |
|-----------------------------------------------------------------------|------------------------------------------------------------------------------|
| `update()`                                                            | Rate-limited 16 ms re-render that rolls the buffers and blits the lines      |
| `update_run_training_thresholds(speed_threshold, duration_threshold)` | Moves the threshold lines, a no-op outside RUN_TRAINING                      |
| `add_lick_event()` / `add_valve_event()` / `add_puff_event()`         | Latch a tick for the next sample                                             |
| `update_running_speed(running_speed)`                                 | Latches the current speed                                                    |
| `add_trial_outcome(*, is_aversive, succeeded, was_guided)`            | Rolls the 20-slot type and outcome buffers and refreshes the blit background |
| `close()`                                                             | Closes the figure                                                            |
