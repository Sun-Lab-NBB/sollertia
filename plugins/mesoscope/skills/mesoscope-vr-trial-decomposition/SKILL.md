---
name: mesoscope-vr-trial-decomposition
description: >-
  Documents Mesoscope-VR's concrete runtime-log decomposition: the runtime message codes, the numba-accelerated greedy
  longest-first cue-sequence-to-trial-motif matching, the per-trial records it extracts, and the task-template trial
  geometry (trigger zones, cue offsets on sequence restart). Use when interpreting the vr_corridor,
  vr_cue, vr_trigger_zone, or trial feathers, or when debugging cue-sequence decomposition failures.
user-invocable: false
---

# Mesoscope-VR trial decomposition

Decomposes the Mesoscope-VR runtime log archive into the per-trial behavior feathers (`vr_corridor_data.feather`,
`vr_cue_data.feather`, `vr_trigger_zone_data.feather`, `trial_data.feather`) by matching the recorded virtual-reality
wall-cue sequence against the experiment's trial motifs.

This skill documents the **forging-side runtime LOG decomposition** implemented in
`src/sollertia_forgery/mesoscope_vr/runtime.py`. It is **not** the acquisition-side runtime behavior layer (the state
machine, orchestrator, GUI, and CLI), which is owned by `/mesoscope-vr-runtime`. That layer is a distinct,
non-delegating counterpart: its only notion of "trial decomposition" is the acquisition-time Unity cue-sequence
decomposition, which it delegates to `experiment:vr-driver-interface`, and it never invokes `parse_runtime`.

---

## Scope

**Covers:**
- The runtime message codes that partition the runtime log archive payloads (system state, runtime state,
  reinforcing guidance, aversive guidance, distance snapshot), and the acquisition-side code this parser never
  dispatches
- The `_RUNTIME_PARSER_REGISTRY` donation binding `RUNTIME_SOURCE_ID` and `parse_runtime` to
  `AcquisitionSystems.MESOSCOPE_VR`
- Decomposing one or more virtual-reality wall-cue sequences into trial motifs via numba-accelerated greedy
  longest-first matching, and stitching multiple sequences end to end at the running totals of their distance
  snapshots
- Extracting one record per trial: its type index, its start and end distances, the distance traveled inside it, and
  whether it opens a corridor or was cut short
- Mapping the decomposed trials into trial geometry: per-cue cumulative distances, trigger-zone start and end
  boundaries, and the cue offset applied to the first cue after a sequence start or restart
- The output schemas for the `vr_corridor_data`, `vr_cue_data`, `vr_trigger_zone_data`, and `trial_data` feathers
- Joining the experiment configuration's ordered trial names to the VR task template's spatial trial geometry

**Does not cover:**
- The runtime NPZ log archive format, source ID, and message payload byte layout (see `forging:processing-input-format`)
- The acquisition-side runtime behavior LAYER: state machine, orchestrator, GUI, CLI (see
  `/mesoscope-vr-runtime`, NOT this skill)
- The Mesoscope-VR experiment configuration and its `MesoscopeWaterRewardTrial` and `MesoscopeGasPuffTrial` runtime
  trial classes (see `/mesoscope-vr-experiment-schema`)
- The `TaskTemplate` schema itself: the cue catalog, the VR environment, and the trial structures (see
  `assets:task-templates`)
- The pipeline-wide `BehaviorDataFiles` and `DatasetColumn` roster enumerations (see
  `/mesoscope-vr-processing-schema`)
- Fluorescence frame timestamps used as the assembly reference vector (see
  `/mesoscope-vr-fluorescence-alignment`)
- Distance-indexed interpolation of trials into the assembled session feather (see
  `/mesoscope-vr-dataset-assembly`)
- The agnostic prepare-then-execute batch orchestration (see `forging:data-processing-design` and
  `forging:batch-processing`)

---

## Runtime-log decomposition contract

The agnostic processing doctrine, covering the prepare-then-execute batch model, worker-budget concurrency, and the
feather-as-interchange contract, is owned by `forging:data-processing-design`.

