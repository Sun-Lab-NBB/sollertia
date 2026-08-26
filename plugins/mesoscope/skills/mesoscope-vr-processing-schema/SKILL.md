---
name: mesoscope-vr-processing-schema
description: >-
  Documents the two pipeline-wide Mesoscope-VR processing-schema rosters in metadata.py: BehaviorDataFiles (the
  canonical feather filenames produced across module parsing, frame extraction, and runtime decomposition) and
  DatasetColumn (every column in the assembled session data.feather, spanning behavior, runtime, and fluorescence).
  Use when you need the master filename or assembled-column roster, when adding a new processed feather or assembled
  column, or when verifying that a producer's output names match the contract.
user-invocable: false
---

# Mesoscope-VR processing schema

Documents the two pipeline-wide schema rosters that bind the Mesoscope-VR processing pipeline to the Mesoscope-VR
forging pipeline: the `BehaviorDataFiles` filename roster and the `DatasetColumn` assembled-column roster, both
defined in `sollertia_forgery.mesoscope_vr.metadata`.

These two rosters are the cross-cutting contract between processing and forging. `BehaviorDataFiles` enumerates the
feather filenames the producing concerns write into a session's `processed_data/microcontroller_data` and
`processed_data/runtime_data` directories. `DatasetColumn` enumerates every column the forging pipeline can emit into
the single assembled `data.feather`. Both rosters are forgery-internal: neither is re-exported from
`sollertia_forgery.mesoscope_vr` or from the package root, so downstream code addresses their members by value rather
than importing the enumerations.

This skill is the canonical home of the two rosters. It does not own the producing logic, the assembly algorithm,
or the forged-output reference — those live in the producing and consuming skills cross-referenced below. Hand off
there for behavior, the way columns are computed, or output verification.

---

## Scope

**Covers:**
- `BehaviorDataFiles`: the canonical feather-filename roster spanning module-parsing, frame-extraction, and
  runtime-decomposition outputs
- `DatasetColumn`: the assembled-session column roster spanning behavior, runtime, and fluorescence columns
- How each roster member maps to its producing concern (module parsing, frame extraction, runtime decomposition)
  and its consuming concern (dataset assembly)
- Why these two rosters are pipeline-wide schema contracts rather than belonging to any single producing stage

**Does not cover:**
- Per-module conversions and the internal column schema of each processed feather (see
  `mesoscope:mesoscope-vr-module-parsing`)
- The assembly algorithm that computes the `DatasetColumn` values (see `mesoscope:mesoscope-vr-dataset-assembly`)
- The forged-`data.feather` output reference and verification (see `forging:dataset-forging-results`)

---

## BehaviorDataFiles: the processed feather-filename roster

`BehaviorDataFiles` is a `StrEnum` whose values are the canonical filenames of the behavior feather files the
Mesoscope-VR parsers write and the assembly worker reads back. There are two output homes, not one: the
microcontroller parsers write the module feathers into a session's `processed_data/microcontroller_data` directory,
and the runtime parser writes its feathers into `processed_data/runtime_data`. There is no
`processed_data/behavior_data` directory. `behavior_data` is the raw-side DataLogger archive directory under
`raw_data`, and the producer-to-roster table below carries the per-member mapping. Because the type is a `StrEnum`,
every member is usable directly as a path component (for example `output_directory / BehaviorDataFiles.SYSTEM_STATE`).

| Member                 | Value                                     | Captures                                                        |
|------------------------|-------------------------------------------|-----------------------------------------------------------------|
| `ENCODER`              | `encoder_data.feather`                    | Traveled-distance time series from the running-wheel encoder    |
| `VALVE`                | `valve_data.feather`                      | Water-dispensing events and cumulative dispensed volume         |
| `GAS_PUFF`             | `gas_puff_data.feather`                   | Aversive-stimulus (gas puff) dispensing events                  |
| `LICK`                 | `lick_data.feather`                       | Thresholded lick events from the capacitive sensor              |
| `BRAKE`                | `brake_data.feather`                      | Instantaneous brake torque applied to the running wheel         |
| `TORQUE`               | `torque_data.feather`                     | Instantaneous torque exerted on the wheel by the animal         |
| `SCREEN`               | `screen_data.feather`                     | VR screen on/off state transitions                              |
| `MESOSCOPE_FRAME`      | `mesoscope_frame_data.feather`            | Mesoscope scan-frame TTL pulse edges (for fluorescence align)   |
| `SYSTEM_STATE`         | `system_state_data.feather`               | System-state code transitions from the runtime log archive      |
| `RUNTIME_STATE`        | `runtime_state_data.feather`              | Experiment-state code transitions from the runtime log archive  |
| `REINFORCING_GUIDANCE` | `reinforcing_guidance_state_data.feather` | Reinforcing guidance-state transitions (written only if any)    |
| `AVERSIVE_GUIDANCE`    | `aversive_guidance_state_data.feather`    | Aversive guidance-state transitions (written only if any)       |
| `VR_CUE`               | `vr_cue_data.feather`                     | VR wall-cue transitions along the corridor                      |
| `VR_TRIGGER_ZONE`      | `vr_trigger_zone_data.feather`            | VR trigger-zone entry and exit events                           |
| `TRIAL`                | `trial_data.feather`                      | Per-trial metadata (trial index, type, distance at trial start) |

