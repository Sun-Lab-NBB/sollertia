---
name: mesoscope-vr-dataset-assembly
description: >-
  Documents the Mesoscope-VR session-assembly worker and admission policy that sollertia-forgery dispatches through its
  forging registries. Covers the session-type dispatcher, the experiment and training assembly paths, the behavior and
  runtime interpolation rules, sentinel masking, and the session-bounds clip. Use when interpreting an assembled session
  feather, debugging an assembly failure or an unexpected row count, or deciding which session types join a forged
  dataset.
user-invocable: false
---

# Mesoscope-VR dataset assembly

Concretizes the agnostic dataset-forging output stage for the Mesoscope-VR acquisition system. This skill owns the two
forging registry seams the system fills, `_FORGING_ASSEMBLY_REGISTRY` and `_FORGING_ADMISSION_REGISTRY`. The assembly
algorithm behind them is owned here too: the dispatcher `assemble_mesoscope_session` in `mesoscope_vr/forging.py`, the
experiment and training assembly paths it routes to, the behavior and runtime sub-datasets, the sentinel masking, and
the session-bounds clip.

The column-roster enum definitions themselves (`DatasetColumn`, `BehaviorDataFiles`) are declared in
`mesoscope_vr/metadata.py` and documented by `/mesoscope-vr-processing-schema`.

---

## Scope

**Covers:**
- The two forging registry seams Mesoscope-VR fills, and the assets donated into each
- `MESOSCOPE_ADMISSION_PIPELINES`, the per-session-type pipeline requirement that decides which sessions join a dataset
- `assemble_mesoscope_session`, the session-type dispatcher, and the two assembly paths it routes to
- The reference clock each path resolves, and the sub-datasets each path concatenates onto it
- Behavior-dataset discrete-versus-continuous interpolation, the running-speed sliding window, and reward
  classification (`no` / `tone` / `yes`)
- Behavior-dataset special cases: torque zeroed during run, distance clamping and holding across idle/rest, brake
  binary thresholding, and system-state code inversion from the hardware state
- The behavior-dataset output column order
- Runtime-dataset distance-indexed interpolation of `trial` / `trial_type` / `cue` and trigger-zone membership
- Sentinel masking outside the run state (`cue` 255, `trial` 65535, `trial_type` `"undefined"`) and the conditional
  guidance-state columns
- `clip_to_session_bounds`, the session-bounds clip that runs last on both paths
- Where a new assembled column is emitted, one site per `DatasetColumn` group
- Which session type makes `vr_configuration.yaml` a required artifact of a forged Mesoscope-VR dataset

**Does not cover:**
- The fluorescence sub-dataset and the experiment path's reference clock. Owned by
  `/mesoscope-vr-fluorescence-alignment`.
- The video sub-dataset, the camera clock resolver, and the pupil-tracking pass that feeds them. Owned by
  `/mesoscope-vr-video-tracking`.
- Runtime cue-to-trial decomposition that produces the trial, cue, and trigger-zone inputs. Owned by
  `/mesoscope-vr-trial-decomposition`.
- Module parser outputs consumed as behavior inputs. Owned by `/mesoscope-vr-module-parsing`.
- The `DatasetColumn`, `BehaviorDataFiles`, and `VideoDataFiles` roster enums, the per-session-type column-presence
  matrix, and `MESOSCOPE_COLUMN_DESCRIPTIONS`. Owned by `/mesoscope-vr-processing-schema`.
- The agnostic forged-dataset layout on disk and the per-stage tracker artifacts. Owned by `forging:processing-results`.
- The batch orchestration that dispatches this assembler, and the agnostic two-layer design. Owned by
  `forging:data-processing-design`.
- Composing a dataset from filtered sessions, and the admission gate that reads the pipeline mapping. Owned by
  `forging:dataset-definition`.
- The forged-dataset container itself: the `DatasetData` marker, the two-level layout, and the column-description
  companion. Owned by `assets:datasets`.

