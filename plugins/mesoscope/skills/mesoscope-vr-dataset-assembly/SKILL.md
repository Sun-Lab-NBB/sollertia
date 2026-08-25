---
name: mesoscope-vr-dataset-assembly
description: >-
  Documents Mesoscope-VR's concrete instance of the session-assembly stage: how the behavior and runtime streams
  are interpolated onto the fluorescence frame reference vector to build the assembled session feather. Covers the
  behavior-dataset interpolation rules and special cases, the runtime-dataset distance-indexed interpolation with
  trigger zones, and sentinel masking outside the run state. Use when interpreting an assembled session feather
  column, debugging interpolation or sentinel masking, or modifying the assembly schema.
user-invocable: false
---

# Mesoscope-VR dataset assembly

Concretizes the agnostic dataset-forging output stage for the Mesoscope-VR acquisition system: aligns the behavior
and runtime streams to the fluorescence frame reference vector and assembles them into the per-session data feather.

This skill is the single owner of the assembly **algorithm** — the interpolation rules, the per-column special
cases, and the sentinel masking — for both `assemble_behavior_dataset` (in `mesoscope_vr/behavior_dataset.py`) and
`assemble_runtime_dataset` (in `mesoscope_vr/runtime_dataset.py`). The agnostic batch orchestration that drives this
stage is owned by the forging plugin (see `forging:data-processing-design`); the final forged-feather column roster
at the output/reference level is owned by `forging:dataset-forging-results`, which points here for the algorithm.

The column-roster enum definitions themselves (`DatasetColumn`, `BehaviorDataFiles`) are defined in `metadata.py`
and documented by `mesoscope:mesoscope-vr-processing-schema`; this skill instantiates the `DatasetColumn` roster as
the output schema of the assembly stage but does not own the enum definition.

---

## Scope

**Covers:**
- The fluorescence-frame reference time vector that both assembly functions align onto (`reference_time`)
- Behavior-dataset discrete-versus-continuous interpolation, the running-speed sliding window, and reward
  classification (`no` / `tone` / `yes`)
- Behavior-dataset special cases: torque zeroed during run, distance clamping and holding across idle/rest, brake
  binary thresholding, and system-state code inversion from the hardware state
- The behavior-dataset output column order
- Runtime-dataset distance-indexed interpolation of `trial` / `trial_type` / `cue` and trigger-zone membership
- Sentinel masking outside the run state (`cue` 255, `trial` 65535, `trial_type` `"undefined"`) and the conditional
  guidance-state columns
- Which session type makes `vr_configuration.yaml` a required artifact of a forged Mesoscope-VR dataset (the current
  `SESSION_TYPES_USING_VR_TASK` membership)

**Does not cover:**
- Fluorescence frame alignment that produces the reference vector (see `mesoscope:mesoscope-vr-fluorescence-alignment`)
- Runtime cue-to-trial decomposition that produces the trial / cue / trigger-zone inputs (see
  `mesoscope:mesoscope-vr-trial-decomposition`)
- Module parser outputs consumed as behavior inputs (see `mesoscope:mesoscope-vr-module-parsing`)
- The `DatasetColumn` and `BehaviorDataFiles` roster enum definitions (see `mesoscope:mesoscope-vr-processing-schema`)
- The forged-feather output schema reference, fluorescence array shapes, and dataset hierarchy (see
  `forging:dataset-forging-results`)
- The dataset-forging batch orchestration and the agnostic processing design (see `forging:data-processing-design`)
- The forged-dataset container itself: the `DatasetData` marker, the two-level layout, and the column-description
  companion (see `assets:datasets`)
- Composing a dataset from filtered sessions and tracking its forging job state (see `forging:dataset-definition`)

---

## Assembly stage contract

The forging pipeline concatenates three independently assembled DataFrames — fluorescence, behavior, and runtime —
horizontally into one per-session `data.feather`. This skill owns the behavior and runtime halves. The contract:

1. The fluorescence assembly runs **first** and its `time_us` column becomes the `reference_time` vector for the
   other two assemblies.
2. `assemble_behavior_dataset` and `assemble_runtime_dataset` run in parallel, each aligning its sources onto
   `reference_time`.
3. The behavior assembly is called with `drop_time_columns=True` so it contributes its data columns but **not** the
   shared `time_us` / `elapsed_minutes` columns (those come from the fluorescence half), avoiding duplicate columns
   in the horizontal concatenation.
4. After the horizontal concatenation, `_mask_non_run_experiment_data` runs over the **combined** frame. It reads
   the `system_state` column (contributed by the behavior half) to mask the runtime half's `cue`, `trial`, and
   `trial_type`. The runtime DataFrame alone has no `system_state` column, so masking can only be applied after the
   concatenation, not inside `assemble_runtime_dataset`.

