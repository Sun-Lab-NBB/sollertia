---
name: dataset-forging-input-format
description: >-
  Documents the forging-pipeline inputs: session eligibility, dataset hierarchy,
  behavior-processing tracker and feathers, cindra single- and multi-recording outputs, and
  required hardware-state / experiment-configuration YAMLs. Use when the user asks why a
  forging job failed, how datasets map to directories, or which upstream pipelines must
  complete first.
user-invocable: false
---

# Dataset forging input format

Authoritative reference for the **forging-pipeline-specific** input artifacts: the
behavior-processing feathers (upstream `forging:behavior-processing`), the cindra
single-recording outputs (upstream `cindra@cindra:single-recording-processing`), and the
cindra multi-day outputs (upstream `cindra@cindra:multi-recording-processing`). Covers
session eligibility, the on-disk dataset hierarchy, the required raw-data YAMLs, and
the cross-library handoff contract. Delegates session layout, hardware state, and
experiment configuration authoring to the assets plugin.

---

## Scope

**Covers:**
- Session eligibility: only `MESOSCOPE_EXPERIMENT` sessions are forgeable
- The self-contained forged dataset hierarchy (dataset directory, `dataset.yaml`,
  `forging_tracker.yaml`, per-animal `surgery_metadata.yaml`, per-session output subdirectories)
- The per-session forged outputs (`data.feather`, the copied `session_descriptor.yaml`, and the
  `trial_geometry.yaml` data file)
- Behavior feather inputs (which files, which columns) read from the canonical
  `processed_data/behavior_data/` directory produced by `forging:behavior-processing`
- Cindra single-recording outputs (fluorescence arrays, classification, metadata) read from the
  canonical `processed_data/cindra/` directory
- Cindra multi-day outputs read from `processed_data/cindra/multi_recording/{animal}_{dataset_name}/`
- `hardware_state.yaml` fields required at assembly time
- `experiment_configuration.yaml` fields required at assembly time
- `session_descriptor.yaml` presence required at assembly time (copied to output)
- `surgery_metadata.yaml` presence required per animal at dataset creation time (copied
  once to the dataset hierarchy)
- Cross-library handoff ordering

**Does not cover:**
- Session directory layout and `raw_data/` / `processed_data/` hierarchy (see
  `assets:session-discovery` and related skills)
- Hardware state authoring and validation (see the assets plugin)
- Experiment configuration authoring and validation (see the assets plugin)
- Batch orchestration workflow (see `forging:dataset-forging`)
- Output schemas and interpretation (see `forging:dataset-forging-results`)
- How the consumed feathers and arrays are interpolated and aligned into the forged columns (see
  `mesoscope:mesoscope-vr-dataset-assembly` and `mesoscope:mesoscope-vr-fluorescence-alignment`)
- Upstream behavior processing workflow (see `forging:behavior-processing` /
  `forging:behavior-input-format` / `forging:behavior-results`)
- Upstream cindra processing workflow (see `cindra@cindra:single-recording-processing` and
  `cindra@cindra:multi-recording-processing` and their results skills)