---

## Registry seams

Mesoscope-VR fills two of the thirteen `sollertia_forgery.registries` seams from `mesoscope_vr/forging.py`, both
keyed on `AcquisitionSystems.MESOSCOPE_VR`.

| Registry                      | Donated value                                                                                                    | Public resolver                                                          |
|-------------------------------|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| `_FORGING_ASSEMBLY_REGISTRY`  | `_ForgingAssemblyAsset(assembler=assemble_mesoscope_session, column_descriptions=MESOSCOPE_COLUMN_DESCRIPTIONS)` | `resolve_forging_assembly_worker`, `resolve_forging_column_descriptions` |
| `_FORGING_ADMISSION_REGISTRY` | `MESOSCOPE_ADMISSION_PIPELINES`                                                                                  | `resolve_forging_admission_pipelines`                                    |

`assemble_mesoscope_session` satisfies the public `ForgingAssembler` protocol, whose call signature is
`__call__(source_session_path, output_path, dataset_name) -> None`. It is a picklable module-level function because
the agnostic forging pipeline resolves it once and invokes it in a worker process, once per admitted session.

---

## Admission policy

`MESOSCOPE_ADMISSION_PIPELINES` maps each admissible session type to the frozen set of `ProcessingPipelines` members
that must have completed before a session of that type may join a forged dataset.

| `SessionTypes` member  | Value                    | Required pipelines                                              |
|------------------------|--------------------------|-----------------------------------------------------------------|
| `MESOSCOPE_EXPERIMENT` | `"mesoscope experiment"` | `CHECKSUM`, `RUNTIME`, `MICROCONTROLLER`, `VIDEO`, `TWO_PHOTON` |
| `RUN_TRAINING`         | `"run training"`         | `CHECKSUM`, `RUNTIME`, `MICROCONTROLLER`, `VIDEO`               |
| `LICK_TRAINING`        | `"lick training"`        | `CHECKSUM`, `RUNTIME`, `MICROCONTROLLER`, `VIDEO`               |
| `WINDOW_CHECKING`      | `"window checking"`      | absent from the mapping, so the session type joins no dataset   |

A training session records no imaging, so the two-photon pipeline is absent from its requirement. The mapping declares
pipelines rather than source counts because every pipeline resolves its own job universe from the acquisition
manifests, so a completed tracker already means every source the session recorded was processed. Admission therefore
checks which pipelines completed.

The agnostic gate that reads this mapping, and the two `ValueError` messages it raises for an inadmissible session type
and for outstanding pipelines, are owned by `forging:dataset-definition`.

---

## Session-type dispatch

`assemble_mesoscope_session(source_session_path, output_path, dataset_name)` loads the session with `SessionData.load`
and routes on `session.session_type`.

| Session type                                              | Routed to                                               | `dataset_name` forwarded                              |
|-----------------------------------------------------------|---------------------------------------------------------|-------------------------------------------------------|
| `SessionTypes.MESOSCOPE_EXPERIMENT`                       | `assemble_experiment_dataset` (`experiment_dataset.py`) | yes, it resolves the cindra multi-recording directory |
| `SessionTypes.RUN_TRAINING`, `SessionTypes.LICK_TRAINING` | `assemble_training_dataset` (`training_dataset.py`)     | no                                                    |
| any other session type                                    | nothing, a `ValueError` is raised                       | not applicable                                        |

The two training types are held in `_TRAINING_SESSION_TYPES`. The rejection message names the session directory, the
rejected type, and the supported set `"lick training, mesoscope experiment, run training"`, which is built by sorting
the three routed values.

---

## The experiment assembly path

`assemble_experiment_dataset(source_session_path, output_path, dataset_name)` writes one `data.feather` and runs these
steps in order:

1. **Pre-flight.** Three `is_dir()` checks, each raising `FileNotFoundError` that names the missing directory and the
   tracker it should contain: `processed_data.microcontroller_data_path` (`ProcessingTrackers.MICROCONTROLLER`),
   `processed_data.runtime_data_path` (`ProcessingTrackers.RUNTIME`), and `processed_data.cindra_data_path`
   (`ProcessingTrackers.TWO_PHOTON`). They run before any expensive work.
2. **Multi-recording resolution.** `multi_recording_dataset_name(animal_id, dataset_name)` returns
   `f"{animal_id}_{dataset_name}"`, and cindra's own `resolve_dataset_path` turns that name into the multiday
   directory under `session.processed_data_path`. Qualifying the name with the animal identifier keeps each animal's
   multi-recording output separate when one dataset spans several animals.
3. **Output and configuration.** `ensure_directory_exists(path=output_path, is_file=True)`, then one
   `MesoscopeExperimentConfiguration.from_yaml` load of `raw_data/experiment_configuration.yaml`
   (`RawDataFiles.EXPERIMENT_CONFIGURATION`), so the runtime assembly resolves its mappings without re-reading it.
4. **Fluorescence first.** `assemble_cindra_dataset` runs alone, and its `time_us` column becomes `reference_time`.
5. **Three sub-datasets in parallel.** The `behavior`, `runtime`, and `video` assemblies are submitted as
   `functools.partial` tasks to a `ThreadPoolExecutor(max_workers=len(tasks))` and collected with `as_completed`. The
   behavior task passes `drop_time_columns=True`, because the fluorescence half already supplies the time axis.
6. **Horizontal stack.** `reduce(pl.DataFrame.hstack, ...)` runs over `[fluorescence, behavior, runtime]`, with the
   video frame appended only when `results["video"].width > 0`. Stacking requires every sub-dataset to carry the
   reference clock's height, so one that drifts off that clock raises rather than being padded.
7. **Mask, clip, write.** `mask_non_run_experiment_data`, then `clip_to_session_bounds`, then
   `result.write_ipc(file=output_path)`.

---

## The training assembly path

`assemble_training_dataset(source_session_path, output_path)` takes no `dataset_name`, because a training session has
no cindra output to locate:

1. **Pre-flight.** Two `is_dir()` checks with the same two messages the experiment path uses, on
   `processed_data.microcontroller_data_path` and `processed_data.runtime_data_path`. There is no cindra check, since
   only the behavior sources are required.
2. **Reference clock.** `resolve_slowest_camera_clock(video_data_path=...)` returns the slowest camera's timestamps
   verbatim. It is resolved **before** `ensure_directory_exists`, so a session with no usable camera clock leaves no
   empty output directory behind.
3. **Two sub-datasets, sequentially.** `assemble_behavior_dataset(..., drop_time_columns=False)`, so the behavior half
   supplies the feather's `time_us` and `elapsed_minutes` columns, then `assemble_video_dataset` on the same clock.
4. **Stack, clip, write.** `hstack` over `[behavior]` with the video frame appended when `video_data.width`, then
   `clip_to_session_bounds`, then `result.write_ipc(file=output_path)`.

There is no runtime sub-dataset on this path and no `mask_non_run_experiment_data` call, because a training session
produces none of the runtime columns that masking rewrites.

---

## Reference clock and interpolation

| Session type                | Reference clock                                                                   | Produced by                                  |
|-----------------------------|-----------------------------------------------------------------------------------|----------------------------------------------|
| mesoscope experiment        | the fluorescence sub-dataset's `time_us`, one sample per acquired mesoscope frame | `mesoscope_vr/two_photon_dataset.py`         |
| run training, lick training | the slowest camera's frame timestamps, verbatim                                   | `video_dataset.resolve_slowest_camera_clock` |

Both clocks are `uint64` NumPy arrays of microsecond timestamps, and every sub-dataset a path assembles is aligned onto
the clock that path resolved. The slowest camera is chosen for training sessions because every other source
interpolates onto its coarser grid without inventing samples between its frames.

