---
name: mesoscope-vr-processing-schema
description: >-
  Documents the Mesoscope-VR metadata schema in metadata.py: the BehaviorDataFiles and VideoDataFiles filename
  rosters, the fifty-member DatasetColumn roster with its per-session-type presence matrix, and the derived
  MESOSCOPE_COLUMN_DESCRIPTIONS mapping donated to the forging assembly registry. Use when you need the master
  filename or assembled-column roster, when adding a processed feather or an assembled column, or when verifying
  that a producer's output names match the contract.
user-invocable: false
---

# Mesoscope-VR processing schema

Documents the schema rosters that bind the Mesoscope-VR processing pipelines to the Mesoscope-VR forging pipeline:
the `BehaviorDataFiles` and `VideoDataFiles` filename rosters, the `DatasetColumn` assembled-column roster, and the
derived `MESOSCOPE_COLUMN_DESCRIPTIONS` mapping, all declared in `sollertia_forgery.mesoscope_vr.metadata`.

The three enumerations are package-internal. None of them appears in `sollertia_forgery.mesoscope_vr.__all__`, so code
outside the package addresses their members by value rather than importing the enumeration. The derived
`MESOSCOPE_COLUMN_DESCRIPTIONS` mapping is the one asset in this module that crosses the package boundary, and it
reaches the agnostic forging pipeline through the `_FORGING_ASSEMBLY_REGISTRY` seam.

---

## Scope

**Covers:**
- `BehaviorDataFiles`: the fifteen behavior feather filenames the donated parsers write into two processed-data
  directories
- `VideoDataFiles`: the five per-camera video feather filenames written into `processed_data/video_data`
- `DatasetColumn`: the fifty assembled-session column names, their five declaration groups, and the presence
  condition attached to each group
- `MESOSCOPE_COLUMN_DESCRIPTIONS`, its import-time completeness enforcement, and the registry seam it fills
- Which producer writes each roster member and which assembly stage reads it back

**Does not cover:**
- The internal column schema of each processed module feather and the per-module conversions behind it. Owned by
  `/mesoscope-vr-module-parsing`.
- The runtime feather schemas and the trial decomposition that fills them. Owned by `/mesoscope-vr-trial-decomposition`.
- The assembly algorithms that compute the `DatasetColumn` values, the sub-dataset concatenation, and the clipping
  applied afterwards. Owned by `/mesoscope-vr-dataset-assembly`.
- The pupil-metric computation behind the face-camera pupil feather. Owned by `/mesoscope-vr-video-tracking`.
- The forged dataset directory layout and the output verification path. Owned by `forging:processing-results`.

---

## BehaviorDataFiles: the processed behavior feather roster

`BehaviorDataFiles` is a `StrEnum` whose values are the canonical filenames of the behavior feather files the donated
Mesoscope-VR parsers write, and the donated assembly worker reads most of them back. There are two output homes, not
one. The microcontroller parsers write the module feathers into a session's `processed_data/microcontroller_data`
directory, and the runtime parser writes its feathers into `processed_data/runtime_data`. There is no
`processed_data/behavior_data` directory, because `behavior_data` is the raw-side DataLogger archive directory under
`raw_data`. Because the type is a `StrEnum`, every member is usable directly as a path component, for example
`output_directory / BehaviorDataFiles.SYSTEM_STATE`.