The donation fills a single registry seam. `_RUNTIME_PARSER_REGISTRY` in `sollertia_forgery/registries.py` holds the
pair `(RUNTIME_SOURCE_ID, parse_runtime)` under the `AcquisitionSystems.MESOSCOPE_VR` key. `RUNTIME_SOURCE_ID` is the
string `"1"`, and the agnostic runtime pipeline reads it to locate the session's single `1_log.npz` archive before it
dispatches the parser over that archive's decoded messages. Every processable session contains exactly one runtime
archive, always under that fixed source ID.

The single entry point is `parse_runtime(decoded_messages, output_directory, session)`. It receives the already decoded
runtime archive as a Polars DataFrame carrying a `time_us` UInt64 column and a `payload` Binary column in archive order.
It partitions every payload by its code and writes the extracted streams as uncompressed Arrow IPC `.feather` files into
`output_directory`.

Before it partitions anything, `parse_runtime` resolves two configuration documents off the loaded `SessionData`:

- `MesoscopeExperimentConfiguration`, from `raw_data/experiment_configuration.yaml`. It supplies the ordered trial
  names and the per-trial experiment parameters.
- `TaskTemplate`, from `raw_data/vr_configuration.yaml`. It supplies the spatial trial geometry: the cue catalog, the
  corridor cue offset, and each trial's cue sequence and trigger zone.

The configuration resolution is gated on `session.session_type == SessionTypes.MESOSCOPE_EXPERIMENT` and the template
resolution on that configuration resolving to a value, which is why the two coincide. Each raises `FileNotFoundError`
naming the session and the missing path when an experiment session lacks its file. Only session types listed in
`SESSION_TYPES_USING_VR_TASK` carry a `vr_configuration.yaml` at all, and that frozenset holds exactly
`MESOSCOPE_EXPERIMENT`. For every other session type both resolvers return `None`.

The system-state and runtime-state feathers are written for **every** session type. The reinforcing-guidance,
aversive-guidance, and the three trial-decomposition feathers sit behind one combined guard,
`if experiment_configuration is not None and task_template is not None:`. The task template resolves to a non-`None`
value exactly when the experiment configuration does, so the second clause never decides the branch on its own. It earns
its place by narrowing the template for the geometry join the branch performs. The guidance feathers are additionally
written only when their corresponding events were actually recorded.

---

## Runtime message codes

`parse_runtime` classifies each message by inspecting `payload[0]` (the leading code byte), except for VR wall-cue
sequences, which are identified by payload length. A payload longer than `_CUE_SEQUENCE_MIN_LENGTH` (500 bytes) is
treated as a cue sequence and is collected only for experiment sessions. The length test runs first in the dispatch
chain, and the length alone classifies the message, so a long payload is consumed by that branch whether or not the
session collects it. Testing the session type **alongside** the length would let a wall-cue sequence fall into the
state-code chain below, where its first two cue codes read as a state transition.

| Concern               | Constant                           | Code | Extracts                                                        |
|-----------------------|------------------------------------|------|-----------------------------------------------------------------|
| VR system state       | `_SYSTEM_STATE_CODE`               | `1`  | `system_state` (uint8 from `payload[1]`) and message timestamp  |
| Session runtime state | `_RUNTIME_STATE_CODE`              | `2`  | `runtime_state` (uint8 from `payload[1]`) and message timestamp |
| Reinforcing guidance  | `_REINFORCING_GUIDANCE_STATE_CODE` | `3`  | `reinforcing_guidance_state` (uint8) and message timestamp      |
| Aversive guidance     | `_AVERSIVE_GUIDANCE_STATE_CODE`    | `4`  | `aversive_guidance_state` (uint8) and message timestamp         |
| Distance snapshot     | `_DISTANCE_SNAPSHOT_CODE`          | `5`  | A float64 traveled distance from `payload[1:9]` (little-endian) |
| VR wall-cue sequence  | (none, length-based)               | n/a  | The full uint8 cue array, when `len(payload) > 500`             |

Message timestamps come from the decoded `time_us` column, which already carries absolute microsecond values because
the decoding stage resolved the archive onsets, so this parser computes no onset offset of its own.