All alignment is performed by `interpolate_data` from `ataraxis-data-structures`, which maps source values sampled at
`source_coordinates` onto `target_coordinates`. The `is_discrete` flag selects nearest-previous (step) interpolation
for `True` and linear interpolation for `False`.

See `/mesoscope-vr-fluorescence-alignment` for the fluorescence clock and `/mesoscope-vr-video-tracking` for the
camera clock and the video sub-dataset.

---

## Behavior-dataset interpolation rules and special cases

`assemble_behavior_dataset(microcontroller_data_path, runtime_data_path, raw_data_path, reference_time, *,
drop_time_columns=False)` reads its sources from two processed-data directories. The module-parsed feathers (encoder,
lick, valve, brake, torque, screen) come from the session's `processed_data/microcontroller_data` directory, and the
runtime system-state feather comes from `processed_data/runtime_data`, both addressed by their `BehaviorDataFiles`
names. The session's `MesoscopeHardwareState` is read from `raw_data/hardware_state.yaml`. Each source is then
interpolated onto `reference_time`. `drop_time_columns` is keyword-only and defaults to `False`. The full
feather-to-directory mapping is owned by `/mesoscope-vr-processing-schema`.

### Always-present sources

| Source feather                      | Aligned column        | `is_discrete` | Notes                                                                                                    |
|-------------------------------------|-----------------------|---------------|----------------------------------------------------------------------------------------------------------|
| `system_state_data.feather`         | `system_state`        | `True`        | Code series, inverted to names and cast to a Polars Enum (below)                                         |
| `lick_data.feather`                 | `lick`                | `True`        | Thresholded lick state                                                                                   |
| `valve_data.feather` (water volume) | `water_uL`            | `True`        | Cast to `Float32`, discrete because the power-law dispensing curve makes linear interpolation inaccurate |
| `valve_data.feather` (tone state)   | (`reward` derivation) | `True`        | Interpolated into a temporary tone column used only to classify rewards                                  |

### Conditional sources

Each of these is included only when its feather exists in the microcontroller-data directory:

| Source feather         | Aligned column | `is_discrete` | Present for                                    |
|------------------------|----------------|---------------|------------------------------------------------|
| `encoder_data.feather` | `distance_cm`  | `False`       | All session types except lick training         |
| `encoder_data.feather` | `speed_cm_s`   | `False`       | Derived from encoder distance (sliding window) |
| `screen_data.feather`  | `screens`      | `True`        | Mesoscope experiments only                     |
| `brake_data.feather`   | `brake`        | `True`        | Mesoscope experiments only, then thresholded   |
| `torque_data.feather`  | `torque_N_cm`  | `False`       | All session types except run training          |

### Running-speed sliding window

`speed_cm_s` is computed by the numba-accelerated `_calculate_running_speed`, which uses a 100 ms sliding window
(`_RUNNING_SPEED_WINDOW_US = 100_000` microseconds) over the **original** encoder samples (not the downsampled
reference grid). For each sample it advances a monotonic window-start index to the first sample within the window,
divides the distance delta by the time delta in seconds, and clamps the result to a non-negative `Float32`. It writes
zero whenever the window holds no distinct earlier sample. The resulting speed series is then interpolated
(`is_discrete=False`) onto `reference_time`.

### Reward classification

The water valve's tone state is interpolated into a temporary column, flagged active wherever it exceeds zero, and
segmented into reward events by detecting transitions in the active flag and taking a cumulative sum. The dispensed
`water_uL` is a cumulative running total, so its per-sample increment is summed within each reward event, and each
sample is classified into a Polars `Enum(["no", "tone", "yes"])`:

- `no`, when no tone is active at the sample
- `yes`, when a tone is active and the event delivered water (summed `water_uL` greater than zero)
- `tone`, when a tone is active but the event delivered no water

The temporary tone, active-flag, event-id, water-delta, and per-event-water columns are dropped after classification.

### System-state code inversion