| Member                 | Value                                     | Captures                                                                     |
|------------------------|-------------------------------------------|------------------------------------------------------------------------------|
| `ENCODER`              | `encoder_data.feather`                    | Traveled-distance time series derived from the running-wheel encoder         |
| `VALVE`                | `valve_data.feather`                      | Cumulative dispensed water volume and the reward tone state                  |
| `GAS_PUFF`             | `gas_puff_data.feather`                   | Aversive-stimulus dispensing events from the gas puff valve module           |
| `LICK`                 | `lick_data.feather`                       | Raw 12-bit ADC sensor voltage and the thresholded lick state derived from it |
| `BRAKE`                | `brake_data.feather`                      | Instantaneous brake torque applied to the running wheel                      |
| `TORQUE`               | `torque_data.feather`                     | Instantaneous torque exerted on the running wheel by the animal              |
| `SCREEN`               | `screen_data.feather`                     | VR screen on/off state transitions                                           |
| `MESOSCOPE_FRAME`      | `mesoscope_frame_data.feather`            | TTL module mesoscope scan-frame pulse edges, used to align fluorescence      |
| `SYSTEM_STATE`         | `system_state_data.feather`               | System-state code transitions from the acquisition runtime's log archive     |
| `RUNTIME_STATE`        | `runtime_state_data.feather`              | Experiment-state code transitions from the acquisition runtime's log archive |
| `REINFORCING_GUIDANCE` | `reinforcing_guidance_state_data.feather` | Reinforcing guidance-state transitions, written only when any were recorded  |
| `AVERSIVE_GUIDANCE`    | `aversive_guidance_state_data.feather`    | Aversive guidance-state transitions, written only when any were recorded     |
| `VR_CUE`               | `vr_cue_data.feather`                     | VR wall-cue transitions along the corridor                                   |
| `VR_TRIGGER_ZONE`      | `vr_trigger_zone_data.feather`            | VR trigger-zone entry and exit events                                        |
| `TRIAL`                | `trial_data.feather`                      | Per-trial trial type index and traveled distance at trial start              |

---

## VideoDataFiles: the per-camera video feather roster

`VideoDataFiles` is a second `StrEnum` holding the canonical filenames of the per-camera video feathers, all of them
written into a session's `processed_data/video_data` directory under the acquisition-time camera names. The roster
carries five members rather than six because `video_dataset.py` binds a fixed two-camera source set, and only the face
camera declares a pupil file.

| Member                   | Value                            | Captures                                                       |
|--------------------------|----------------------------------|----------------------------------------------------------------|
| `FACE_CAMERA_TIMESTAMPS` | `face_camera_timestamps.feather` | One frame-acquisition timestamp per recorded face-camera frame |
| `FACE_CAMERA_ENERGY`     | `face_camera_energy.feather`     | Per-frame face-camera motion energy and frame luminance        |
| `FACE_CAMERA_PUPIL`      | `face_camera_pupil.feather`      | Per-frame face-camera pupil and eye metrics                    |
| `BODY_CAMERA_TIMESTAMPS` | `body_camera_timestamps.feather` | One frame-acquisition timestamp per recorded body-camera frame |
| `BODY_CAMERA_ENERGY`     | `body_camera_energy.feather`     | Per-frame body-camera motion energy and frame luminance        |

The timestamp and energy feathers come from the acquisition-system-agnostic video pipeline, while the pupil feather is
written by `process_mesoscope_video_tracking` in `mesoscope_vr/video_tracking.py`. All five are read back by
`assemble_video_dataset` in `mesoscope_vr/video_dataset.py`.

---

## DatasetColumn: the assembled-session column roster

`DatasetColumn` is a `StrEnum` defining every column that can appear in the single assembled session `data.feather`
the forging pipeline produces. Each member's value is the column name exactly as it appears in that file, so members
compare equal to the raw column strings. The source declares fifty members in five comment-delimited groups, one group
per emitting assembly stage.

| # | Group                       | Members | Emitting assembly stage                                                 |
|---|-----------------------------|---------|-------------------------------------------------------------------------|
| 1 | Behavior alignment          | 11      | Behavior assembly, except the two time columns on mesoscope experiments |
| 2 | Runtime and experiment      | 7       | Runtime assembly                                                        |
| 3 | cindra fluorescence         | 9       | Fluorescence assembly                                                   |
| 4 | Video motion energy         | 4       | Video assembly                                                          |
| 5 | Pupil tracking, face camera | 19      | Video assembly, from the face-camera pupil feather                      |