All interpolation is performed by `interpolate_data` from `ataraxis-data-structures`, which maps source values
sampled at `source_coordinates` onto `target_coordinates`. The `is_discrete` flag selects nearest-previous (step)
interpolation for `True` and linear interpolation for `False`.

---

## The fluorescence-frame reference vector

Both assembly functions take a `reference_time` argument typed as a `uint64` NumPy array of microsecond timestamps.
It is the fluorescence dataset's `time_us` column — one entry per acquired mesoscope frame — so every assembled
behavior and runtime column is sampled at the fluorescence frame rate. The behavior assembly derives
`elapsed_minutes` from this same `time_us` vector (minutes since the first sample, rounded to two decimals as
`Float32`), but it is dropped before concatenation when `drop_time_columns=True`; the fluorescence half supplies the
canonical `time_us` and `elapsed_minutes` columns in the final feather.

Producing the reference vector is out of scope here — see `mesoscope:mesoscope-vr-fluorescence-alignment`.

---

## Behavior-dataset interpolation rules and special cases

`assemble_behavior_dataset` reads its sources from two processed-data directories, matching its
`assemble_behavior_dataset(microcontroller_data_path, runtime_data_path, ...)` signature: the module-parsed feathers
(encoder, lick, valve, brake, torque, screen) come from the session's `processed_data/microcontroller_data`
directory, and the runtime system-state feather comes from `processed_data/runtime_data`, both addressed by their
`BehaviorDataFiles` names. The session's `MesoscopeHardwareState` is read from `raw_data/hardware_state.yaml`. Each
source is then interpolated onto `reference_time`. The full feather-to-directory mapping is owned by
`mesoscope:mesoscope-vr-processing-schema`.

### Always-present sources

| Source feather                       | Aligned column          | `is_discrete` | Notes                                                              |
|--------------------------------------|-------------------------|---------------|--------------------------------------------------------------------|
| `system_state_data.feather`          | `system_state`          | `True`        | Code series; inverted to names and cast to a Polars Enum (below)   |
| `lick_data.feather`                  | `lick`                  | `True`        | Thresholded lick state                                             |
| `valve_data.feather` (water volume)  | `water_uL`              | `True`        | Cast to `Float32`; discrete because the power-law dispensing curve makes linear interpolation inaccurate |
| `valve_data.feather` (tone state)    | (`reward` derivation)   | `True`        | Interpolated into a temporary tone column used only to classify rewards |

### Conditional sources

Each of these is included only when its feather exists in the microcontroller-data directory:

| Source feather              | Aligned column | `is_discrete` | Present for                                  |
|-----------------------------|----------------|---------------|----------------------------------------------|
| `encoder_data.feather`      | `distance_cm`  | `False`       | All session types except lick training       |
| `encoder_data.feather`      | `speed_cm_s`   | `False`       | Derived from encoder distance (sliding window) |
| `screen_data.feather`       | `screens`      | `True`        | Mesoscope experiments only                   |
| `brake_data.feather`        | `brake`        | (thresholded) | Mesoscope experiments only                   |
| `torque_data.feather`       | `torque_N_cm`  | `False`       | All session types except run training        |

### Running-speed sliding window

`speed_cm_s` is computed by the numba-accelerated `_calculate_running_speed`, which uses a 100 ms sliding window
(`_RUNNING_SPEED_WINDOW_US = 100_000` microseconds) over the **original** encoder samples (not the downsampled
reference grid). For each sample it advances a monotonic window-start index to the first sample within the window,
divides the distance delta by the time delta in seconds, clamps the result to a non-negative `Float32`, and writes
zero whenever the window holds no distinct earlier sample. The resulting speed series is then interpolated
(`is_discrete=False`) onto `reference_time`.

### Reward classification

The water valve's tone state is interpolated into a temporary column, flagged active wherever it exceeds zero, and
segmented into reward events by detecting transitions in the active flag and taking a cumulative sum. The dispensed
`water_uL` is summed within each reward event, and each sample is classified into a Polars `Enum(["no", "tone",
"yes"])`:

- `no` — no tone active at the sample
- `yes` — tone active and the event delivered water (summed `water_uL` greater than zero)
- `tone` — tone active but the event delivered no water

The temporary tone, active-flag, event-id, and per-event-water columns are dropped after classification.

### System-state code inversion