The dispatch chain consumes message codes `1` through `5` and nothing else. The acquisition-side orchestrator emits a
sixth code, `6` (`MESOSCOPE_ACQUISITION_STATE` in the roster owned by `/mesoscope-vr-runtime`), carrying
whether the ScanImagePC is expected to be writing session frames. No branch of the chain tests for it, so a payload
carrying that code falls off the end of the chain and is discarded without a warning, and no output feather reports
it. Teaching this parser a new code means adding a branch here and a matching feather to the `BehaviorDataFiles`
roster.

A distance-snapshot message is logged from exactly one acquisition-side site: the branch that handles Unity signaling
runtime termination. That branch emergency-pauses the runtime and records the wheel encoder's traveled distance since
the current corridor's onset. Resuming afterwards re-arms Unity through the editor bridge, logs the regenerated VR
wall-cue sequence, and zeroes that distance tracker, so each snapshot measures only the corridor it ends. The
decomposition therefore closes each corridor at its own snapshot, and `_resolve_corridor_starts` sums the snapshots into
running totals that place every corridor's opening distance (see below). A termination the operator resolves by ending
the session instead of resuming leaves a trailing snapshot with no following sequence. With a single recorded sequence
that surplus snapshot is ignored, since only sequences before the last close at a snapshot. With two or more sequences,
the surplus snapshot makes the snapshot count equal the sequence count and trips the `ValueError` described below.

The system-state and runtime-state streams are written to the feathers named by the `SYSTEM_STATE` and
`RUNTIME_STATE` members of the `BehaviorDataFiles` roster, and the guidance streams use `REINFORCING_GUIDANCE` and
`AVERSIVE_GUIDANCE`. The roster definitions themselves are owned by `/mesoscope-vr-processing-schema`.

---

## Cue-sequence to trial-motif decomposition

`_decompose_multiple_cue_sequences_into_trials` converts the collected wall-cue sequences into a single ordered list
of trials, taking the experiment configuration, the task template, the cue sequences, and the distance snapshots as
`corridor_distances`. It handles the case where the original cue sequence was interrupted during runtime and a new
sequence was generated, closing each interrupted corridor at its snapshot.

It raises:

- `ValueError` when no cue sequences are provided.
- `ValueError` when there are multiple cue sequences but the number of distance snapshots is not exactly the
  number of sequences minus one.
- `ValueError` when the experiment configuration names a trial the task template does not define. The message lists
  the missing names alongside the template's available trial names.

**The two documents are joined by trial name.** The experiment configuration owns the ordered trial names, read as
`list(experiment_configuration.trial_structures.keys())`, and every `trial_type_index` this stage emits refers to that
order. The task template owns the spatial data: `_resolve_trial_geometries` looks each name up in
`task_template.trial_structures` and returns the matching `TrialStructure` values in configuration order. The split is
deliberate. The template carries only the spatial data Unity needs to build and run the corridor, while the
experiment-specific parameters live on the runtime trial classes `MesoscopeWaterRewardTrial` and
`MesoscopeGasPuffTrial`, whose schema is owned by `/mesoscope-vr-experiment-schema`. The `TaskTemplate` schema
is owned by `assets:task-templates`.

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

Each corridor's trials are placed in that corridor's own frame, the distance traveled since its onset, which is the
frame the snapshot records. The animal enters each corridor already `task_template.vr_environment.cue_offset_cm` into
its first cue, so the corridor's first trial spans its motif distance minus that offset, and every later trial spans
its full motif distance. In every sequence except the last, the first trial whose end passes the corridor's snapshot is
truncated at the snapshot, and the rest of the sequence is abandoned. The truncated trial is recorded only when the
snapshot lies past its start, so a swap landing on a trial boundary contributes no zero-length trial. Each decision
compares sums of template lengths with the measured snapshot, so none depends on the rounding of cumulative distances.

The function returns one `_Trial` record per trial, in runtime order:

| Field                | Holds                                                                                    |
|----------------------|------------------------------------------------------------------------------------------|
| `trial_type`         | The trial-type index, in the experiment configuration's order                            |
| `start_cm`, `end_cm` | The cumulative distances at the trial's start and end, offset by the corridor's opening  |
| `length_cm`          | The distance traveled inside the trial, measured in the corridor's own frame             |
| `opens_corridor`     | Whether the trial is its corridor's first, entered partway into its first cue            |
| `truncated`          | Whether a corridor swap cut the trial short                                              |