`TIME_US` and `ELAPSED_MINUTES` are grouped with the behavior columns but sourced per session type. They come from the
fluorescence assembly for mesoscope experiment sessions and from the behavior assembly for training sessions.

### Group 1: behavior alignment columns

| Member            | Value             | Description                                                                                                                                                                                                                     |
|-------------------|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `TIME_US`         | `time_us`         | Microsecond-precision sample timestamps from the acquisition reference clock                                                                                                                                                    |
| `ELAPSED_MINUTES` | `elapsed_minutes` | Elapsed minutes since the first sample of the reference clock. The clock starts during setup, so the clipped dataset's first row carries a value above zero                                                                     |
| `BRAKE`           | `brake`           | The running wheel brake engagement at each sample                                                                                                                                                                               |
| `SCREENS`         | `screens`         | The Virtual Reality display state at each sample                                                                                                                                                                                |
| `TORQUE_N_CM`     | `torque_N_cm`     | The torque applied by the animal to the running wheel in N·cm at each sample, forced to zero during 'run' periods upstream                                                                                                      |
| `DISTANCE_CM`     | `distance_cm`     | Cumulative distance traveled by the animal in centimeters at each sample                                                                                                                                                        |
| `SPEED_CM_S`      | `speed_cm_s`      | Animal's running speed in cm/s at each sample                                                                                                                                                                                   |
| `LICK`            | `lick`            | Lick sensor engagement state at each sample                                                                                                                                                                                     |
| `WATER_UL`        | `water_uL`        | The cumulative water reward volume delivered to the animal at each sample in microliters                                                                                                                                        |
| `REWARD`          | `reward`          | The reward-classification state at each sample, one of 'no' (no reward tone playing), 'tone' (reward tone playing with no water delivered during the tone), or 'yes' (reward tone playing with water delivered during the tone) |
| `SYSTEM_STATE`    | `system_state`    | Acquisition system state at each sample (idle, rest, run for experiment sessions, lick training or run training for training sessions)                                                                                          |

### Group 2: runtime and experiment columns

| Member               | Value                | Description                                                                                    |
|----------------------|----------------------|------------------------------------------------------------------------------------------------|
| `TRIAL`              | `trial`              | One-based trial identifier at each sample. 65535 marks samples outside the run state           |
| `TRIAL_TYPE`         | `trial_type`         | Trial type label at each sample (e.g. 'ABC', 'ABCD'). 'undefined' marks non-run samples        |
| `CUE`                | `cue`                | Active virtual reality cue identifier at each sample. 255 marks samples outside the run state  |
| `IN_TRIGGER_ZONE`    | `in_trigger_zone`    | Boolean flag indicating whether the animal is inside a stimulus trigger zone at each sample    |
| `RUNTIME_STATE`      | `runtime_state`      | Experiment runtime state label at each sample                                                  |
| `REINFORCING_GUIDED` | `reinforcing_guided` | Reinforcing guidance state at each sample. Present only when reinforcing guidance was recorded |
| `AVERSIVE_GUIDED`    | `aversive_guided`    | Aversive guidance state at each sample. Present only when aversive guidance was recorded       |

The out-of-run-state sentinels are distinct values, `65535` for `trial` (the maximum UInt16) and `255` for `cue` (the
maximum UInt8). The masking pass that assigns them belongs to `/mesoscope-vr-dataset-assembly`.

### Group 3: cindra fluorescence columns

