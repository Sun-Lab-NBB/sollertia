---
name: dataset-forging-input-format
description: >-
  Documents the forging-pipeline-specific inputs: session eligibility, dataset
  hierarchy, the behavior-processing tracker and feather files consumed during
  assembly, the cindra single-recording outputs, the cindra multi-day outputs, and the
  required hardware state / experiment configuration YAMLs. Use when the user asks why
  a forging job failed during prepare or execution, how datasets map to project
  directories, or which upstream pipelines must complete before forging.
user-invocable: true
---

# Dataset forging input format

Authoritative reference for the **forging-pipeline-specific** input artifacts: the
behavior-processing feathers (upstream `/behavior-processing`), the cindra
single-recording outputs (upstream `/cindra:single-recording-processing`), and the
cindra multi-day outputs (upstream `/cindra:multi-recording-processing`). Covers
session eligibility, the on-disk dataset hierarchy, the required raw-data YAMLs, and
the cross-library handoff contract. Delegates session layout, hardware state, and
experiment configuration authoring to the assets plugin.

---

## Scope

**Covers:**
- Session eligibility: only `MESOSCOPE_EXPERIMENT` sessions are forgeable
- The forged dataset hierarchy (dataset directory, `dataset.yaml`,
  `forging_tracker.yaml`, per-animal `surgery_metadata.yaml`)
- The per-session `data.feather` output path and the copied
  `session_descriptor.yaml`
- Behavior feather inputs (which files, which columns) read from the
  `/behavior-processing` output directory
- Cindra single-recording outputs (fluorescence arrays, classification, metadata)
  discovered via `single_recording_tracker.yaml`
- Cindra multi-day outputs discovered at `{cindra_parent}/multiday/{dataset_name}/`
- `hardware_state.yaml` fields required at assembly time
- `experiment_configuration.yaml` fields required at assembly time
- `session_descriptor.yaml` presence required at assembly time (copied to output)
- `surgery_metadata.yaml` presence required per animal at dataset creation time (copied
  once to the dataset hierarchy)
- Cross-library handoff ordering

**Does not cover:**
- Session directory layout and `raw_data/` / `processed_data/` hierarchy (see
  `/session-discovery` and the assets plugin)
- Hardware state authoring and validation (see the assets plugin)
- Experiment configuration authoring and validation (see the assets plugin)
- Batch orchestration workflow (see `/dataset-forging`)
- Output schemas and interpretation (see `/dataset-forging-results`)
- Upstream behavior processing workflow (see `/behavior-processing` /
  `/behavior-input-format` / `/behavior-results`)
- Upstream cindra processing workflow (see `/cindra:single-recording-processing` and
  `/cindra:multi-recording-processing` and their results skills)