The `MesoscopeHardwareState` supplies a `system_state_codes` mapping (name to code). The assembly inverts it (code to
name) and casts `system_state` to a Polars Enum built from the mapping's keys, yielding the five descriptive state
names (`idle`, `rest`, `run`, `lick training`, `run training`). The downstream special cases and the masking test only
`idle`, `rest`, and `run`. If the hardware state is missing `system_state_codes`, assembly raises a `ValueError`.

### Torque, distance, and brake special cases

- **Torque zeroed during run:** if `torque_N_cm` is present, its value is set to `0.0` wherever `system_state` is
  `run` (the torque sensor is disabled in the run state).
- **Distance clamping and holding:** if `distance_cm` is present, the cumulative traveled distance is held constant
  outside the run state. The initial idle distance (before the system has ever left idle) is forced to `0.0`,
  distance updates only while `system_state` is `run`, intermediate non-run samples forward-fill the last run value,
  and any remaining nulls fill to `0.0`. The running speed is simultaneously set to `0.0` outside the run state.
- **Brake binary threshold:** the brake torque is interpolated (`is_discrete=True`) and converted to a binary `uint8`
  `brake` column, computed as `brake = brake_torque > minimum_brake_strength` against the hardware state's
  `minimum_brake_strength`. If the hardware state is missing that field, assembly raises a `ValueError`.

---

## Behavior-dataset column order

After cleanup, the behavior DataFrame is reordered to this canonical sequence. Only columns that actually exist for the
session are selected, so lick-training and run-training sessions omit the columns whose source feathers are absent:

```text
time_us
elapsed_minutes
brake
screens
torque_N_cm
distance_cm
speed_cm_s
lick
water_uL
reward
system_state
```

The experiment path passes `drop_time_columns=True`, which removes `time_us` and `elapsed_minutes` (the
`_TIME_COLUMNS` frozen set) from the selection before the frame is returned. The training path leaves the `False`
default in place, so both time columns survive into the assembled feather.

---

## Runtime-dataset distance-indexed interpolation and trigger zones

`assemble_runtime_dataset(microcontroller_data_path, runtime_data_path, experiment_configuration, reference_time)`
runs on the experiment path only. It reads the encoder feather from the microcontroller-data directory and the VR cue,
trigger zone, trial, and runtime state feathers from the runtime-data directory, and it receives the
`MesoscopeExperimentConfiguration` the experiment path already loaded. Unlike the behavior assembly, several runtime
columns are indexed by **traveled distance** rather than by time:

1. The encoder's traveled distance is interpolated onto `reference_time` (`is_discrete=False`) to produce a
   `reference_distance` vector, the cumulative distance at each reference sample.
2. `trial`, `trial_type`, and `cue` are then interpolated against `reference_distance` (their source coordinates are
   the per-trial / per-cue traveled-distance values, not timestamps), all with `is_discrete=True`:

| Aligned column | Source feather        | Source coordinates               | Source values                                                 |
|----------------|-----------------------|----------------------------------|---------------------------------------------------------------|
| `trial`        | `trial_data.feather`  | per-trial `traveled_distance_cm` | sequential 1-based trial numbers (`uint32`), cast to `UInt16` |
| `trial_type`   | `trial_data.feather`  | per-trial `traveled_distance_cm` | `trial_type_index`, mapped to an Enum (below)                 |
| `cue`          | `vr_cue_data.feather` | per-cue `traveled_distance_cm`   | `vr_cue`                                                      |

3. `runtime_state` is interpolated against `reference_time` (`is_discrete=True`) from `runtime_state_data.feather`.

The trial sources are produced upstream by runtime cue-to-trial decomposition. See `/mesoscope-vr-trial-decomposition`.

### Trigger-zone membership

`in_trigger_zone` is computed by the numba-accelerated `_check_trigger_zones`, which marks each `reference_distance`
point `1` (inside) or `0` (outside) by walking the per-trial trigger-zone start/end boundaries from
`vr_trigger_zone_data.feather`. It relies on both the distance series and the trigger-zone boundaries being sorted
and monotonically increasing. It walks a zone index forward as distance increases (and backward to absorb slight
non-monotonicity), and returns an all-zero array when no trigger zones are defined.