---

## Trial geometry and cue offsets

`_process_trial_sequence(experiment_configuration, task_template, trials)` maps the decomposed trials onto cue and
trigger-zone distances. It resolves the per-trial geometry exactly as the decomposition step does: the ordered trial
names come from `experiment_configuration.trial_structures`, and each name is looked up in
`task_template.trial_structures` to yield its `TrialStructure`. Per-cue codes and lengths come from
`task_template.cues`, keyed by cue **name**, and the corridor offset comes from
`task_template.vr_environment.cue_offset_cm`.

These are the only template fields this stage reads. Their full schema, including the fields this stage ignores, is
owned by `assets:task-templates`.

| Field                                           | Read for                                                        |
|-------------------------------------------------|-----------------------------------------------------------------|
| `TrialStructure.cue_sequence`                   | The ordered cue names forming the trial's motif and cue walk    |
| `TrialStructure.stimulus_trigger_zone_start_cm` | The trial-relative start of the stimulus trigger zone           |
| `TrialStructure.stimulus_trigger_zone_end_cm`   | The trial-relative end of the stimulus trigger zone             |
| `Cue.name`, `Cue.code`, `Cue.length_cm`         | The cue catalog, indexed by name into a uint8 code and a length |
| `VREnvironment.cue_offset_cm`                   | The offset applied to the first trial of every corridor         |

Walking the trial's cue sequence, the stage appends each cue's code and its onset, the trial's start distance plus
the distance traveled inside the trial before that cue. Every decision compares a position inside the trial with the
trial's `length_cm`, never two cumulative distances. Two special cases apply:

- **Cue offset at a corridor's onset.** In a trial that opens a corridor, the first cue spans `cue_length - cue_offset`.
  The offset is the animal's starting position relative to the cue sequence origin, so it enters every corridor
  already that far into its first cue.
- **Trial truncation.** A truncated trial lists only the cues whose onset inside the trial falls below its
  `length_cm`, which are the cues the animal entered before the swap.

Trigger-zone boundaries are computed per trial. A trial that opens a corridor is shorter than its corridor by the cue
offset, so every position the template declares against the corridor is reached that much earlier in the traveled
distance. Inside the trial, the zone starts at `stimulus_trigger_zone_start_cm` minus that entry offset and the actor
trigger lead, and ends at `stimulus_trigger_zone_end_cm` minus the entry offset plus the same lead. The lead,
`_ACTOR_TRIGGER_LEAD_CM` (4.0 cm), is the distance by which Unity registers a zone crossing ahead of the tracked
position, because the Gimbl actor is a 5 cm collider sphere whose surface meets the zone before and after its center
does. A trigger-zone start is emitted, offset by the trial's start distance, only when it falls at or before the
trial's `length_cm`, and the matching end is clamped to the trial's end distance when it would otherwise overshoot. A
trial that ends before its trigger zone begins contributes no entry, so the two trigger-zone arrays can be shorter than
the trial list.

`_process_trial_sequence` returns four arrays: the cue codes (uint8), the per-cue cumulative distances (float64), the
trigger-zone start distances (float64), and the trigger-zone end distances (float64).

---

## Output feather schemas

`parse_runtime` writes the four decomposition feathers (uncompressed Arrow IPC) under the names defined by the
`BehaviorDataFiles` roster. The roster member, on-disk filename, and column schema are:

| Roster member     | Filename                       | Columns                                                             |
|-------------------|--------------------------------|---------------------------------------------------------------------|
| `VR_CORRIDOR`     | `vr_corridor_data.feather`     | `time_us` (corridor onset), `traveled_distance_cm` (corridor start) |
| `VR_CUE`          | `vr_cue_data.feather`          | `vr_cue` (cue id), `traveled_distance_cm` (cumulative distance)     |
| `VR_TRIGGER_ZONE` | `vr_trigger_zone_data.feather` | `trigger_zone_start_cm`, `trigger_zone_end_cm`                      |
| `TRIAL`           | `trial_data.feather`           | `trial_type_index`, `traveled_distance_cm` (trial start distance)   |