**Note:** `/cindra:*` refers to the **cindra** plugin from the
[cindra marketplace](https://github.com/Sun-Lab-NBB/cindra).

---

## Session eligibility

The forging pipeline only accepts sessions whose `session_type` is
`MESOSCOPE_EXPERIMENT`. The gate is enforced inside `_create_dataset`: the first
resolved session's type is inspected, and a `ValueError` is raised if it is anything
else. The user-facing error message is
`"Unable to define dataset '{name}'. Dataset creation is currently supported only for
mesoscope experiment sessions..."`.

Every subsequent session in the batch must additionally share the first session's
`session_type` and `acquisition_system`. A mismatch raises a `ValueError` at dataset
creation time. Split the batch by acquisition system and session type before
calling `prepare_forging_batch_tool`.

| Session type           | Eligible for forging | Notes                                                       |
|------------------------|----------------------|-------------------------------------------------------------|
| `MESOSCOPE_EXPERIMENT` | yes                  | The only currently supported type                           |
| `LICK_TRAINING`        | no                   | Behavior processing runs, but forging refuses               |
| `RUN_TRAINING`         | no                   | Behavior processing runs, but forging refuses               |
| Anything else          | no                   | Refused at dataset creation                                 |

Use `/session-discovery` with `session_types=["mesoscope_experiment"]` to discover
forgeable sessions.

Eligibility is re-validated on every prepare call that hits a fresh dataset (existing
datasets are assumed already validated). A batch that mixes eligible and ineligible
sessions cannot be forged as a single dataset — split it first.

---

## Project and dataset hierarchy

The forging pipeline expects the canonical sollertia project layout:

```text
{project_root}/
├── {animal_name_A}/
│   ├── {session_name_1}/
│   │   ├── raw_data/
│   │   │   ├── session_data.yaml            ← session marker (see /session-discovery)
│   │   │   ├── hardware_state.yaml          ← required here
│   │   │   ├── experiment_configuration.yaml ← required here
│   │   │   ├── session_descriptor.yaml   ← required here (copied to output)
│   │   │   └── surgery_metadata.yaml            ← required on each animal's latest session
│   │   ├── processed_data/
│   │   │   ├── .../behavior_processing_tracker.yaml  ← discovered by rglob
│   │   │   └── .../single_recording_tracker.yaml     ← discovered by rglob
│   │   ├── data.feather                     ← FORGED OUTPUT (this pipeline)
│   │   └── session_descriptor.yaml       ← FORGED COPY (this pipeline)
│   └── {session_name_2}/...
└── {dataset_name}/                          ← FORGED DATASET HIERARCHY
    ├── dataset.yaml                         ← dataset marker
    ├── forging_tracker.yaml                 ← processing tracker
    └── {animal_name_A}/
        └── surgery_metadata.yaml                ← FORGED COPY (one per animal)
```

Key facts:

- **Dataset directory name:** the dataset name passed to `prepare_forging_batch_tool`
  becomes the directory name directly under `project_root`.
- **Session resolution:** each session name is resolved by `rglob(session_name)`
  against `project_root`. Zero or multiple matches is a fatal error.
- **Animal assignment:** the owning animal is derived from the session directory's
  parent name (`session_path.parent.name`).
- **Output path:** per-session forged data lives at `{session_path}/data.feather` —
  alongside `raw_data/` and `processed_data/`, not under them.
- **Dataset metadata vs output:** `clean_forging_output_tool` removes the dataset
  directory (tracker + `dataset.yaml` + per-animal `surgery_metadata.yaml` copies) but
  never touches per-session `data.feather` or `session_descriptor.yaml` files.

---

## Upstream prerequisite: behavior-processing output

The forging pipeline discovers the behavior data directory by locating its tracker via
`rglob("behavior_processing_tracker.yaml")` against `session.processed_data_path`. The
tracker parent is the behavior data directory (conventionally
`{processed_data_path}/behavior_data/`).

Exactly one `behavior_processing_tracker.yaml` must exist under `processed_data/` —
zero or multiple hits raise `FileNotFoundError` / `RuntimeError` respectively.

The pipeline then reads the following feathers from the behavior data directory. All
are uncompressed Arrow IPC produced by `/behavior-processing`:

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

See `/behavior-input-format` for upstream module eligibility rules and
`/behavior-results` for the column-level schema of each feather.

---

## Upstream prerequisite: cindra single-recording output

The forging pipeline discovers the cindra single-recording directory by locating its
tracker via `rglob("single_recording_tracker.yaml")` against
`session.processed_data_path`. The tracker parent is the cindra output directory
(conventionally `{processed_data_path}/mesoscope_data/<recording>/`).

Exactly one `single_recording_tracker.yaml` must exist under `processed_data/` — zero
or multiple hits raise `FileNotFoundError` / `RuntimeError` respectively.

The following files are read from the cindra directory at assembly time. All are
produced by `/cindra:single-recording-processing`:

| File                          | Loaded via               | Use                                                                      |
|-------------------------------|--------------------------|--------------------------------------------------------------------------|
| `cell_fluorescence.npy`       | `np.load(mmap_mode="r")` | Frame count (via shape); single-day cell fluorescence (filtered by mask) |
| `combined_metadata.npz`       | `np.load`                | `sampling_rate[0]` → scanning frequency (for TTL pulse duration window)  |
| `cell_classification.npy`     | `np.load(mmap_mode="r")` | Boolean mask (column 0 == 1) to filter non-cell ROIs                     |
| `neuropil_fluorescence.npy`   | `np.load(mmap_mode="r")` | Single-day neuropil fluorescence (filtered)                              |
| `subtracted_fluorescence.npy` | `np.load(mmap_mode="r")` | Single-day subtracted fluorescence (filtered)                            |
| `spikes.npy`                  | `np.load(mmap_mode="r")` | Single-day spike inference (filtered)                                    |

The frame count comes from the column dimension of `cell_fluorescence.npy` (its shape
is `(rois, frames)`). If the mesoscope frame log contains more TTL pulses than cindra
produced frames, the pipeline keeps only the **last** `frames` pulses, since aberrant
frames are assumed to come from pre-experiment triggering.

The ROI axis is filtered once by the classification mask before the
array is transposed to `(frames, rois)` and stored as a Polars `Array(Float32)`
column.

---

## Upstream prerequisite: cindra multi-day output

The forging pipeline derives the multi-day directory as
`{cindra_data_path.parent}/multiday/{dataset_name}/`. The dataset name passed to
`prepare_forging_batch_tool` MUST match the dataset name under which the multi-day
cindra pipeline was run — the two are joined blindly without fallback discovery.

The following files are read from the multi-day directory. All are produced by
`/cindra:multi-recording-processing`:

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
`{cindra_parent}/multiday/{dataset_name}/` join then points at a non-existent
directory, and the assembly raises `FileNotFoundError` on `cell_fluorescence.npy`.
Fix by rerunning `/cindra:multi-recording-processing` with the matching dataset name,
or by pointing the forging run at the name that was used upstream.

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
`data.feather` at the end of each session assembly (the copy is placed directly under
the session root, next to `data.feather`). The file is parsed as
`MesoscopeExperimentDescriptor` and carries experimenter-authored runtime context —
`experimenter`, `animal_weight_g`, dispensed / consumed water volumes, the `incomplete`
completion flag, and `experimenter_notes`.

Presence is verified up front inside `_assemble_session_dataset`, before any cindra or
behavior work is performed: a missing file raises `FileNotFoundError` with
`"Unable to assemble session '{name}'. The session's raw data directory does not
contain a 'session_descriptor.yaml' file at '{path}'. The experiment descriptor is
required for every session in a forged dataset."`. Content is not validated by the
forging pipeline itself (the file is only copied, not read), so authoring and
validation belong to the assets plugin.

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

| Upstream producer                | Required skill                        | Artifact location                                                                                                                                                                 | Gates forging |
|----------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------|
| Behavior processing pipeline     | `/behavior-processing`                | `{processed_data}/.../behavior_processing_tracker.yaml` + feathers                                                                                                                | yes           |
| Cindra single-recording pipeline | `/cindra:single-recording-processing` | `{processed_data}/.../single_recording_tracker.yaml` + `*.npy` / `*.npz`                                                                                                          | yes           |
| Cindra multi-recording pipeline  | `/cindra:multi-recording-processing`  | `{cindra_parent}/multiday/{dataset_name}/*.npy` (dataset name must match)                                                                                                         | yes           |
| Mesoscope-VR acquisition runtime | (acquisition-side; no skill)          | `{raw_data}/hardware_state.yaml`, `{raw_data}/experiment_configuration.yaml`, `{raw_data}/session_descriptor.yaml`, `{raw_data}/surgery_metadata.yaml` (latest session per animal) | yes           |

**Ordering constraint:** all four upstream producers MUST complete for every session
in a dataset BEFORE `/dataset-forging` can assemble. Running the forging pipeline
against a partially-processed session raises during dataset assembly — the
`prepare_forging_batch_tool` call itself will mostly succeed (it only resolves the
dataset hierarchy and initializes the tracker), but dataset creation will fail up
front if any animal's latest session is missing `surgery_metadata.yaml`, and per-session
jobs will fail during execution with a `FileNotFoundError` on a missing tracker,
missing `.npy`, or missing `session_descriptor.yaml`.

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
- [ ] /behavior-processing completed — behavior_processing_tracker.yaml + feathers present
-   [ ] mesoscope_frame_data.feather
-   [ ] system_state_data.feather, lick_data.feather, valve_data.feather
-   [ ] encoder_data.feather
-   [ ] vr_trigger_zone_data.feather, vr_cue_data.feather, trial_data.feather, runtime_state_data.feather
- [ ] /cindra:single-recording-processing completed — single_recording_tracker.yaml + outputs present
-   [ ] cell_fluorescence.npy, combined_metadata.npz, cell_classification.npy
-   [ ] neuropil_fluorescence.npy, subtracted_fluorescence.npy, spikes.npy
- [ ] /cindra:multi-recording-processing completed with matching dataset name
-   [ ] multiday/{dataset_name}/cell_fluorescence.npy and companions present
```

---

## Related skills

| Skill                                  | Relationship                                                                |
|----------------------------------------|-----------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`       | Prerequisite: MCP server connectivity                                       |
| `/session-discovery`                   | Upstream: session discovery and filtering                                   |
| `/dataset-forging`                     | Downstream: consumes the inputs documented here                             |
| `/dataset-forging-results`             | Downstream: documents the output derived from these inputs                  |
| `/behavior-processing`                 | Upstream producer of behavior feathers                                      |
| `/behavior-input-format`               | Reference: upstream-of-upstream input format for behavior feathers          |
| `/behavior-results`                    | Reference: schema of the behavior feathers consumed here                    |
| `/cindra:single-recording-processing`  | Upstream producer of cindra single-recording outputs                        |
| `/cindra:multi-recording-processing`   | Upstream producer of cindra multi-day outputs                               |
| `/cindra:single-recording-results`     | Reference: schemas of the cindra single-recording outputs                   |
| `/cindra:multi-recording-results`      | Reference: schemas of the cindra multi-day outputs                          |