### Enum mappings

The experiment configuration supplies the categorical mappings:

- `trial_type` is mapped from integer index to name via `enumerate(experiment_configuration.trial_structures.keys())`
  and cast to a Polars Enum whose categories are the trial-type names plus the literal `"undefined"` (reserved for
  masking).
- `runtime_state` is mapped from each experiment state's `experiment_state_code` to its state name, with code `0`
  (`_RUNTIME_STATE_IDLE`) added as the implicit `idle` state the configuration never lists, and cast to a Polars Enum
  of those names.

### Conditional guidance-state columns

The two guidance columns are added only when their feather files were produced for the session:

| Aligned column       | Source feather                            | `is_discrete` |
|----------------------|-------------------------------------------|---------------|
| `reinforcing_guided` | `reinforcing_guidance_state_data.feather` | `True`        |
| `aversive_guided`    | `aversive_guidance_state_data.feather`    | `True`        |

Each guidance source value is cast to `uint8` before interpolation.

---

## Sentinel masking

`mask_non_run_experiment_data` is public, is exported from `runtime_dataset.py`, and is called from
`assemble_experiment_dataset` over the horizontally concatenated frame. It has to run there because only the
concatenated frame carries `system_state`, which the behavior half contributes. Where `system_state` is `idle` or
`rest`, the runtime columns are not meaningful and are replaced with sentinels that sit outside every legitimate value
range:

| Column       | Masked value  | Sentinel rationale                                                                  |
|--------------|---------------|-------------------------------------------------------------------------------------|
| `cue`        | `255`         | `_CUE_UNDEFINED`, maximum of `UInt8`, outside the valid cue-code range              |
| `trial`      | `65535`       | `_TRIAL_UNDEFINED`, maximum of `UInt16`, outside the trial-ID range for any session |
| `trial_type` | `"undefined"` | the dedicated Enum member added to the trial-type categories                        |

Only `idle` and `rest` are masked. `run` samples keep their interpolated values. Membership is tested by imploding the
two-state series into a single list value, which matches each row against the collection as a whole rather than
pairing the column with it element-wise. The masking is then applied column-by-column with
`pl.when(is_non_run).then(<sentinel>).otherwise(<column>)`, preserving each column's dtype.

Which columns an assembled feather carries at all depends on its session type. The complete per-session-type
column-presence matrix is owned by `/mesoscope-vr-processing-schema`.

---

## Session-bounds clipping

`clip_to_session_bounds(assembled_data, runtime_data_path)` is public, runs last on **both** paths, and can change the
row count of a forged feather. It reads `system_state_data.feather` and `runtime_state_data.feather` from the
runtime-data directory:

- **Head.** The session begins at the first `system_state_data.feather` row whose `system_state` differs from
  `_SYSTEM_STATE_IDLE` (`0`). Rows whose `time_us` falls before that timestamp are dropped.
- **Tail.** Rows whose `time_us` falls after the last `runtime_state_data.feather` `time_us` are dropped.

The acquisition assets start and stop in sequence around the session, so the head holds the setup span and the tail
holds the teardown span. Later idle spans stay in place, because a paused session also returns to idle. A session that
never leaves idle keeps its head, and a session with no runtime-state entry keeps its tail.

`elapsed_minutes` is measured from the first sample of the unclipped reference clock, which already runs during
setup, so the first row of an assembled feather carries an `elapsed_minutes` value above zero.

---

## Emitting a new assembled column

The `DatasetColumn` group that a column joins names the one sub-assembler that emits it. `time_us` and `elapsed_minutes`
are the exception, coming from the fluorescence assembly for experiment sessions and from the behavior assembly for
training sessions. The member declaration, its `_COLUMN_DESCRIPTIONS` entry, and the rebuild a new column forces on
datasets already defined belong to `/mesoscope-vr-processing-schema`, and this table serves that skill's third touch.