| Member                               | Value                                | Description                                                                                                                                             |
|--------------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| `FRAME`                              | `frame`                              | One-based mesoscope acquisition frame index at each sample                                                                                              |
| `SINGLE_DAY_CELL_FLUORESCENCE`       | `single_day_cell_fluorescence`       | Single-recording raw cell fluorescence trace per ROI, over the cells detected in this session alone                                                     |
| `SINGLE_DAY_NEUROPIL_FLUORESCENCE`   | `single_day_neuropil_fluorescence`   | Single-recording raw neuropil fluorescence trace per ROI, over the cells detected in this session alone                                                 |
| `SINGLE_DAY_SUBTRACTED_FLUORESCENCE` | `single_day_subtracted_fluorescence` | Single-recording neuropil- and baseline-subtracted fluorescence trace per ROI, over the cells detected in this session alone                            |
| `SINGLE_DAY_SPIKES`                  | `single_day_spikes`                  | Single-recording OASIS-deconvolved spike rates per ROI, over the cells detected in this session alone                                                   |
| `MULTI_DAY_CELL_FLUORESCENCE`        | `multi_day_cell_fluorescence`        | Multi-recording raw cell fluorescence trace per ROI, over the cells tracked across every session of this animal in the dataset                          |
| `MULTI_DAY_NEUROPIL_FLUORESCENCE`    | `multi_day_neuropil_fluorescence`    | Multi-recording raw neuropil fluorescence trace per ROI, over the cells tracked across every session of this animal in the dataset                      |
| `MULTI_DAY_SUBTRACTED_FLUORESCENCE`  | `multi_day_subtracted_fluorescence`  | Multi-recording neuropil- and baseline-subtracted fluorescence trace per ROI, over the cells tracked across every session of this animal in the dataset |
| `MULTI_DAY_SPIKES`                   | `multi_day_spikes`                   | Multi-recording OASIS-deconvolved spike rates per ROI, over the cells tracked across every session of this animal in the dataset                        |

### Group 4: video motion-energy columns

| Member                        | Value                         | Description                                                                                                                                                                                 |
|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `FACE_CAMERA_MOTION_ENERGY`   | `face_camera_motion_energy`   | Face camera motion energy at each sample, the mean absolute inter-frame intensity change in gray levels. A within-session movement magnitude, high during movement and low during stillness |
| `FACE_CAMERA_FRAME_LUMINANCE` | `face_camera_frame_luminance` | Face camera mean frame intensity at each sample in gray levels, tracking scene illumination                                                                                                 |
| `BODY_CAMERA_MOTION_ENERGY`   | `body_camera_motion_energy`   | Body camera motion energy at each sample, the mean absolute inter-frame intensity change in gray levels. A within-session movement magnitude, high during movement and low during stillness |
| `BODY_CAMERA_FRAME_LUMINANCE` | `body_camera_frame_luminance` | Body camera mean frame intensity at each sample in gray levels, tracking scene illumination                                                                                                 |

Motion energy is a within-session magnitude. Its gray-level scale depends on the camera gain and the scene, so
comparing it across sessions or across cameras is unsound.

### Group 5: pupil-tracking columns

Every member of this group is derived from the face camera alone, and every one of them carries the face-camera pixel
frame of reference unless its description names a different unit.

