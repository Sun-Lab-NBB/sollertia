---
name: mesoscope-vr-trial-decomposition
description: >-
  Documents Mesoscope-VR's concrete runtime-log decomposition: the runtime message codes, the numba-accelerated greedy
  longest-first cue-sequence-to-trial-motif matching, trial-type and cumulative-distance sequence extraction, and the
  task-template trial geometry (trigger zones, cue offsets on sequence restart). Use when interpreting the vr_cue,
  vr_trigger_zone, or trial feathers, or debugging cue-sequence decomposition failures. This is the forging-side runtime
  LOG decomposition, distinct from the acquisition-side runtime behavior layer in mesoscope:mesoscope-vr-runtime.
user-invocable: false
---

# Mesoscope-VR trial decomposition

Decomposes the Mesoscope-VR runtime log archive into the per-trial behavior feathers (`vr_cue_data.feather`,
`vr_trigger_zone_data.feather`, `trial_data.feather`) by matching the recorded virtual-reality wall-cue sequence
against the experiment's trial motifs.

This skill documents the **forging-side runtime LOG decomposition** implemented in
`src/sollertia_forgery/mesoscope_vr/runtime.py`. It is **not** the acquisition-side runtime behavior layer (the state
machine, orchestrator, GUI, and CLI) — that is owned by `mesoscope:mesoscope-vr-runtime`. That layer is a distinct,
non-delegating counterpart: its only notion of "trial decomposition" is the acquisition-time Unity cue-sequence
decomposition, which it delegates to `experiment:vr-driver-interface`, and it never invokes `parse_runtime`. The two
concerns share no producer/consumer relationship. See [Related skills](#related-skills) for the disambiguation.

---

## Scope

**Covers:**
- The runtime message codes that partition the runtime log archive payloads (system state, runtime state,
  reinforcing guidance, aversive guidance, distance snapshot)
- Decomposing one or more virtual-reality wall-cue sequences into trial motifs via numba-accelerated greedy
  longest-first matching, and stitching multiple sequences together across distance breakpoints
- Extracting the trial-type-index sequence and the cumulative trial-end distance sequence
- Mapping the decomposed trials into trial geometry: per-cue cumulative distances, trigger-zone start and end
  boundaries, and the cue offset applied to the first cue after a sequence start or restart
- The output schemas for the `vr_cue_data`, `vr_trigger_zone_data`, and `trial_data` feathers
- Joining the experiment configuration's ordered trial names to the VR task template's spatial trial geometry

**Does not cover:**
- The runtime NPZ log archive format, source ID, and message payload byte layout (see
  `forging:behavior-input-format`)
- The acquisition-side runtime behavior LAYER: state machine, orchestrator, GUI, CLI (see
  `mesoscope:mesoscope-vr-runtime` — NOT this skill)
- The Mesoscope-VR experiment configuration and its `MesoscopeWaterRewardTrial` and `MesoscopeGasPuffTrial` runtime
  trial classes (see `mesoscope:mesoscope-vr-experiment-schema`)
- The `TaskTemplate` schema itself: the cue catalog, the VR environment, and the trial structures (see
  `assets:task-templates`)
- The pipeline-wide `BehaviorDataFiles` and `DatasetColumn` roster enumerations (see
  `mesoscope:mesoscope-vr-processing-schema`)
- Fluorescence frame timestamps used as the assembly reference vector (see
  `mesoscope:mesoscope-vr-fluorescence-alignment`)
- Distance-indexed interpolation of trials into the assembled session feather (see
  `mesoscope:mesoscope-vr-dataset-assembly`)
- The agnostic prepare-then-execute batch orchestration (see `forging:data-processing-design` and
  `forging:behavior-processing`)

---

## Runtime-log decomposition contract

This is the Mesoscope-VR concrete instance of the runtime-log decomposition stage. The agnostic processing doctrine
— the prepare-then-execute batch model, worker-budget concurrency, and the feather-as-interchange contract — is
owned by `forging:data-processing-design`; this skill documents only what Mesoscope-VR does inside that stage.

The single entry point is `parse_runtime(decoded_messages, output_directory, session)`. It receives the already
decoded runtime archive as a Polars DataFrame carrying a `time_us` UInt64 column and a `payload` Binary column in
archive order, partitions every payload by its code, and writes the extracted streams as uncompressed Arrow IPC
`.feather` files into `output_directory`. The parser is registered against `AcquisitionSystems.MESOSCOPE_VR` paired
with `RUNTIME_SOURCE_ID` (the string `"1"`), which locates the session's single `1_log.npz` runtime archive. Every
processable session contains exactly one runtime archive, always under that fixed source ID.

Before it partitions anything, `parse_runtime` resolves two configuration documents off the loaded `SessionData`:

- `MesoscopeExperimentConfiguration`, from `raw_data/experiment_configuration.yaml`. It supplies the ordered trial
  names and the per-trial experiment parameters.
- `TaskTemplate`, from `raw_data/vr_configuration.yaml`. It supplies the spatial trial geometry: the cue catalog, the
  corridor cue offset, and each trial's cue sequence and trigger zone.

Both resolutions are gated on `session.session_type == SessionTypes.MESOSCOPE_EXPERIMENT`, and each raises
`FileNotFoundError` naming the session and the missing path when an experiment session lacks its file. Only session
types listed in `SESSION_TYPES_USING_VR_TASK` carry a `vr_configuration.yaml` at all, and that frozenset holds
exactly `MESOSCOPE_EXPERIMENT`, so the two gates coincide. For every other session type both resolvers return `None`.

The system-state and runtime-state feathers are written for **every** session type. The reinforcing-guidance,
aversive-guidance, and the three trial-decomposition feathers are written **only for** sessions that clear that
session-type gate. The guidance feathers are additionally written only when their corresponding events were actually
recorded.

---

## Runtime message codes

`parse_runtime` classifies each message by inspecting `payload[0]` (the leading code byte), except for VR wall-cue
sequences, which are identified by payload length. A payload longer than `_CUE_SEQUENCE_MIN_LENGTH` (500 bytes) is
treated as a cue sequence and is collected only for experiment sessions. The length alone classifies the message, so
a long payload is consumed by that branch whether or not the session collects it; testing the session type
length would let a wall-cue sequence fall through to the code chain, where its first two cue codes read as a state
transition.

| Concern               | Constant                           | Code | Extracts                                                        |
|-----------------------|------------------------------------|------|-----------------------------------------------------------------|
| VR system state       | `_SYSTEM_STATE_CODE`               | `1`  | `system_state` (uint8 from `payload[1]`) and message timestamp  |
| Session runtime state | `_RUNTIME_STATE_CODE`              | `2`  | `runtime_state` (uint8 from `payload[1]`) and message timestamp |
| Reinforcing guidance  | `_REINFORCING_GUIDANCE_STATE_CODE` | `3`  | `reinforcing_guidance_state` (uint8) and message timestamp      |
| Aversive guidance     | `_AVERSIVE_GUIDANCE_STATE_CODE`    | `4`  | `aversive_guidance_state` (uint8) and message timestamp         |
| Distance snapshot     | `_DISTANCE_SNAPSHOT_CODE`          | `5`  | A float64 traveled distance from `payload[1:9]` (little-endian) |
| VR wall-cue sequence  | (none — length-based)              | n/a  | The full uint8 cue array, when `len(payload) > 500`             |

Message timestamps come from the decoded `time_us` column, which already carries absolute microsecond values because
the decoding stage resolved the archive onsets, so this parser computes no onset offset of its own.
Distance-snapshot messages are logged whenever the active VR wall-cue sequence changes, and their float64 values
become the breakpoints used to stitch multiple cue sequences together (see below).

The system-state and runtime-state streams are written to the feathers named by the `SYSTEM_STATE` and
`RUNTIME_STATE` members of the `BehaviorDataFiles` roster; the guidance streams use `REINFORCING_GUIDANCE` and
`AVERSIVE_GUIDANCE`. The roster definitions themselves are owned by `mesoscope:mesoscope-vr-processing-schema`.

---

## Cue-sequence to trial-motif decomposition

`_decompose_multiple_cue_sequences_into_trials` converts the collected wall-cue sequences into a single ordered list
of trials, taking the experiment configuration, the task template, the cue sequences, and the distance breakpoints.
It handles the case where the original cue sequence was interrupted during runtime and a new sequence was generated,
using the distance breakpoints to stitch the sequences together.

It raises:

- `ValueError` when no cue sequences are provided.
- `ValueError` when there are multiple cue sequences but the number of distance breakpoints is not exactly the
  number of sequences minus one.
- `ValueError` when the experiment configuration names a trial the task template does not define. The message lists
  the missing names alongside the template's available trial names.

**The two documents are joined by trial name.** The experiment configuration owns the ordered trial names, read as
`list(experiment_configuration.trial_structures.keys())`, and that order is what every `trial_type_index` this stage
emits refers to. The task template owns the spatial data: `_resolve_trial_geometries` looks each name up in
`task_template.trial_structures` and returns the matching `TrialStructure` values in configuration order. The split
is deliberate. The template carries only the spatial data Unity needs to build and run the corridor, while the
experiment-specific parameters live on the runtime trial classes `MesoscopeWaterRewardTrial` and
`MesoscopeGasPuffTrial`, whose schema is owned by `mesoscope:mesoscope-vr-experiment-schema`. The `TaskTemplate`
schema is owned by `assets:task-templates`.

Each trial's motif and motif distance are built from the template, never from a stored trial length:

- The **motif** is `task_template.trial_structures[<trial name>].cue_sequence`, a list of cue **names**, mapped to
  uint8 codes through the `{cue.name: cue.code}` index built from `task_template.cues`.
- The **motif distance** is the **sum of those cues' `length_cm` values**, looked up through the matching
  `{cue.name: cue.length_cm}` index. Neither document carries a `trial_length_cm` field.

`_prepare_motif_data` flattens the motifs into a single contiguous array for the numba kernel and **sorts the
motifs by length, longest first**. This longest-first ordering is what makes the greedy matcher prefer a longer
motif over a shorter one that is its prefix, preventing partial matches. It returns the flattened motif array, the
per-motif start indices, the per-motif lengths, the original (pre-sort) motif indices, and a float64 array of motif
distances. The accumulator that sums those distances adopts their dtype, and a narrower one drifts off the recorded
value over a session's trials.

`_decompose_cue_sequence_into_trials` (decorated with `@njit(cache=True)`) performs the actual greedy match. Walking
the cue sequence from the front, at each position it scans the length-sorted motifs and accepts the first one whose
cues match exactly, advances the position by that motif's length, and records the motif's original index. If no motif
matches at the current position, it returns a trial count of `-1` together with the position at which it stopped. The
caller bounds the walk with `maximum_trials`, the total cue count across all sequences floor-divided by the
shortest motif's length, plus one.

When a sequence fails to decompose, `_decompose_multiple_cue_sequences_into_trials` raises `RuntimeError` with a
message naming the offending sequence and the next 20 unmatched cues, for example `Unable to decompose VR wall cue
sequence N of M into a sequence of trial distances. No trial motif matched at position P. The next 20 cues: [...]`.

Trials are accumulated with a running `cumulative_distance` that **opens at minus the corridor cue offset**
(`task_template.vr_environment.cue_offset_cm`), because the animal enters each corridor already that far into its
first cue and therefore completes a trial after traveling that much less than the trial's corridor length. Opening
the accumulator there puts every emitted distance in the traveled-distance frame that the encoder reports and that
carries the recorded corridor swap snapshots.

For every sequence except the last, when a trial would push the cumulative distance past that sequence's
`distance_breakpoint`, the trial is truncated. It is recorded at the breakpoint distance, but only when the truncated
remainder is positive; accumulation then resets to the breakpoint minus the cue offset and the walk moves on to the
next sequence. The function returns two arrays: `trial_type_sequence` (int32 trial-type indices in runtime order) and
`trial_distance_sequence` (float64 cumulative distances at the end of each trial).

---

## Trial geometry and cue offsets

`_process_trial_sequence(experiment_configuration, task_template, trial_types, trial_distances)` maps the decomposed
trial sequence onto absolute corridor positions. It resolves the per-trial geometry exactly as the decomposition step
does: the ordered trial names come from `experiment_configuration.trial_structures`, and each name is looked up in
`task_template.trial_structures` to yield its `TrialStructure`. Per-cue codes and lengths come from
`task_template.cues`, keyed by cue **name** rather than by code, and the corridor offset comes from
`task_template.vr_environment.cue_offset_cm`.

These are the only template fields this stage reads. Their full schema, including the fields this stage ignores, is
owned by `assets:task-templates`.

| Field                                           | Read for                                                        |
|-------------------------------------------------|-----------------------------------------------------------------|
| `TrialStructure.cue_sequence`                   | The ordered cue names forming the trial's motif and cue walk    |
| `TrialStructure.stimulus_trigger_zone_start_cm` | The trial-relative start of the stimulus trigger zone           |
| `TrialStructure.stimulus_trigger_zone_end_cm`   | The trial-relative end of the stimulus trigger zone             |
| `Cue.name`, `Cue.code`, `Cue.length_cm`         | The cue catalog, indexed by name into a uint8 code and a length |
| `VREnvironment.cue_offset_cm`                   | The corridor offset applied at a sequence start or restart      |

Walking the trial's cue sequence, each cue contributes its `length_cm` to a running cumulative distance, and the
cue's code together with the cumulative distance at its onset are appended to the cue/distance output. Two special
cases apply:

- **Cue offset on sequence start or restart.** For the first cue after a sequence start or restart, the effective
  distance to the next cue is `cue_length - cue_offset` rather than the full `cue_length`. The offset accounts for
  the runtime beginning to record mid-cue. The flag that applies it is reset after each trial truncation so the
  offset is reapplied at the next restart.
- **Trial truncation.** If adding the next cue's effective distance would exceed the trial's actual distance (the
  trial-end distance minus the previous trial's end distance), the trial was abruptly ended before completion: the
  current cue is recorded, the cumulative distance is snapped to the trial's actual end, and the offset flag is
  re-armed.

Trigger-zone boundaries are computed per trial against the corridor the trial was **entered** into, so the offset
flag is captured before the cue walk consumes it. A trial entered partway into its first cue is shorter than its
corridor by that offset, so every position the template declares against the corridor is reached that much earlier in
the traveled distance. The absolute start is therefore the previous trial's end distance plus
`stimulus_trigger_zone_start_cm` minus the entry offset, which is the cue offset when this trial was entered mid-cue
and zero otherwise, and the absolute end is the same sum built from `stimulus_trigger_zone_end_cm`. A trigger-zone
start is emitted only when it falls at or before the trial's end distance, and the matching end is clamped to the
trial's end distance when it would otherwise overshoot. A trial that ends before its trigger zone begins contributes
no entry, so the two trigger-zone arrays can be shorter than the trial-type array.

`_process_trial_sequence` returns five arrays: the cue codes (uint8), the per-cue cumulative distances (float64), the
trigger-zone start distances (float64), the trigger-zone end distances (float64), and the per-trial start distances
(float64).

---

## Output feather schemas

`parse_runtime` writes the three decomposition feathers (uncompressed Arrow IPC) under the names defined by
the `BehaviorDataFiles` roster. The roster member, on-disk filename, and column schema are:

| Roster member     | Filename                       | Columns                                                           |
|-------------------|--------------------------------|-------------------------------------------------------------------|
| `VR_CUE`          | `vr_cue_data.feather`          | `vr_cue` (cue id), `traveled_distance_cm` (cumulative distance)   |
| `VR_TRIGGER_ZONE` | `vr_trigger_zone_data.feather` | `trigger_zone_start_cm`, `trigger_zone_end_cm`                    |
| `TRIAL`           | `trial_data.feather`           | `trial_type_index`, `traveled_distance_cm` (trial start distance) |

The `vr_cue` column is the cue/distance pair produced by `_process_trial_sequence`; the trigger-zone columns are its
trigger start/end arrays; and `trial_data` pairs the `trial_type_index` array (the decomposed trial-type indices)
with the per-trial start distances. The roster enumeration that owns these filenames is documented by
`mesoscope:mesoscope-vr-processing-schema`; the distance-indexed interpolation that consumes these feathers into the
assembled session feather is documented by `mesoscope:mesoscope-vr-dataset-assembly`.

---

## Related skills

| Skill                                           | Relationship                                                       |
|-------------------------------------------------|--------------------------------------------------------------------|
| `mesoscope:mesoscope-vr-runtime`                | NOT this skill: the acquisition-side runtime behavior layer        |
| `forging:data-processing-design`                | Owns the agnostic processing doctrine this stage instantiates      |
| `forging:behavior-input-format`                 | Owns the runtime NPZ archive format, source ID, and payload layout |
| `assets:task-templates`                         | Owns the `TaskTemplate` schema this stage reads the geometry from  |
| `mesoscope:mesoscope-vr-experiment-schema`      | Owns the experiment configuration and its runtime trial classes    |
| `mesoscope:mesoscope-vr-processing-schema`      | Owns the `BehaviorDataFiles` roster naming the output feathers     |
| `forging:behavior-processing`                   | Owns the prepare-then-execute batch orchestration for this stage   |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | Owns the fluorescence timestamps used as the assembly reference    |
| `mesoscope:mesoscope-vr-dataset-assembly`       | Consumes these feathers via distance-indexed interpolation         |

---

## Verification checklist

```text
- [ ] Message codes match runtime.py: system=1, runtime=2, reinforcing guidance=3, aversive guidance=4,
      distance snapshot=5; cue sequences are length-identified (> 500 bytes)
- [ ] Decomposition described as greedy longest-first numba matching; failure returns trial_count == -1
      and raises RuntimeError naming the position and next 20 cues; empty / mismatched-breakpoint inputs
      raise ValueError
- [ ] Cue offset applies only to the first cue after a sequence start or restart (cue_length - cue_offset)
- [ ] Output feathers named via BehaviorDataFiles (vr_cue_data, vr_trigger_zone_data, trial_data) with the
      documented columns; written only for MESOSCOPE_EXPERIMENT sessions
- [ ] Trial motifs, motif distances, and the cue offset sourced from the task template (task_template.cues,
      task_template.trial_structures, task_template.vr_environment.cue_offset_cm), keyed by cue name
- [ ] No trial_length_cm, TrialGeometryEntry, or StimulusMode claimed anywhere; none of the three exist
- [ ] No reStructuredText specifiers (:class:/:func:/:meth:) anywhere; cross-references use plugin:skill
      syntax with the ataraxis@ prefix for ataraxis-marketplace plugins
- [ ] "feather" used only as the Arrow IPC file-format term, never as a module or skill name
- [ ] Disambiguated from mesoscope:mesoscope-vr-runtime (acquisition runtime layer, not log decomposition)
```