| `DatasetColumn` group       | Sub-assembler                                      | Where the value is emitted                                                              |
|-----------------------------|----------------------------------------------------|-----------------------------------------------------------------------------------------|
| Behavior alignment          | `assemble_behavior_dataset`, `behavior_dataset.py` | The aligned column, plus an entry in the `final_columns` list behind the order above    |
| Runtime and experiment      | `assemble_runtime_dataset`, `runtime_dataset.py`   | A key in the `aligned_data` dict, handed to `pl.DataFrame` whole with no order list     |
| cindra fluorescence         | `assemble_cindra_dataset`, `two_photon_dataset.py` | A `with_columns` call, or a `fluorescence_sources` row for a cindra trace array         |
| Video motion energy         | `assemble_video_dataset`, `video_dataset.py`       | A per-camera key in that module's own `aligned_data` dict, built from `_CAMERA_SOURCES` |
| Pupil tracking, face camera | `assemble_video_dataset`, `video_dataset.py`       | Nothing here for a continuous metric, copied from the pupil feather's own schema        |

The per-emission-site rules each sub-assembler adds, covering the `final_columns` select, the `PupilColumn` split
between continuous metrics and state flags, and the `mask_non_run_experiment_data` branch a new runtime column needs,
live in [`references/column-emission.md`](references/column-emission.md).

---

## Error surface

| Raise                   | Origin                                         | Condition                                                                                                  |
|-------------------------|------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `ValueError`            | `forging.py`                                   | The session type is neither the experiment type nor a member of `_TRAINING_SESSION_TYPES`                  |
| `FileNotFoundError`     | `experiment_dataset.py`, `training_dataset.py` | The processed microcontroller-data or runtime-data directory does not exist                                |
| `FileNotFoundError`     | `experiment_dataset.py`                        | The single-recording cindra output directory does not exist                                                |
| `FileNotFoundError`     | `video_dataset.py`                             | No camera clock qualifies as the training reference clock                                                  |
| `FileNotFoundError`     | `behavior_dataset.py`                          | The hardware state YAML, or the valve, lick, or system-state feather, is missing                           |
| `ValueError`            | `behavior_dataset.py`                          | The hardware state is missing `system_state_codes`, or `minimum_brake_strength` when brake data is present |
| `InvalidOperationError` | `behavior_dataset.py`                          | The system-state feather carries a code absent from the `system_state_codes` mapping                       |
| `FileNotFoundError`     | `runtime_dataset.py`                           | Any of the five required encoder, VR cue, trigger zone, trial, or runtime state feathers is missing        |
| `InvalidOperationError` | `runtime_dataset.py`                           | A recorded trial type index or runtime state code has no entry in the experiment configuration's mappings  |

`InvalidOperationError` is the Polars exception `replace_strict` raises on an unmapped value, so it surfaces as an
unmapped code rather than as a missing file. Every other raise here is routed through
`ataraxis_base_utilities.console.error`.

---

## VR configuration as a forged-dataset artifact

A forged dataset re-exports each session's `vr_configuration.yaml` next to the assembled `data.feather`, and dataset
inspection treats that copy as required only when the dataset's own `session_type` belongs to
`SESSION_TYPES_USING_VR_TASK`. That frozen set holds exactly one member today, `SessionTypes.MESOSCOPE_EXPERIMENT`, so
the copy is a required dataset artifact only for datasets whose `session_type` is `"mesoscope experiment"`. In a
dataset of lick-training, run-training, or window-checking sessions, a missing `vr_configuration.yaml` describes the
source sessions rather than a defective dataset.

The registry-dispatched mechanism behind that rule, the `DatasetData` marker it is read from, and the inspection
report that applies it belong to `assets:datasets`.

---

## Related skills