| Member                         | Value                          | Description                                                                                                                                                 |
|--------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `PUPIL_CENTER_X_PX`            | `pupil_center_x_px`            | Horizontal position of the fitted pupil ellipse center in face-camera pixels at each sample                                                                 |
| `PUPIL_CENTER_Y_PX`            | `pupil_center_y_px`            | Vertical position of the fitted pupil ellipse center in face-camera pixels at each sample                                                                   |
| `PUPIL_DIAMETER_PX`            | `pupil_diameter_px`            | Mean of the fitted pupil ellipse's two axis diameters in face-camera pixels at each sample, the primary arousal proxy. NaN marks blink and dilation samples |
| `PUPIL_AREA_PX2`               | `pupil_area_px2`               | Area enclosed by the fitted pupil ellipse in square face-camera pixels at each sample                                                                       |
| `PUPIL_FIT_CONDITION`          | `pupil_fit_condition`          | Condition number of the fitted pupil ellipse at each sample, where higher values indicate a less reliable fit. NaN marks unmeasured samples                 |
| `PUPIL_FIT_RESIDUAL_PX`        | `pupil_fit_residual_px`        | Root-mean-square distance in pixels between the confident pupil-perimeter points and the fitted ellipse at each sample, a fit-quality measure               |
| `EYE_CENTER_X_PX`              | `eye_center_x_px`              | Horizontal position of the fitted eye ellipse center in face-camera pixels at each sample                                                                   |
| `EYE_CENTER_Y_PX`              | `eye_center_y_px`              | Vertical position of the fitted eye ellipse center in face-camera pixels at each sample                                                                     |
| `EYE_WIDTH_PX`                 | `eye_width_px`                 | Length of the fitted eye ellipse's left-right chord in face-camera pixels at each sample                                                                    |
| `EYE_HEIGHT_PX`                | `eye_height_px`                | Length of the fitted eye ellipse's top-bottom chord in face-camera pixels at each sample                                                                    |
| `EYE_OPENNESS`                 | `eye_openness`                 | Ratio of the fitted eye ellipse's height to its width at each sample, a distance-invariant measure of eye openness                                          |
| `BLINKING_STATE`               | `blinking_state`               | Blink state at each sample, marking a closed or covered eye. Encoded as 1 during a blink and 0 otherwise                                                    |
| `DILATION_STATE`               | `dilation_state`               | Dilation-clip state at each sample, marking a pupil dilated past the eye aperture. Encoded as 1 when clipped and 0 otherwise                                |
| `REFLECTION_X_PX`              | `reflection_x_px`              | Horizontal position of the corneal reflection in face-camera pixels at each sample                                                                          |
| `REFLECTION_Y_PX`              | `reflection_y_px`              | Vertical position of the corneal reflection in face-camera pixels at each sample                                                                            |
| `PUPIL_REFLECTION_OFFSET_X_PX` | `pupil_reflection_offset_x_px` | Horizontal offset of the pupil center from the corneal reflection in pixels at each sample, a motion-robust horizontal eye-position signal                  |
| `PUPIL_REFLECTION_OFFSET_Y_PX` | `pupil_reflection_offset_y_px` | Vertical offset of the pupil center from the corneal reflection in pixels at each sample, a motion-robust vertical eye-position signal                      |
| `PUPIL_IN_EYE_X`               | `pupil_in_eye_x`               | Horizontal offset of the pupil center from the eye center at each sample, normalized to the eye ellipse's horizontal semi-axis and dimensionless            |
| `PUPIL_IN_EYE_Y`               | `pupil_in_eye_y`               | Vertical offset of the pupil center from the eye center at each sample, normalized to the eye ellipse's vertical semi-axis and dimensionless                |

---

## Column presence by session type

Only six behavior columns are present in every forged session, so any consumer reading a column outside that set must
first confirm the column exists in the loaded frame. The `DatasetColumn` class docstring states the full condition set.

| Column set                                                                 | Presence condition                                          |
|----------------------------------------------------------------------------|-------------------------------------------------------------|
| `TIME_US`, `ELAPSED_MINUTES`, `LICK`, `WATER_UL`, `REWARD`, `SYSTEM_STATE` | Present in every forged session                             |
| `TRIAL`, `TRIAL_TYPE`, `CUE`, `IN_TRIGGER_ZONE`, `RUNTIME_STATE`           | Mesoscope experiment sessions only                          |
| `FRAME` and every `SINGLE_DAY_*` and `MULTI_DAY_*` member                  | Mesoscope experiment sessions only                          |
| `BRAKE`, `SCREENS`                                                         | Mesoscope experiment sessions only                          |
| `REINFORCING_GUIDED`, `AVERSIVE_GUIDED`                                    | Only when the corresponding guidance events were recorded   |
| `TORQUE_N_CM`                                                              | Absent for run training                                     |
| `DISTANCE_CM`, `SPEED_CM_S`                                                | Absent for lick training                                    |
| The four per-camera video columns                                          | Only when that camera's feathers were produced              |
| The nineteen pupil columns                                                 | Only when the face camera's pose predictions were processed |