`REINFORCING_GUIDANCE` and `AVERSIVE_GUIDANCE` are conditional: per the runtime decomposition logic and the member
docstrings, each file is written only when the corresponding guidance events were recorded during the session.

---

## DatasetColumn: the assembled-session column roster

`DatasetColumn` is a `StrEnum` defining every column that can appear in the single assembled session `data.feather`
produced by the forging pipeline. Per its docstring, the optional members (`REINFORCING_GUIDED`, `AVERSIVE_GUIDED`)
appear only when the corresponding upstream events were recorded; every other member is guaranteed to exist in
every forged session. The source groups the members into three assembly origins.

### Behavior alignment columns (forging behavior assembly)

| Member            | Value             | Captures                                                           |
|-------------------|-------------------|--------------------------------------------------------------------|
| `TIME_US`         | `time_us`         | Microsecond-precision sample timestamps from the acquisition clock |
| `ELAPSED_MINUTES` | `elapsed_minutes` | Elapsed session time in minutes since the first sample             |
| `BRAKE`           | `brake`           | Wheel brake engagement at each sample                              |
| `SCREENS`         | `screens`         | Display panel state at each sample                                 |
| `TORQUE_N_CM`     | `torque_N_cm`     | Wheel torque in N·cm at each sample (forced to zero during 'run')  |
| `DISTANCE_CM`     | `distance_cm`     | Cumulative distance traveled by the animal in centimeters          |
| `SPEED_CM_S`      | `speed_cm_s`      | Animal running speed in cm/s at each sample                        |
| `LICK`            | `lick`            | Lick sensor state at each sample                                   |
| `WATER_UL`        | `water_uL`        | Per-sample water reward delivery in microliters                    |
| `REWARD`          | `reward`          | Reward event flag at each sample                                   |
| `SYSTEM_STATE`    | `system_state`    | Acquisition system state at each sample (idle, rest, run)          |

### Runtime/experiment columns (forging runtime assembly)

| Member               | Value                | Captures                                                            |
|----------------------|----------------------|---------------------------------------------------------------------|
| `TRIAL`              | `trial`              | One-based trial identifier; `65535` marks samples outside any trial |
| `TRIAL_TYPE`         | `trial_type`         | Trial type label (e.g. 'ABC'); 'undefined' marks non-run samples    |
| `CUE`                | `cue`                | Active virtual-reality cue identifier at each sample                |
| `IN_TRIGGER_ZONE`    | `in_trigger_zone`    | Whether the animal is inside a stimulus trigger zone at each sample |
| `RUNTIME_STATE`      | `runtime_state`      | Experiment runtime state label at each sample                       |
| `REINFORCING_GUIDED` | `reinforcing_guided` | Optional. Reinforcing guidance state; present only when recorded    |
| `AVERSIVE_GUIDED`    | `aversive_guided`    | Optional. Aversive guidance state; present only when recorded       |

The out-of-trial `trial` sentinel (`65535`, max UInt16) and the undefined-`cue` sentinel (`255`, max UInt8) are
distinct values; the masking algorithm that assigns them is owned by `mesoscope:mesoscope-vr-dataset-assembly`,
not by this roster.

### Fluorescence columns (forging fluorescence assembly)

| Member                               | Value                                | Captures                                                 |
|--------------------------------------|--------------------------------------|----------------------------------------------------------|
| `SINGLE_DAY_CELL_FLUORESCENCE`       | `single_day_cell_fluorescence`       | Single-recording raw cell fluorescence trace per ROI     |
| `SINGLE_DAY_NEUROPIL_FLUORESCENCE`   | `single_day_neuropil_fluorescence`   | Single-recording raw neuropil fluorescence trace per ROI |
| `SINGLE_DAY_SUBTRACTED_FLUORESCENCE` | `single_day_subtracted_fluorescence` | Single-recording neuropil-subtracted dF/F0               |
| `SINGLE_DAY_SPIKES`                  | `single_day_spikes`                  | Single-recording OASIS-deconvolved spike rates per ROI   |
| `MULTI_DAY_CELL_FLUORESCENCE`        | `multi_day_cell_fluorescence`        | Multi-recording raw cell fluorescence trace per ROI      |
| `MULTI_DAY_NEUROPIL_FLUORESCENCE`    | `multi_day_neuropil_fluorescence`    | Multi-recording raw neuropil fluorescence trace per ROI  |
| `MULTI_DAY_SUBTRACTED_FLUORESCENCE`  | `multi_day_subtracted_fluorescence`  | Multi-recording neuropil-subtracted dF/F0 across days    |
| `MULTI_DAY_SPIKES`                   | `multi_day_spikes`                   | Multi-recording OASIS-deconvolved spike rates per ROI    |

The `data.feather` filename itself is fixed by the cross-system dataset layout (`DATA = "data.feather"`), not by
this roster; this roster governs only the column set inside that file.

---

## Producer-to-roster and roster-to-consumer mapping