**Note:** `cindra@cindra:*` refers to the **cindra** plugin from the
[cindra marketplace](https://github.com/Sun-Lab-NBB/cindra).

---

## Session eligibility

The forging pipeline only accepts sessions whose `session_type` is
`MESOSCOPE_EXPERIMENT`. The gate is enforced at dataset creation time: the first
resolved session's type is inspected, and a `ValueError` is raised if it is anything
else. The user-facing error message is
`"Unable to define dataset '{name}'. Dataset creation for this acquisition system is
supported only for '{required_session_type}' sessions, but the first session's type
resolved to '{session_type}'."`.

Every subsequent session in the batch must additionally share the first session's
`session_type` and `acquisition_system`. A mismatch raises a `ValueError` at dataset
creation time. Split the batch by acquisition system and session type before
calling `prepare_forging_batch_tool`.

| Session type           | Eligible for forging | Notes                                  |
|------------------------|----------------------|----------------------------------------|
| `MESOSCOPE_EXPERIMENT` | yes                  | The only currently supported type      |
| `LICK_TRAINING`        | no                   | Refused at dataset creation            |
| `RUN_TRAINING`         | no                   | Refused at dataset creation            |
| `WINDOW_CHECKING`      | no                   | Refused at dataset creation            |
| Anything else          | no                   | Refused at dataset creation            |

Use `assets:session-discovery` to enumerate sessions, then filter client-side to
`session_type == "mesoscope experiment"` (the `SessionTypes` value for
`MESOSCOPE_EXPERIMENT`) before passing names to `prepare_forging_batch_tool`.

Eligibility is re-validated on every prepare call that hits a fresh dataset (existing
datasets are assumed already validated). A batch that mixes eligible and ineligible
sessions cannot be forged as a single dataset — split it first.

---

## Project and dataset hierarchy

The forging pipeline reads from the canonical sollertia project layout and writes a fully
self-contained dataset hierarchy under the same `project_root`. The source sessions are
read-only; no forged artifact is written back into them.

```text
{project_root}/
├── {animal_name_A}/                          ← SOURCE (read-only inputs)
│   ├── {session_name_1}/
│   │   ├── raw_data/
│   │   │   ├── session_data.yaml             ← session marker (see assets:session-discovery)
│   │   │   ├── hardware_state.yaml           ← required here
│   │   │   ├── experiment_configuration.yaml ← required here
│   │   │   ├── session_descriptor.yaml       ← required here (copied to output)
│   │   │   ├── surgery_metadata.yaml         ← required on each animal's latest session
│   │   │   └── mesoscope_data/
│   │   │       └── frame_variant_metadata.npz ← ScanImage fallback alignment input
│   │   └── processed_data/
│   │       ├── behavior_data/                ← canonical behavior feather directory
│   │       └── cindra/                       ← canonical cindra single-recording directory
│   │           └── multi_recording/
│   │               └── {animal}_{dataset_name}/  ← canonical cindra multi-day directory
│   └── {session_name_2}/...
└── {dataset_name}/                           ← FORGED DATASET HIERARCHY (this pipeline)
    ├── dataset.yaml                          ← dataset marker
    ├── forging_tracker.yaml                  ← processing tracker
    └── {animal_name_A}/
        ├── surgery_metadata.yaml             ← FORGED COPY (one per animal)
        └── {session_name_1}/
            ├── data.feather                  ← FORGED OUTPUT
            ├── session_descriptor.yaml       ← FORGED COPY
            └── trial_geometry.yaml           ← FORGED OUTPUT (canonical trial geometry)
```

Key facts:

- **Dataset directory name:** the dataset name passed to `prepare_forging_batch_tool`
  becomes the directory name directly under `project_root`.
- **Session resolution:** each session name is resolved by discovering `session_data.yaml`
  markers under `project_root` (shared-assets session discovery) and requiring exactly one
  match. Zero matches raises `FileNotFoundError`; multiple matches (a name colliding across
  animals) raises `RuntimeError`.
- **Animal assignment:** the owning animal is derived from the source session directory's
  parent name.
- **Output location:** forged per-session data is self-contained inside the dataset
  hierarchy at `{project_root}/{dataset_name}/{animal}/{session}/`, NOT next to the source
  session's `raw_data/` and `processed_data/`.
- **Cleanup semantics:** `clean_forging_output_tool` deletes the entire dataset directory
  tree — the tracker, `dataset.yaml`, per-animal `surgery_metadata.yaml` copies, AND every
  per-session `data.feather`, `session_descriptor.yaml`, and `trial_geometry.yaml`. Because
  the forged outputs live inside the dataset hierarchy, cleaning a dataset removes them all;
  the read-only source sessions are never touched.

---

## Upstream prerequisite: behavior-processing output

The forging pipeline resolves the behavior data directory to the canonical
`{processed_data}/behavior_data/` location via the loaded session's path-resolution
properties — it does not search for the tracker at arbitrary depth. The directory must
exist; a missing directory raises `FileNotFoundError` naming the expected
`behavior_processing_tracker.yaml` it is expected to contain.

The pipeline then reads the following feathers from the behavior data directory. All
are uncompressed Arrow IPC (`.feather`) produced by `forging:behavior-processing`:

### Always required

| Feather file                   | Columns consumed                                     | Use                                                |
|--------------------------------|------------------------------------------------------|----------------------------------------------------|
| `mesoscope_frame_data.feather` | `time_us`, `ttl_state`                               | TTL-to-frame alignment (cindra assembly)           |
| `system_state_data.feather`    | `time_us`, `system_state`                            | System state interpolation onto reference time     |
| `lick_data.feather`            | `time_us`, `lick_state`                              | Lick state interpolation                           |
| `valve_data.feather`           | `time_us`, `dispensed_water_volume_uL`, `tone_state` | Water delivery + reward classification             |
| `encoder_data.feather`         | `time_us`, `traveled_distance_cm`                    | Running speed, distance, reference distance        |
| `vr_trigger_zone_data.feather` | `trigger_zone_start_cm`, `trigger_zone_end_cm`       | Trigger-zone membership for the reference distance |
| `vr_cue_data.feather`          | `traveled_distance_cm`, `vr_cue`                     | Per-distance VR cue                                |
| `trial_data.feather`           | `traveled_distance_cm`, `trial_type_index`           | Trial number and trial type                        |
| `runtime_state_data.feather`   | `time_us`, `runtime_state`                           | Runtime state interpolation                        |

Only `MESOSCOPE_EXPERIMENT` sessions produce the trigger-zone, cue, trial, and runtime
state feathers, which is why lick-training and run-training sessions are not forgeable
(even though they share most of the behavior-data feathers).

### Conditional

| Feather file                                  | Produced when                                             | Resulting forged column        |
|-----------------------------------------------|-----------------------------------------------------------|--------------------------------|
| `screen_data.feather`                         | `screens_initially_on` populated (mesoscope only)         | `screens`                      |
| `brake_data.feather`                          | `maximum_brake_strength` / `minimum_brake_strength` set   | `brake`                        |
| `torque_data.feather`                         | `torque_per_adc_unit` set                                 | `torque_N_cm`                  |
| `reinforcing_guidance_state_data.feather`     | Reinforcing guidance toggled during the session           | `reinforcing_guided`           |
| `aversive_guidance_state_data.feather`        | Aversive guidance toggled during the session              | `aversive_guided`              |

See `forging:behavior-input-format` for upstream module eligibility rules and
`forging:behavior-results` for the column-level schema of each feather. See
`mesoscope:mesoscope-vr-dataset-assembly` for how these feathers are interpolated onto the
fluorescence frame reference vector to build the assembled session columns.

---

## Upstream prerequisite: cindra single-recording output

The forging pipeline resolves the cindra single-recording directory to the canonical
`{processed_data}/cindra/` location via the loaded session's path-resolution properties —
it does not search for the tracker at arbitrary depth. The directory must exist; a missing
directory raises `FileNotFoundError` naming the expected `single_recording_tracker.yaml` it
is expected to contain.

The following files are read from the cindra directory at assembly time. All are
produced by `cindra@cindra:single-recording-processing`:

| File                          | Loaded via               | Use                                                                      |
|-------------------------------|--------------------------|--------------------------------------------------------------------------|
| `cell_fluorescence.npy`       | `np.load(mmap_mode="r")` | Frame count (via shape); single-day cell fluorescence (filtered by mask) |
| `combined_metadata.npz`       | `np.load`                | `sampling_rate[0]` → scanning frequency (for TTL pulse duration window)  |
| `cell_classification.npy`     | `np.load(mmap_mode="r")` | Boolean mask (column 0 == 1) to filter non-cell ROIs                     |
| `neuropil_fluorescence.npy`   | `np.load(mmap_mode="r")` | Single-day neuropil fluorescence (filtered)                              |
| `subtracted_fluorescence.npy` | `np.load(mmap_mode="r")` | Single-day subtracted fluorescence (filtered)                            |
| `spikes.npy`                  | `np.load(mmap_mode="r")` | Single-day spike inference (filtered)                                    |

The frame count comes from the column dimension of `cell_fluorescence.npy` (its shape
is `(rois, frames)`). The primary alignment path keeps only the in-window TTL pulses whose
duration matches the scanning frequency. If the log yields more in-window pulses than cindra
frames, the pipeline keeps the **last** `frames` pulses (aberrant frames are assumed to come
from pre-experiment triggering); if it yields fewer, the pipeline falls back to matching
logged TTL rising edges against the ScanImage per-frame timestamps in
`raw_data/mesoscope_data/frame_variant_metadata.npz`. The fallback raises `ValueError` when
that archive is missing, its frame count disagrees with cindra, or the match cannot recover
exactly `frames` pulses.

The ROI axis is filtered once by the classification mask before the
array is transposed to `(frames, rois)` and stored as a Polars `Array(Float32)`
column. See `mesoscope:mesoscope-vr-fluorescence-alignment` for the full alignment logic,
tolerances, and anchor-search behavior.

---

## Upstream prerequisite: cindra multi-day output

The forging pipeline derives the multi-day directory as
`{processed_data}/cindra/multi_recording/{animal}_{dataset_name}/`. The directory name is
the owning animal identifier joined to the dataset name with an underscore, matching cindra's
on-disk qualification convention for collision-free multi-animal batches. The dataset name
passed to `prepare_forging_batch_tool` MUST match the dataset name under which the multi-day
cindra pipeline was run — the path is joined directly without fallback discovery.

The following files are read from the multi-day directory. All are produced by
`cindra@cindra:multi-recording-processing`:

| File                          | Use                                                                      |
|-------------------------------|--------------------------------------------------------------------------|
| `cell_fluorescence.npy`       | Multi-day tracked cell fluorescence (already cell-filtered)              |
| `neuropil_fluorescence.npy`   | Multi-day tracked neuropil                                               |
| `subtracted_fluorescence.npy` | Multi-day tracked subtracted fluorescence                                |
| `spikes.npy`                  | Multi-day tracked spike inference                                        |

No classification mask is applied here — the multi-day pipeline already emits
cell-filtered arrays. The arrays are transposed to `(frames, rois)` and stored as
Polars `Array(Float32)` columns.

**Common failure mode:** The multi-day pipeline was run under a different dataset
name than the one passed to `prepare_forging_batch_tool`. The
`{processed_data}/cindra/multi_recording/{animal}_{dataset_name}/` join then points at a
non-existent directory, and the assembly raises `FileNotFoundError` on
`cell_fluorescence.npy`. Fix by rerunning `cindra@cindra:multi-recording-processing` with the
matching dataset name, or by pointing the forging run at the name that was used upstream.

---

## Raw-data prerequisite: hardware state YAML

`raw_data/hardware_state.yaml` is loaded via `MesoscopeHardwareState.from_yaml` at
behavior-dataset assembly time. The forging pipeline consults the following fields:

| Field                    | Use                                                                     |
|--------------------------|-------------------------------------------------------------------------|
| `system_state_codes`     | Int → name mapping used to convert raw codes to the `system_state` enum |
| `minimum_brake_strength` | Threshold for deriving the binary `brake` column from brake torque      |

Missing fields raise `ValueError` at assembly time. The YAML itself is authored and
validated via the assets plugin — this skill only documents which fields the
forging pipeline consumes.

---

## Raw-data prerequisite: experiment configuration YAML

`raw_data/experiment_configuration.yaml` is loaded via
`MesoscopeExperimentConfiguration.from_yaml` at runtime-dataset assembly time. The
forging pipeline consults the following structures:

| Structure                                           | Use                                                                     |
|-----------------------------------------------------|-------------------------------------------------------------------------|
| `trial_structures` keys                             | Enum categories for the forged `trial_type` column (plus `"undefined"`) |
| `experiment_states` values' `experiment_state_code` | Int → name mapping used to build the `runtime_state` enum               |

The runtime state mapping also hardcodes `0 → "idle"` as the default system state.

Missing or malformed files raise at load time. Authoring and validation belong to the
assets plugin.

---

## Raw-data prerequisite: experiment descriptor YAML

`raw_data/session_descriptor.yaml` is required per session and is copied alongside
`data.feather` at the end of each session assembly (the copy is placed in the forged
session subdirectory under the dataset hierarchy, next to `data.feather` and
`trial_geometry.yaml`). The forging pipeline only checks that the file exists and copies
it — it never parses the file at assembly time. The descriptor carries experimenter-authored
runtime context (`experimenter`, `animal_weight_g`, dispensed / consumed water volumes, the
`incomplete` completion flag, and `experimenter_notes`); `MesoscopeExperimentDescriptor` is
the schema that the separate verification tool (`verify_forging_output_tool`) and downstream
analysis parse the copied file as, not a schema applied during assembly.

Presence is verified up front inside `_assemble_session_dataset`, before any cindra or
behavior work is performed: a missing file raises `FileNotFoundError` with
`"Unable to assemble session '{name}'. The session's raw data directory does not
contain a 'session_descriptor.yaml' file at '{path}'. The experiment descriptor is
required for every session in a forged dataset."`. Content is not validated by the
forging pipeline itself, so authoring and validation belong to the assets plugin.

---

## Raw-data prerequisite: per-animal surgery data YAML

`raw_data/surgery_metadata.yaml` is required for every animal represented in a dataset,
but only on the animal's **most recent** (natural-sorted last) session. Dataset
creation groups resolved session paths by owning animal and, for each animal, copies
that animal's latest session's `surgery_metadata.yaml` into
`{project_root}/{dataset_name}/{animal}/surgery_metadata.yaml`. Surgery metadata is
per-animal rather than per-session, so a single copy is materialized for each animal
in the dataset.

Missing files raise `FileNotFoundError` during dataset creation with
`"Unable to define dataset '{name}'. The latest session '{session}' for animal
'{animal}' does not contain a 'surgery_metadata.yaml' file at '{path}'. Surgery metadata
is required for every animal in a forged dataset."`. Older animal sessions are
allowed to omit the file; only the newest session per animal is consulted.

---

## Cross-library handoff contract

The forging pipeline is a pure consumer of upstream outputs:

| Upstream producer                | Required skill                              | Artifact location (under the session)                  | Gates forging |
|----------------------------------|---------------------------------------------|--------------------------------------------------------|---------------|
| Behavior processing pipeline     | `forging:behavior-processing`               | `behavior_data/` tracker + feathers                    | yes           |
| Cindra single-recording pipeline | `cindra@cindra:single-recording-processing` | `cindra/` tracker + `*.npy` / `*.npz`                  | yes           |
| Cindra multi-recording pipeline  | `cindra@cindra:multi-recording-processing`  | `cindra/multi_recording/{animal}_{dataset_name}/*.npy` | yes           |
| Mesoscope-VR acquisition runtime | (acquisition-side; no skill)                | the four `raw_data/` YAMLs (see prerequisites above)   | yes           |

**Ordering constraint:** all four upstream producers MUST complete for every session
in a dataset BEFORE `forging:dataset-forging` can assemble. Running the forging pipeline
against a partially-processed session raises during dataset assembly — the
`prepare_forging_batch_tool` call itself will mostly succeed (it resolves the dataset
hierarchy and initializes the tracker), but dataset creation will fail up front if any
animal's latest session is missing `surgery_metadata.yaml`, and per-session jobs will
fail during execution with a `FileNotFoundError` on a missing canonical `behavior_data/`
or `cindra/` directory, a missing `.npy`, or a missing `session_descriptor.yaml`.

---

## Prerequisites checklist

```text
Dataset Forging Prerequisites:
- [ ] Every session is MESOSCOPE_EXPERIMENT
- [ ] Every session shares the same acquisition_system
- [ ] raw_data/hardware_state.yaml valid per assets plugin
-   [ ] system_state_codes populated
-   [ ] minimum_brake_strength set (if brake module present)
- [ ] raw_data/experiment_configuration.yaml valid per assets plugin
-   [ ] trial_structures defined
-   [ ] experiment_states defined with experiment_state_code values
- [ ] raw_data/session_descriptor.yaml present on every session
- [ ] raw_data/surgery_metadata.yaml present on each animal's latest (natural-sorted) session
- [ ] forging:behavior-processing completed — behavior_processing_tracker.yaml + feathers present
-   [ ] mesoscope_frame_data.feather
-   [ ] system_state_data.feather, lick_data.feather, valve_data.feather
-   [ ] encoder_data.feather
-   [ ] vr_trigger_zone_data.feather, vr_cue_data.feather, trial_data.feather, runtime_state_data.feather
- [ ] cindra@cindra:single-recording-processing completed — single_recording_tracker.yaml + outputs present
-   [ ] cell_fluorescence.npy, combined_metadata.npz, cell_classification.npy
-   [ ] neuropil_fluorescence.npy, subtracted_fluorescence.npy, spikes.npy
- [ ] cindra@cindra:multi-recording-processing completed with matching dataset name
-   [ ] cindra/multi_recording/{animal}_{dataset_name}/cell_fluorescence.npy and companions present
```

---

## Related skills

| Skill                                           | Relationship                                                          |
|-------------------------------------------------|----------------------------------------------------------------------|
| `forging:forging-mcp-environment-setup`                | Prerequisite: MCP server connectivity                                |
| `assets:session-discovery`                      | Upstream: session discovery and filtering                            |
| `forging:dataset-forging`                              | Downstream: consumes the inputs documented here                      |
| `forging:dataset-forging-results`                      | Downstream: documents the output derived from these inputs           |
| `forging:data-processing-design`                       | Reference: the cross_system versus per-system forging design pattern |
| `forging:microcontroller-primitives`                   | Reference: agnostic microcontroller module-feather parsing helpers   |
| `forging:camera-timestamp-extraction`                  | Reference: agnostic camera timestamp extraction stage                |
| `forging:behavior-processing`                          | Upstream producer of behavior feathers                               |
| `forging:behavior-input-format`                        | Reference: upstream-of-upstream input format for behavior feathers   |
| `forging:behavior-results`                             | Reference: schema of the behavior feathers consumed here             |
| `mesoscope:mesoscope-vr-module-parsing`         | Reference: Mesoscope-VR per-module feather outputs feeding behavior  |
| `mesoscope:mesoscope-vr-dataset-assembly`       | Reference: how consumed feathers become forged session columns       |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | Reference: TTL and ScanImage fluorescence frame alignment logic      |
| `cindra@cindra:single-recording-processing`     | Upstream producer of cindra single-recording outputs                 |
| `cindra@cindra:multi-recording-processing`      | Upstream producer of cindra multi-day outputs                        |
| `cindra@cindra:single-recording-results`        | Reference: schemas of the cindra single-recording outputs            |
| `cindra@cindra:multi-recording-results`         | Reference: schemas of the cindra multi-day outputs                   |