The per-camera video condition is per camera, not per group. A session whose body camera recorded and whose face
camera did not carries the two `BODY_CAMERA_*` columns and neither `FACE_CAMERA_*` column.

---

## MESOSCOPE_COLUMN_DESCRIPTIONS: the donated description mapping

`metadata.py` declares a module-private `_COLUMN_DESCRIPTIONS: dict[DatasetColumn, str]` carrying one human-readable
description per member, then derives the public mapping from it:

```python
MESOSCOPE_COLUMN_DESCRIPTIONS: dict[str, str] = {column.value: _COLUMN_DESCRIPTIONS[column] for column in DatasetColumn}
```

The comprehension is the completeness check. It walks every `DatasetColumn` member and indexes `_COLUMN_DESCRIPTIONS`
with it, so a member with no description entry raises `KeyError` at module import. There is no guarded raise and no
custom message here, unlike the coverage checks in `registries.py` and `mesoscope_vr/two_photon.py`. Adding a
`DatasetColumn` member therefore requires adding its `_COLUMN_DESCRIPTIONS` entry in the same edit, or the package
stops importing.

The derived mapping is keyed by the column value rather than by the enumeration member, which is what lets an agnostic
consumer use it without importing `DatasetColumn`. It is registered on the `_FORGING_ASSEMBLY_REGISTRY` entry for the
Mesoscope-VR system, alongside the `assemble_mesoscope_session` worker, and the forging pipeline bakes it into each
forged dataset's `data_descriptions.feather` at dataset-definition time.

---

## Producer-to-roster and roster-to-consumer mapping

Each producer writes its members under the exact roster value into the directory named by the `Directories` member in
the last column, and each consumer reads them back from that same directory under the same member. The `Directories`
enumeration itself, and the `ProcessedData` path properties that resolve its members against a session root, belong to
`assets:session-data`.