Every distance column holds the corridor frame, in which each corridor starts where the one before it ended.
`vr_corridor_data` carries one row per cue sequence. Its `time_us` is the sequence's log timestamp, which marks the
corridor onset because the acquisition runtime logs each sequence immediately before zeroing its distance tracker. Its
`traveled_distance_cm` is the running total of the snapshots logged before it, zero for the first corridor.

The `vr_cue` column is the cue/distance pair produced by `_process_trial_sequence`, the trigger-zone columns are its
trigger start/end arrays, and `trial_data` pairs each decomposed trial's type index with its start distance. The roster
enumeration that owns these filenames is documented by `/mesoscope-vr-processing-schema`. The distance-indexed
interpolation that consumes these feathers into the assembled session feather is documented by
`/mesoscope-vr-dataset-assembly`.

---

## Related skills

| Skill                                  | Relationship                                                      |
|----------------------------------------|-------------------------------------------------------------------|
| `/mesoscope-vr-runtime`                | NOT this skill: the acquisition-side runtime behavior layer       |
| `forging:data-processing-design`       | Owns the agnostic processing doctrine this stage instantiates     |
| `forging:processing-input-format`      | Owns the agnostic runtime archive discovery and decode contract   |
| `assets:task-templates`                | Owns the `TaskTemplate` schema this stage reads the geometry from |
| `/mesoscope-vr-experiment-schema`      | Owns the experiment configuration and its runtime trial classes   |
| `/mesoscope-vr-processing-schema`      | Owns the `BehaviorDataFiles` roster naming the output feathers    |
| `forging:batch-processing`             | Owns the prepare-then-execute batch orchestration for this stage  |
| `/mesoscope-vr-fluorescence-alignment` | Owns the fluorescence timestamps used as the assembly reference   |
| `/mesoscope-vr-dataset-assembly`       | Consumes these feathers via distance-indexed interpolation        |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing
- [ ] rg -n ':class:|:func:|:meth:' <file> finds nothing
- [ ] rg -n 'trial_length_cm|TrialGeometryEntry|StimulusMode' <file> finds nothing, because none of the three exist

Runtime message dispatch:
- [ ] Message codes match runtime.py: system=1, runtime=2, reinforcing guidance=3, aversive guidance=4,
      distance snapshot=5, and cue sequences are length-identified (> 500 bytes)
- [ ] The length test stated as running first, with the reason testing the session type alongside the
      length would misroute a wall-cue sequence into the state-code chain
- [ ] Code 6 stated as emitted by the acquisition runtime and left undispatched by parse_runtime
- [ ] Distance snapshot stated as logged only when Unity signals runtime termination and as counting from the
      current corridor's onset, with each corridor closed at its own snapshot and the running totals placing every
      corridor's opening distance
- [ ] The registry seam named as _RUNTIME_PARSER_REGISTRY, holding (RUNTIME_SOURCE_ID, parse_runtime)

Decomposition and outputs:
- [ ] Decomposition described as greedy longest-first numba matching, failure returns trial_count == -1
      and raises RuntimeError naming the position and next 20 cues, empty or mismatched-snapshot inputs
      raise ValueError
- [ ] Cue offset applies only to the first cue of a trial that opens a corridor (cue_length - cue_offset)
- [ ] Truncation and trigger-zone decisions stated as taken inside the corridor or the trial, against the snapshot or
      the trial's length_cm
- [ ] Output feathers named via BehaviorDataFiles (vr_corridor_data, vr_cue_data, vr_trigger_zone_data,
      trial_data) with the documented columns
- [ ] The experiment-only outputs gated by the combined experiment_configuration and task_template guard
- [ ] Trial motifs, motif distances, and the cue offset sourced from the task template (task_template.cues,
      task_template.trial_structures, task_template.vr_environment.cue_offset_cm), keyed by cue name
- [ ] "feather" used only as the Arrow IPC file-format term, never as a module or skill name
- [ ] Disambiguated from /mesoscope-vr-runtime (acquisition runtime layer, not log decomposition)
```