| Skill                                  | Relationship                                                                                 |
|----------------------------------------|----------------------------------------------------------------------------------------------|
| `forging:dataset-forging`              | Runs the forging pipeline that dispatches this assembler once per admitted session           |
| `forging:data-processing-design`       | Owns the agnostic two-layer design and the registry seams this skill fills                   |
| `forging:processing-results`           | Owns the agnostic forged-dataset layout and the tracker artifacts around this stage          |
| `forging:dataset-definition`           | Composes a dataset from filtered sessions and owns the gate that reads the admission mapping |
| `assets:datasets`                      | Owns the `DatasetData` marker, the dataset layout, and the seven slsa dataset tools          |
| `/mesoscope-vr-fluorescence-alignment` | Produces the fluorescence sub-dataset and the experiment path's reference clock              |
| `/mesoscope-vr-video-tracking`         | Owns the video sub-dataset, the camera clock resolver, and the pupil columns                 |
| `/mesoscope-vr-trial-decomposition`    | Produces the `trial` / `cue` / `vr_trigger_zone` feathers consumed by the runtime assembly   |
| `/mesoscope-vr-module-parsing`         | Produces the per-module behavior feathers consumed by the behavior assembly                  |
| `/mesoscope-vr-processing-schema`      | Owns the roster enums, the presence matrix, and the recipe that declares a new column        |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Assembly claims:
- [ ] The entry point is named as assemble_mesoscope_session, and both routed branches match forging.py
- [ ] The experiment path is stated as four sub-datasets, with behavior, runtime, and video in one thread pool and
      the fluorescence sub-dataset assembled first and alone
- [ ] The training path is stated as behavior plus video on the slowest camera clock, with no runtime sub-dataset
      and no masking
- [ ] drop_time_columns is stated as True on the experiment path and False on the training path
- [ ] MESOSCOPE_ADMISSION_PIPELINES is quoted with its three mapped session types, and window checking is stated
      as absent from the mapping
- [ ] Every interpolation claim (is_discrete True/False, source vs reference, distance vs time) matches
      behavior_dataset.py / runtime_dataset.py
- [ ] Running-speed window stated as 100 ms (_RUNNING_SPEED_WINDOW_US = 100_000 us) over original encoder samples
- [ ] Reward classes named exactly no / tone / yes with the correct water-volume condition
- [ ] Masking attributed to the public mask_non_run_experiment_data, applied only for idle and rest over the
      concatenated frame, with sentinels cue 255, trial 65535, and trial_type "undefined"
- [ ] clip_to_session_bounds stated as running last on both paths, head at the first non-idle system state and tail
      at the last runtime-state entry
- [ ] Behavior column order quoted exactly
- [ ] A newly emitted column was placed in the one sub-assembler its DatasetColumn group names, and a behavior
      column also joined final_columns, whose closing select drops what the list omits
- [ ] references/column-emission.md was applied to the new column, with a boolean pupil state flag also joining
      _PUPIL_FLAG_COLUMNS in video_dataset.py
- [ ] Conditional columns (encoder/screen/brake/torque, reinforcing_guided, aversive_guided) gated on file existence
- [ ] Behavior sources homed in processed_data/microcontroller_data and processed_data/runtime_data, hardware state
      in raw_data/hardware_state.yaml, with no processed_data/behavior_data claim
- [ ] vr_configuration.yaml stated as a required dataset artifact only for the "mesoscope experiment" session type,
      the sole SESSION_TYPES_USING_VR_TASK member
- [ ] No invented symbols, filenames, tolerances, or event codes, every one derived from the cited source files
- [ ] Did not redefine the DatasetColumn / BehaviorDataFiles enums or the column-presence matrix, handed off to
      /mesoscope-vr-processing-schema
- [ ] Did not restate the video sub-dataset or the camera clock internals, handed off to
      /mesoscope-vr-video-tracking
- [ ] Did not restate the DatasetData marker, the dataset layout, or the dataset tools, handed off to assets:datasets
```