| Roster member                            | Producing concern | Producing module      | Owning `Directories` member |
|------------------------------------------|-------------------|-----------------------|-----------------------------|
| `BehaviorDataFiles.ENCODER`              | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.VALVE`                | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.GAS_PUFF`             | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.LICK`                 | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.BRAKE`                | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.TORQUE`               | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.SCREEN`               | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.MESOSCOPE_FRAME`      | Module parsing    | `microcontrollers.py` | `MICROCONTROLLER_DATA`      |
| `BehaviorDataFiles.SYSTEM_STATE`         | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `BehaviorDataFiles.RUNTIME_STATE`        | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `BehaviorDataFiles.REINFORCING_GUIDANCE` | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `BehaviorDataFiles.AVERSIVE_GUIDANCE`    | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `BehaviorDataFiles.VR_CUE`               | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `BehaviorDataFiles.VR_TRIGGER_ZONE`      | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `BehaviorDataFiles.TRIAL`                | Runtime parsing   | `runtime.py`          | `RUNTIME_DATA`              |
| `VideoDataFiles.FACE_CAMERA_TIMESTAMPS`  | Video processing  | agnostic video worker | `VIDEO_DATA`                |
| `VideoDataFiles.FACE_CAMERA_ENERGY`      | Video processing  | agnostic video worker | `VIDEO_DATA`                |
| `VideoDataFiles.FACE_CAMERA_PUPIL`       | Video tracking    | `video_tracking.py`   | `VIDEO_DATA`                |
| `VideoDataFiles.BODY_CAMERA_TIMESTAMPS`  | Video processing  | agnostic video worker | `VIDEO_DATA`                |
| `VideoDataFiles.BODY_CAMERA_ENERGY`      | Video processing  | agnostic video worker | `VIDEO_DATA`                |

`parse_mesoscope_frame` is one of the eight microcontroller module parsers, so `MESOSCOPE_FRAME` is a module-parsing
output like the other seven. It is listed here as such even though its only consumer is the fluorescence assembly.

Four assembly stages read the two filename rosters back, and the union of their outputs is the forged `data.feather`:

| Assembly stage        | Assembling module       | Roster members read                                                                                                  | `DatasetColumn` group emitted       |
|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| Behavior assembly     | `behavior_dataset.py`   | `SYSTEM_STATE`, `LICK`, `VALVE` always, plus `ENCODER`, `SCREEN`, `BRAKE`, `TORQUE` when present                     | Behavior alignment                  |
| Runtime assembly      | `runtime_dataset.py`    | `ENCODER`, `VR_CUE`, `VR_TRIGGER_ZONE`, `TRIAL`, `RUNTIME_STATE` always, plus the two guidance feathers when present | Runtime and experiment              |
| Fluorescence assembly | `two_photon_dataset.py` | `MESOSCOPE_FRAME`, alongside the cindra single-recording and multi-recording arrays                                  | cindra fluorescence                 |
| Video assembly        | `video_dataset.py`      | Every `VideoDataFiles` member present under `video_data`                                                             | Video motion energy, pupil tracking |

Two roster facts do not fit either table. `GAS_PUFF` is written by the module parsers and read by no assembler, so it
reaches no `DatasetColumn` member. `SYSTEM_STATE` and `RUNTIME_STATE` are read a second time after concatenation, by
the clipping pass that trims the setup and teardown spans off the assembled frame.

Because `BehaviorDataFiles` spans module parsing and runtime parsing, `VideoDataFiles` spans the agnostic video worker
and the Mesoscope-VR tracking pass, and `DatasetColumn` spans four assembly stages, no roster belongs to a single
producing or consuming stage. This skill owns all three, and the producing and consuming skills reference it.

---

## Related skills

| Skill                                  | Relationship                                                                           |
|----------------------------------------|----------------------------------------------------------------------------------------|
| `/mesoscope-vr-module-parsing`         | Producer of the eight microcontroller feathers and owner of their column schemas       |
| `/mesoscope-vr-trial-decomposition`    | Producer of the seven runtime feathers and owner of their column schemas               |
| `/mesoscope-vr-video-tracking`         | Producer of the face-camera pupil feather and owner of the pupil-metric definitions    |
| `/mesoscope-vr-fluorescence-alignment` | Consumer of `MESOSCOPE_FRAME`, producer of the cindra fluorescence columns             |
| `/mesoscope-vr-dataset-assembly`       | Consumer: assembles the roster feathers into the `DatasetColumn` set of `data.feather` |
| `forging:dataset-definition`           | Consumer: bakes `MESOSCOPE_COLUMN_DESCRIPTIONS` into `data_descriptions.feather`       |
| `forging:processing-results`           | Reference: the forged dataset layout and the output verification path                  |
| `forging:data-processing-design`       | Reference: the registry seams every Mesoscope-VR donation fills                        |
| `assets:session-data`                  | Owner of `Directories` and the `ProcessedData` path properties                         |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Roster fidelity:
- [ ] BehaviorDataFiles lists 15 members and VideoDataFiles lists 5, matching metadata.py exactly
- [ ] DatasetColumn lists 50 members across the 5 comment-delimited groups declared in metadata.py
- [ ] Every column value matches its source spelling, including torque_N_cm, water_uL, and pupil_area_px2
- [ ] Every member description matches its metadata.py docstring, with no invented unit, transform, or sensor
- [ ] MESOSCOPE_COLUMN_DESCRIPTIONS is the exported asset and the three enumerations are package-internal
- [ ] The import-time completeness check is described as a bare KeyError from the dict comprehension

Presence and placement:
- [ ] Only time_us, elapsed_minutes, lick, water_uL, reward, and system_state are claimed unconditional
- [ ] Microcontroller feathers homed in processed_data/microcontroller_data, runtime feathers in runtime_data
- [ ] Video feathers homed in processed_data/video_data, and no processed_data/behavior_data claim appears
- [ ] GAS_PUFF is stated to have no assembler consumer, and no DatasetColumn member is invented for it
- [ ] No producing logic, assembly algorithm, or output-verification content is authored here
```