The `MesoscopeHardwareState` supplies a `system_state_codes` mapping (name to code). The assembly inverts it
(code to name) and casts `system_state` to a Polars Enum built from the mapping's keys, yielding the descriptive
state names (`idle`, `rest`, `run`) used by the downstream special cases and by masking. If the hardware state is
missing `system_state_codes`, assembly raises a `ValueError`.

### Torque, distance, and brake special cases

- **Torque zeroed during run:** if `torque_N_cm` is present, its value is set to `0.0` wherever `system_state` is
  `run` (the torque sensor is disabled in the run state).
- **Distance clamping and holding:** if `distance_cm` is present, the cumulative traveled distance is held constant
  outside the run state. The initial idle distance (before the system has ever left idle) is forced to `0.0`,
  distance updates only while `system_state` is `run`, intermediate non-run samples forward-fill the last run value,
  and any remaining nulls fill to `0.0`. The running speed is simultaneously set to `0.0` outside the run state.
- **Brake binary threshold:** the brake torque is interpolated (`is_discrete=True`) and converted to a binary
  `uint8` `brake` column by comparing it against the hardware state's `minimum_brake_strength`
  (`brake = brake_torque > minimum_brake_strength`). If the hardware state is missing `minimum_brake_strength`,
  assembly raises a `ValueError`.

---

## Behavior-dataset column order

After cleanup, the behavior DataFrame is reordered to this canonical sequence; only columns that actually exist for
the session are selected (so lick-training and run-training sessions omit the columns whose source feathers are
absent):

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

When `drop_time_columns=True` (how the forging pipeline calls it), the first two entries (`time_us`,
`elapsed_minutes`) are removed from the selection before returning.

---

## Runtime-dataset distance-indexed interpolation and trigger zones

`assemble_runtime_dataset` reads the runtime feather files and the `MesoscopeExperimentConfiguration`, then aligns
each source. Unlike the behavior assembly, several runtime columns are indexed by **traveled distance** rather than
by time:

1. The encoder's traveled distance is interpolated onto `reference_time` (`is_discrete=False`) to produce a
   `reference_distance` vector — the cumulative distance at each fluorescence frame.
2. `trial`, `trial_type`, and `cue` are then interpolated against `reference_distance` (their source coordinates are
   the per-trial / per-cue traveled-distance values, not timestamps), all with `is_discrete=True`:

| Aligned column | Source feather              | Source coordinates              | Source values        |
|----------------|-----------------------------|---------------------------------|----------------------|
| `trial`        | `trial_data.feather`        | per-trial `traveled_distance_cm` | sequential 1-based trial numbers (`uint32`), cast to `UInt16` |
| `trial_type`   | `trial_data.feather`        | per-trial `traveled_distance_cm` | `trial_type_index`, mapped to an Enum (below) |
| `cue`          | `vr_cue_data.feather`       | per-cue `traveled_distance_cm`   | `vr_cue`             |

3. `runtime_state` is interpolated against `reference_time` (`is_discrete=True`) from `runtime_state_data.feather`.

The trial sources are produced upstream by runtime cue-to-trial decomposition — see
`mesoscope:mesoscope-vr-trial-decomposition`.

### Trigger-zone membership

`in_trigger_zone` is computed by the numba-accelerated `_check_trigger_zones`, which marks each `reference_distance`
point `1` (inside) or `0` (outside) by walking the per-trial trigger-zone start/end boundaries from
`vr_trigger_zone_data.feather`. It relies on both the distance series and the trigger-zone boundaries being sorted
and monotonically increasing, walks a zone index forward as distance increases (and backward to absorb slight
non-monotonicity), and returns an all-zero array when no trigger zones are defined.

### Enum mappings

The experiment configuration supplies the categorical mappings:

- `trial_type` is mapped from integer index to name via `enumerate(experiment_configuration.trial_structures.keys())`
  and cast to a Polars Enum whose categories are the trial-type names plus the literal `"undefined"` (reserved for
  masking).
- `runtime_state` is mapped from each experiment state's `experiment_state_code` to its state name, with code `0`
  added as the default `idle` state, and cast to a Polars Enum of those names.

### Conditional guidance-state columns

The two guidance columns are added only when their feather files were produced for the session:

| Aligned column        | Source feather                              | `is_discrete` |
|-----------------------|---------------------------------------------|---------------|
| `reinforcing_guided`  | `reinforcing_guidance_state_data.feather`   | `True`        |
| `aversive_guided`     | `aversive_guidance_state_data.feather`      | `True`        |

Each guidance source value is cast to `uint8` before interpolation.

---

## Sentinel masking and conditional columns

`_mask_non_run_experiment_data` runs over the horizontally concatenated frame (so it can read `system_state`). Where
`system_state` is `idle` or `rest`, the runtime columns are not meaningful and are replaced with sentinels that sit
outside every legitimate value range:

| Column       | Masked value | Sentinel rationale                                                                 |
|--------------|--------------|-----------------------------------------------------------------------------------|
| `cue`        | `255`        | `_CUE_UNDEFINED` — maximum of `UInt8`, outside the valid cue-code range            |
| `trial`      | `65535`      | `_TRIAL_UNDEFINED` — maximum of `UInt16`, outside the trial-ID range for any session |
| `trial_type` | `"undefined"` | the dedicated Enum member added to the trial-type categories                       |

Only `idle` and `rest` are masked; `run` samples keep their interpolated values. The masking is applied
column-by-column with `pl.when(is_non_run).then(<sentinel>).otherwise(<column>)`, preserving each column's dtype.

The optional `reinforcing_guided` and `aversive_guided` columns are present in the assembled feather only when their
upstream guidance feathers existed; all other assembly columns are present in every forged Mesoscope-VR session
(subject to the behavior-dataset session-type omissions described above).

---

## VR configuration as a forged-dataset artifact

A forged dataset re-exports each session's `vr_configuration.yaml` next to the assembled `data.feather`, and dataset
inspection treats that copy as required only when the dataset's own `session_type` belongs to
`SESSION_TYPES_USING_VR_TASK`. That frozen set holds exactly one member today, `SessionTypes.MESOSCOPE_EXPERIMENT`,
so `vr_configuration.yaml` is a required dataset artifact only for datasets whose `session_type` is
`"mesoscope experiment"`. In a dataset of lick-training, run-training, or window-checking sessions, a missing
`vr_configuration.yaml` describes the source sessions rather than a defective dataset.

The registry-dispatched mechanism behind that rule, the `DatasetData` marker it is read from, and the inspection
report that applies it belong to `assets:datasets`.

---

## Related skills

| Skill                                           | Relationship                                                                                |
|-------------------------------------------------|---------------------------------------------------------------------------------------------|
| `forging:data-processing-design`                | Owns the agnostic batch orchestration and stage seam this skill concretizes                 |
| `forging:dataset-forging-results`               | Owns the forged-feather output schema at the reference level; points here for the algorithm |
| `forging:dataset-definition`                    | Composes a dataset from filtered sessions and owns its forging job state                    |
| `assets:datasets`                               | Owns the `DatasetData` marker, the dataset layout, and the seven slsa dataset tools         |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | Produces the `reference_time` vector this stage aligns onto                                 |
| `mesoscope:mesoscope-vr-trial-decomposition`    | Produces the `trial` / `cue` / `vr_trigger_zone` feathers consumed by the runtime assembly  |
| `mesoscope:mesoscope-vr-module-parsing`         | Produces the per-module behavior feathers consumed by the behavior assembly                 |
| `mesoscope:mesoscope-vr-processing-schema`      | Owns the `DatasetColumn` and `BehaviorDataFiles` enum definitions this stage instantiates   |

---

## Verification checklist

```text
- [ ] Every interpolation claim (is_discrete True/False, source vs reference, distance vs time) matches
      behavior_dataset.py / runtime_dataset.py
- [ ] Running-speed window stated as 100 ms (_RUNNING_SPEED_WINDOW_US = 100_000 us) over original encoder samples
- [ ] Reward classes named exactly no / tone / yes with the correct water-volume condition
- [ ] Sentinels stated as cue 255 (_CUE_UNDEFINED), trial 65535 (_TRIAL_UNDEFINED), trial_type "undefined";
      masking applied only for idle and rest states over the concatenated frame
- [ ] Behavior column order quoted exactly; drop_time_columns removes time_us and elapsed_minutes
- [ ] Conditional columns (encoder/screen/brake/torque, reinforcing_guided, aversive_guided) gated on file existence
- [ ] No invented symbols, filenames, tolerances, or event codes — all derived from the cited source files
- [ ] Behavior sources homed in processed_data/microcontroller_data and processed_data/runtime_data, hardware state
      in raw_data/hardware_state.yaml; no processed_data/behavior_data claim
- [ ] vr_configuration.yaml stated as a required dataset artifact only for the "mesoscope experiment" session type,
      the sole SESSION_TYPES_USING_VR_TASK member
- [ ] Did not redefine the DatasetColumn / BehaviorDataFiles enums — handed off to
      mesoscope:mesoscope-vr-processing-schema
- [ ] Did not restate the DatasetData marker, the dataset layout, or the dataset tools — handed off to
      assets:datasets
- [ ] No reStructuredText specifiers; cross-references use the plugin:skill syntax
```
