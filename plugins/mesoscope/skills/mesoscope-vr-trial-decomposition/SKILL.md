---
name: mesoscope-vr-trial-decomposition
description: >-
  Documents Mesoscope-VR's concrete runtime-log decomposition: the runtime message codes, the
  numba-accelerated greedy longest-first cue-sequence-to-trial-motif matching, trial-type and
  cumulative-distance sequence extraction, trial geometry (trigger zones, cue offsets on sequence
  restart), and the trial-geometry metadata (TrialGeometryEntry, StimulusMode). Use when interpreting
  the vr_cue, vr_trigger_zone, or trial feathers, debugging cue-sequence decomposition failures, or
  modifying trial geometry. This is the forging-side runtime LOG decomposition, distinct from the
  acquisition-side runtime behavior layer in mesoscope:mesoscope-vr-runtime.
user-invocable: false
---

# Mesoscope-VR trial decomposition

Decomposes the Mesoscope-VR runtime log archive into the per-trial behavior feathers (`vr_cue_data.feather`,
`vr_trigger_zone_data.feather`, `trial_data.feather`) by matching the recorded virtual-reality wall-cue sequence
against the experiment's trial motifs.

This skill documents the **forging-side runtime LOG decomposition** implemented in
`src/sollertia_forgery/mesoscope_vr/runtime.py` (plus the two trial-geometry definitions in
`src/sollertia_forgery/mesoscope_vr/metadata.py`). It is **not** the acquisition-side runtime behavior layer (the
state machine, orchestrator, GUI, and CLI) — that is owned by `mesoscope:mesoscope-vr-runtime`. That layer is a
distinct, non-delegating counterpart: its only notion of "trial decomposition" is the acquisition-time Unity
cue-sequence decomposition, which it delegates to `experiment:vr-driver-interface`, and it never invokes
`process_runtime_data`. The two concerns share no producer/consumer relationship. See
[Related skills](#related-skills) for the disambiguation.

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
- The trial-geometry metadata `TrialGeometryEntry` and `StimulusMode` — the canonical per-trial-type geometry
  projection, grouped here as documentation companions to the trial-geometry mapping (they are produced and
  consumed downstream of this stage, not by `runtime.py`)

**Does not cover:**
- The runtime NPZ log archive format, source ID, and message payload byte layout (see
  `forging:behavior-input-format`)
- The acquisition-side runtime behavior LAYER: state machine, orchestrator, GUI, CLI (see
  `mesoscope:mesoscope-vr-runtime` — NOT this skill)
- Mesoscope-VR experiment configuration and the trial classes that supply the motifs (see
  `mesoscope:mesoscope-vr-experiment-schema`)
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

The single entry point is `process_runtime_data(log_path, output_directory, experiment_configuration=None)`. It reads
the runtime log archive with a `LogArchiveReader` (from ataraxis-data-structures), partitions every message by its
code, and writes the extracted streams as uncompressed Arrow IPC `.feather` files into `output_directory`.

The archive itself is resolved by `find_log_archive(data_directory)`, which looks for the single
`{RUNTIME_SOURCE_ID}_log.npz` file at `data_directory / 1_log.npz` (where `RUNTIME_SOURCE_ID` is the string `"1"`)
and returns `None` if the directory is missing or the archive is absent. Every processable session contains exactly
one runtime archive, always under that fixed source ID.

The system-state and runtime-state feathers are written for **every** session. The reinforcing-guidance,
aversive-guidance, and the three trial-decomposition feathers are written **only when** an
`experiment_configuration` is supplied (that is, only for experiment sessions). The guidance feathers are
additionally written only when their corresponding events were actually recorded.

---

## Runtime message codes

`process_runtime_data` classifies each message by inspecting `payload[0]` (the leading code byte), except for VR
wall-cue sequences, which are identified by payload length. A payload longer than `_CUE_SEQUENCE_MIN_LENGTH`
(500 bytes) is treated as a cue sequence and is only collected when an `experiment_configuration` is present.

| Concern                | Constant                          | Code | Extracts                                                          |
|------------------------|-----------------------------------|------|------------------------------------------------------------------|
| VR system state        | `_SYSTEM_STATE_CODE`              | `1`  | `system_state` (uint8 from `payload[1]`) and message timestamp   |
| Session runtime state  | `_RUNTIME_STATE_CODE`            | `2`  | `runtime_state` (uint8 from `payload[1]`) and message timestamp  |
| Reinforcing guidance   | `_REINFORCING_GUIDANCE_STATE_CODE`| `3`  | `reinforcing_guidance_state` (uint8) and message timestamp       |
| Aversive guidance      | `_AVERSIVE_GUIDANCE_STATE_CODE`  | `4`  | `aversive_guidance_state` (uint8) and message timestamp          |
| Distance snapshot      | `_DISTANCE_SNAPSHOT_CODE`        | `5`  | A float64 traveled distance from `payload[1:9]` (little-endian)  |
| VR wall-cue sequence   | (none — length-based)            | n/a  | The full uint8 cue array, when `len(payload) > 500`              |

Message timestamps come from `message.timestamp_us`; the `LogArchiveReader` resolves onsets and yields absolute
microsecond timestamps, so no manual onset offset is computed. Distance-snapshot messages are logged whenever the
active VR wall-cue sequence changes, and their float64 values become the breakpoints used to stitch multiple cue
sequences together (see below).

The system-state and runtime-state streams are written to the feathers named by the `SYSTEM_STATE` and
`RUNTIME_STATE` members of the `BehaviorDataFiles` roster; the guidance streams use `REINFORCING_GUIDANCE` and
`AVERSIVE_GUIDANCE`. The roster definitions themselves are owned by `mesoscope:mesoscope-vr-processing-schema`.

---

## Cue-sequence to trial-motif decomposition

`_decompose_multiple_cue_sequences_into_trials(experiment_configuration, cue_sequences, distance_breakpoints)`
converts the collected wall-cue sequences into a single ordered list of trials. It handles the case where the
original cue sequence was interrupted during runtime and a new sequence was generated, using the distance
breakpoints to stitch the sequences together.

It raises:

- `ValueError` when no cue sequences are provided.
- `ValueError` when there are multiple cue sequences but the number of distance breakpoints is not exactly the
  number of sequences minus one.

The trial motifs are the per-trial-type cue sequences read from `experiment_configuration.trial_structures`: each
trial's `cue_sequence` becomes a uint8 motif, and each trial's `trial_length_cm` becomes that motif's distance.

`_prepare_motif_data` flattens the motifs into a single contiguous array for the numba kernel and **sorts the
motifs by length, longest first**. This longest-first ordering is what makes the greedy matcher prefer a longer
motif over a shorter one that is its prefix, preventing partial matches. It returns the flattened motif array, the
per-motif start indices, the per-motif lengths, the original (pre-sort) motif indices, and a float32 array of motif
distances.

`_decompose_sequence_numba_flat` (decorated with `@njit(cache=True)`) performs the actual greedy match. Walking the
cue sequence from the front, at each position it scans the length-sorted motifs and accepts the first one whose cues
match exactly, advances the position by that motif's length, and records the motif's original index. If no motif
matches at the current position, it returns a trial count of `-1` to signal failure.

When a sequence fails to decompose, `_decompose_multiple_cue_sequences_into_trials` reconstructs the failure
position and raises `RuntimeError` with a message naming the offending sequence and the next 20 unmatched cues, for
example `Unable to decompose VR wall cue sequence N of M into a sequence of trial distances. No trial motif matched
at position P. The next 20 cues: [...]`.

For multi-sequence sessions, trials are accumulated with a running `cumulative_distance`. For every sequence except
the last, when a trial would push the cumulative distance past that sequence's `distance_breakpoint`, the trial is
truncated to the breakpoint distance and accumulation resumes from the breakpoint for the next sequence. The
function returns two arrays: `trial_type_sequence` (int32 trial-type indices in runtime order) and
`trial_distance_sequence` (float64 cumulative distances at the end of each trial).

---

## Trial geometry and cue offsets

`_process_trial_sequence(experiment_configuration, trial_types, trial_distances)` maps the decomposed trial sequence
onto absolute corridor positions. For each trial it looks up the trial type in
`experiment_configuration.trial_structures` and reads the per-cue lengths from `experiment_configuration.cues` (the
`{cue.code: cue.length_cm}` map) and the corridor offset from `experiment_configuration.cue_offset_cm`.

Walking the trial's cue sequence, each cue contributes its `length_cm` to a running cumulative distance, and the
cue's id together with the cumulative distance at its onset are appended to the cue/distance output. Two special
cases apply:

- **Cue offset on sequence start or restart.** For the first cue after a sequence start or restart, the effective
  distance to the next cue is `cue_length - cue_offset` rather than the full `cue_length`. The offset accounts for
  the runtime beginning to record mid-cue. The flag that applies it is reset after each trial truncation so the
  offset is reapplied at the next restart.
- **Trial truncation.** If adding the next cue's effective distance would exceed the trial's actual distance (the
  trial-end distance minus the previous trial's end distance), the trial was abruptly ended before completion: the
  current cue is recorded, the cumulative distance is snapped to the trial's actual end, and the offset flag is
  re-armed.

Trigger-zone boundaries are computed per trial from trial-relative positions. The absolute start is the previous
trial's end distance plus `stimulus_trigger_zone_start_cm`, and the absolute end is the previous trial's end
distance plus `stimulus_trigger_zone_end_cm`. A trigger-zone start is emitted only when it falls at or before the
trial's end distance; the end is clamped to the trial's end distance when it would otherwise overshoot.

`_process_trial_sequence` returns five arrays: the cue ids (uint8), the per-cue cumulative distances (float64), the
trigger-zone start distances (float64), the trigger-zone end distances (float64), and the per-trial start distances
(float64).

---

## Output feather schemas

`process_runtime_data` writes the three decomposition feathers (uncompressed Arrow IPC) under the names defined by
the `BehaviorDataFiles` roster. The roster member, on-disk filename, and column schema are:

| Roster member     | Filename                    | Columns                                                          |
|-------------------|-----------------------------|-----------------------------------------------------------------|
| `VR_CUE`          | `vr_cue_data.feather`       | `vr_cue` (cue id), `traveled_distance_cm` (cumulative distance)  |
| `VR_TRIGGER_ZONE` | `vr_trigger_zone_data.feather`| `trigger_zone_start_cm`, `trigger_zone_end_cm`               |
| `TRIAL`           | `trial_data.feather`        | `trial_type_index`, `traveled_distance_cm` (trial start distance)|

The `vr_cue` column is the cue/distance pair produced by `_process_trial_sequence`; the trigger-zone columns are its
trigger start/end arrays; and `trial_data` pairs the `trial_type_index` array (the decomposed trial-type indices)
with the per-trial start distances. The roster enumeration that owns these filenames is documented by
`mesoscope:mesoscope-vr-processing-schema`; the distance-indexed interpolation that consumes these feathers into the
assembled session feather is documented by `mesoscope:mesoscope-vr-dataset-assembly`.

---

## Trial-geometry metadata

Two definitions in `src/sollertia_forgery/mesoscope_vr/metadata.py` capture the canonical per-trial-type geometry
projection. They are **not** consumed by the trial-decomposition code in `runtime.py` — that code reads its spatial
fields (trigger-zone bounds, cue lengths, `cue_offset_cm`) directly off `experiment_configuration.trial_structures`,
`experiment_configuration.cues`, and `experiment_configuration.cue_offset_cm`, never off `TrialGeometryEntry`.
Instead, `TrialGeometryEntry` and `StimulusMode` are produced by `TrialGeometry.from_experiment_configuration` (also
in `metadata.py`) and consumed downstream by the dataset-forging stage. They are grouped here as a
documentation-companion to the trial-geometry mapping above — the same "trial-geometry projection" grouping the
processing-schema exemplar uses — because they describe the canonical geometry of the same trials. The cross-cutting
roster enums in the same module are owned elsewhere (see `mesoscope:mesoscope-vr-processing-schema`).

`StimulusMode` is a string enumeration of the semantic meaning of the stimulus delivered when a trial's stimulus
trigger zone fires:

| Member     | Value        | Meaning                                                                    |
|------------|--------------|----------------------------------------------------------------------------|
| `REWARD`   | `"reward"`   | An appetitive stimulus (water delivery in a `WaterRewardTrial`) on trigger |
| `AVERSIVE` | `"aversive"` | An aversive stimulus (gas puff in a `GasPuffTrial`) when the trigger fails  |

`TrialGeometryEntry` is a frozen, slotted dataclass holding the canonical geometry for a single trial type:

| Field                            | Type          | Captures                                                          |
|----------------------------------|---------------|------------------------------------------------------------------|
| `stimulus_mode`                  | `StimulusMode`| Whether the trigger delivers a reward or an aversive stimulus    |
| `trial_length_cm`                | `float`       | The canonical track length for this trial type, in centimeters   |
| `stimulus_trigger_zone_start_cm` | `float`       | Trial-relative start of the stimulus trigger zone, in centimeters|
| `stimulus_trigger_zone_end_cm`   | `float`       | Trial-relative end of the stimulus trigger zone, in centimeters  |
| `stimulus_location_cm`           | `float`       | Trial-relative location of the stimulus boundary, in centimeters |
| `cue_offset_cm`                  | `float`       | Offset between runtime trial start and the canonical first cue   |

The same `cue_offset_cm` documented here as a `TrialGeometryEntry` field is the value `_process_trial_sequence`
reads from `experiment_configuration.cue_offset_cm` and applies to the first cue after a sequence start or restart.
When it is non-zero, the runtime begins recording mid-cue, so downstream trial boundaries must be re-aligned to the
first-cue transition before the cue zones and the trigger zone read at canonical positions.

---

## Related skills

| Skill                                    | Relationship                                                                      |
|------------------------------------------|----------------------------------------------------------------------------------|
| `mesoscope:mesoscope-vr-runtime`         | NOT this skill — distinct, non-delegating acquisition-side runtime behavior layer |
| `forging:data-processing-design`         | Owns the agnostic processing doctrine this stage instantiates                     |
| `forging:behavior-input-format`          | Owns the runtime NPZ archive format, source ID, and message payload layout        |
| `mesoscope:mesoscope-vr-experiment-schema`| Owns the experiment configuration and trial classes that supply the motifs       |
| `mesoscope:mesoscope-vr-processing-schema`| Owns the `BehaviorDataFiles` filename roster used by the output feathers          |
| `forging:behavior-processing`            | Owns the prepare-then-execute batch orchestration that drives this stage          |
| `mesoscope:mesoscope-vr-fluorescence-alignment`| Owns the fluorescence frame timestamps used as the assembly reference vector |
| `mesoscope:mesoscope-vr-dataset-assembly`| Consumes these feathers via distance-indexed interpolation into `data.feather`    |

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
      documented columns; written only for experiment sessions
- [ ] TrialGeometryEntry fields and StimulusMode members (REWARD / AVERSIVE) quoted from metadata.py
- [ ] No reStructuredText specifiers (:class:/:func:/:meth:) anywhere; cross-references use plugin:skill
      syntax with the ataraxis@ prefix for ataraxis-marketplace plugins
- [ ] "feather" used only as the Arrow IPC file-format term, never as a module or skill name
- [ ] Disambiguated from mesoscope:mesoscope-vr-runtime (acquisition runtime layer, not log decomposition)
```
