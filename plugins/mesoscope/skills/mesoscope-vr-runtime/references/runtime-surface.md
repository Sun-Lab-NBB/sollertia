# Mesoscope-VR runtime surface

Enumerates the per-mode runtime logic functions and the `sle mesoscope` CLI commands that invoke them. See
[`../SKILL.md`](../SKILL.md) for the state machine, the orchestrator, and the workflow for adding a new runtime mode.

---

## Per-mode runtime logic functions

Each runtime mode has a top-level function in `sollertia_experiment/mesoscope_vr/data_acquisition.py`
that:

1. Validates inputs and prepares the output directories (builds the `SessionData` hierarchy).
2. Builds the session-specific descriptor from default values, the previous same-type session's
   parameters (when available), and per-flag overrides; experiment sessions additionally load the
   `MesoscopeExperimentConfiguration` from its YAML.
3. Builds the hardware assets the mode needs. `lick_training_logic`, `run_training_logic`, and
   `experiment_logic` construct `MesoscopeVRSystem`, which owns and starts its own `DataLogger`.
   `window_checking_logic` and `maintenance_logic` construct their own `DataLogger` and hardware
   assets directly, with no orchestrator.
4. Marks the session as initialized. Every session-running function, once its hardware assets are built and the session
   is ready to acquire, calls `session_data.mark_runtime_initialized()`, which removes the `nk.bin` marker from the
   session's `raw_data` directory. This is a different signal from the descriptor's `incomplete` flag, which the
   runtime sets at session end. See `assets:session-data` for the marker's semantics. `maintenance_logic` runs no
   session and never performs this step.
5. Runs the session's control loop. The three `MesoscopeVRSystem` modes drive state transitions and
   call `runtime_cycle()` each iteration.
6. Tears down on completion or error.

Current functions:

| Function                | Purpose                                                                          |
|-------------------------|----------------------------------------------------------------------------------|
| `window_checking_logic` | Cranial window maintenance session (no behavior)                                 |
| `lick_training_logic`   | Lickport-only training (animal learns to operate the lickport for water rewards) |
| `run_training_logic`    | Wheel-only training (animal learns to run for water rewards)                     |
| `experiment_logic`      | Full Mesoscope-VR experiment session with VR trial structure                     |
| `maintenance_logic`     | Hardware maintenance (valve calibration, motor positioning, brake testing)       |

Every session-running function takes the experimenter, project, and animal identifiers and builds the
`SessionData` and descriptor internally. `lick_training_logic`, `run_training_logic`, and
`experiment_logic` additionally take the animal weight and per-flag parameter overrides, and
`experiment_logic` also takes `experiment_name`. `window_checking_logic` takes exactly the three
identifiers. `maintenance_logic()` takes no session.

### Session descriptor consumption

The descriptor dataclasses are owned by `sollertia-shared-assets`. Their field surface is documented in
`/mesoscope-vr-session-schema`, and the generic descriptor read and write tools in `assets:session-descriptors`. The
per-mode logic function instantiates the descriptor with default values, layers in the previous same-type session's
parameters (when available) and per-flag overrides, and parameterizes the state machine from it.

The runtime both consumes and completes the descriptor:

- At construction the orchestrator caches the partially configured descriptor to the session's
  `raw_data` directory, so the session can be preprocessed even if the runtime terminates
  unexpectedly.
- During runtime it records runtime-discovered values into the descriptor in place: the dispensed and
  pause-dispensed water volumes, the pre-filled experimenter-delivered water volume, the `incomplete`
  flag, and (for run-training sessions) the final speed and duration thresholds as the operator
  adjusts them through the control UI.
- At session end `_generate_session_descriptor()` calls `finalize_session_descriptor(...)`, which
  collects the supervising experimenter's notes through a blocking `questionary` terminal prompt and,
  for window-checking sessions, a 0-3 cranial-window quality rating; stores both on the descriptor;
  writes the completed descriptor to the session's `raw_data` directory; and copies it to the animal's
  persistent directory, where it is used to restore the training parameters across sessions of the
  same type.

---

## CLI command surface

The user-facing entry points live in `sollertia_experiment/interfaces/mesoscope_vr.py`, registered
under the `sle mesoscope` command group (itself registered on the top-level `sle` group in
`interfaces/entry_points.py`):

| Command                              | Calls                                  | Notes                                                    |
|--------------------------------------|----------------------------------------|----------------------------------------------------------|
| `sle mesoscope configure system`     | `create_system_configuration_file`     | Writes the system configuration YAML                     |
| `sle mesoscope configure experiment` | `create_experiment_configuration_file` | Creates an experiment configuration from a task template |
| `sle mesoscope maintain`             | `maintenance_logic`                    | Hardware maintenance GUI (no session)                    |
| `sle mesoscope run window-checking`  | `window_checking_logic`                | Cranial-window maintenance mode                          |
| `sle mesoscope run lick-training`    | `lick_training_logic`                  | Defaults match the lick-training descriptor              |
| `sle mesoscope run run-training`     | `run_training_logic`                   | Absolute speed/duration threshold targets via flags      |
| `sle mesoscope run experiment`       | `experiment_logic`                     | Takes `--experiment` for the experiment configuration    |
| `sle mesoscope preprocess`           | (session data lifecycle)               | See `experiment:data-management`                         |
| `sle mesoscope delete`               | (session data lifecycle)               | See `experiment:data-management`                         |
| `sle mesoscope migrate`              | (session data lifecycle)               | See `experiment:data-management`                         |

`sle mesoscope configure` is a command group with two targets. `configure system` takes no options
and writes the system configuration YAML under the working directory. `configure experiment` takes
the required `-p/--project`, `-e/--experiment`, and `-t/--template` options plus `-sc/--state-count`
(default 1), `--reward-size` (default 5.0), `--reward-tone-duration` (default 300), and
`--puff-duration` (default 100), and writes the experiment configuration under the configured data
root. See `assets:experiment-configuration` for the configuration contract itself.

`sle mesoscope run` is a command group. The `--user`, `--project`, `--animal`, and `--animal-weight`
options are supplied on `run` and shared by every session subcommand. Each session subcommand reads
those shared values out of the Click context, adds its own per-flag overrides, and calls the matching
per-mode logic function, which builds the `SessionData` and the descriptor. The CLI is the only
public surface for starting a session.