`BehaviorDataFiles` is written by three producing concerns into two processed-data directories and read back by the
dataset-assembly concern. Each producer writes its members under the exact `BehaviorDataFiles` value into the
directory named by the `Directories` member in the last column, and each consumer reads them back from that same
directory under the same member. The `Directories` enumeration itself, and the `ProcessedData` path properties that
resolve its members against a session root, belong to `assets:session-data`.

| `BehaviorDataFiles` member | Producing concern      | Producing module      | Owning `Directories` member |
|----------------------------|------------------------|-----------------------|-----------------------------|
| `ENCODER`                  | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `VALVE`                    | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `GAS_PUFF`                 | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `LICK`                     | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BRAKE`                    | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `TORQUE`                   | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `SCREEN`                   | Module parsing         | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `MESOSCOPE_FRAME`          | Frame (TTL) extraction | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `SYSTEM_STATE`             | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |
| `RUNTIME_STATE`            | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |
| `REINFORCING_GUIDANCE`     | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |
| `AVERSIVE_GUIDANCE`        | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |
| `VR_CUE`                   | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |
| `VR_TRIGGER_ZONE`          | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |
| `TRIAL`                    | Runtime decomposition  | `runtime.py`          | `RUNTIME_DATA`              |

`MESOSCOPE_FRAME` is produced by the TTL module parser (it carries scan-frame pulse edges), so in the live source
both module parsing and frame extraction are emitted from `microcontrollers.py`. The two are listed separately
because they are distinct producing concerns and own distinct downstream consumers.

This table is the single home of the Mesoscope-VR producer-to-directory mapping. The sibling skills
`mesoscope:mesoscope-vr-module-parsing`, `mesoscope:mesoscope-vr-dataset-assembly`, and
`mesoscope:mesoscope-vr-fluorescence-alignment` reference it rather than restating the layout, so a directory change
lands in one place.

`DatasetColumn` members are produced by the three assembly stages that consume `BehaviorDataFiles` feathers and the
fluorescence assets, and the union of their outputs is the forged `data.feather`:

| `DatasetColumn` group | Assembly stage                | Reads                                                                      |
|-----------------------|-------------------------------|----------------------------------------------------------------------------|
| Behavior alignment    | Forging behavior assembly     | `ENCODER`, `VALVE`, `LICK`, `BRAKE`, `TORQUE`, `SCREEN`, `SYSTEM_STATE`    |
| Runtime/experiment    | Forging runtime assembly      | `ENCODER`, `VR_CUE`, `VR_TRIGGER_ZONE`, `TRIAL`, `RUNTIME_STATE`, guidance |
| Fluorescence          | Forging fluorescence assembly | `MESOSCOPE_FRAME` plus the fluorescence assets                             |

Because `BehaviorDataFiles` spans module parsing, frame extraction, and runtime decomposition, and `DatasetColumn`
spans three assembly stages, neither roster belongs to a single producing or consuming stage. Attaching either to
one stage would split the codebase by module rather than by concern, leaving every other stage referencing a roster
it does not own. This skill therefore owns both rosters, and the producing and consuming skills reference here.

---

## Related skills

| Skill                                           | Relationship                                                                            |
|-------------------------------------------------|-----------------------------------------------------------------------------------------|
| `mesoscope:mesoscope-vr-module-parsing`         | Producer — writes the per-module `BehaviorDataFiles` feathers and owns their schemas    |
| `mesoscope:mesoscope-vr-trial-decomposition`    | Producer of runtime `BehaviorDataFiles`                                                 |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | Producer of the fluorescence `DatasetColumn` values via `MESOSCOPE_FRAME` align         |
| `mesoscope:mesoscope-vr-dataset-assembly`       | Consumer — assembles `BehaviorDataFiles` into the `DatasetColumn` set of `data.feather` |
| `forging:dataset-forging-results`               | Reference and verification for the forged `data.feather` output                         |
| `forging:microcontroller-primitives`            | Owns the five-column microcontroller feather schema the module parsers read upstream    |
| `assets:session-data`                           | Owns `Directories` and the `ProcessedData` path properties                              |

---

## Verification checklist

```text
- [ ] Filename claims match BehaviorDataFiles values in mesoscope_vr/metadata.py exactly (.feather extension)
- [ ] Column claims match DatasetColumn values in mesoscope_vr/metadata.py exactly (e.g. torque_N_cm, water_uL)
- [ ] Guidance feathers and guidance columns (reinforcing and aversive) flagged optional
- [ ] Module-parsed and TTL feathers homed in processed_data/microcontroller_data, runtime feathers in runtime_data
- [ ] No processed_data/behavior_data claim anywhere (behavior_data is the raw-side DataLogger archive directory)
- [ ] No producing logic, assembly algorithm, or output-verification content authored here (handed off via cross-refs)
- [ ] No TrialGeometryEntry or StimulusMode claimed anywhere; neither symbol exists in metadata.py
- [ ] Cross-references use exact plugin:skill syntax (mesoscope:, forging:) with no ataraxis@ prefix
- [ ] Word "feather" used only as the file-format term, never as a module or skill name
```
