---
name: dataset-forging-results
description: >-
  Reference for dataset-forging output formats, per-session `data.feather` discovery,
  dataset-level tracker state, and output verification. Use when evaluating forged datasets,
  when the user asks about fluorescence array shapes or trial masking semantics, or when
  auditing datasets before downstream analysis.
user-invocable: false
---

# Dataset forging results

Complete output data format documentation for the forging pipeline. Covers the
per-session `data.feather` layout, the dataset-level tracker, the full column schema,
verification via MCP tool, data querying, and interpretation guidance.

---

## Scope

**Covers:**
- Per-session output files (`{session_root}/data.feather` plus a copy of
  `session_descriptor.yaml`) and the dataset-level hierarchy
  (`{project_root}/{dataset_name}/dataset.yaml`, `forging_tracker.yaml`, and
  per-animal `{animal}/surgery_metadata.yaml` copies)
- Full column schema: cindra fluorescence + behavior + runtime / experiment
- Column masking rules (`_mask_non_run_experiment_data`) and conditional columns
- Fluorescence array interpretation (shape, dtype, cell filtering)
- Frame alignment: `frame`, `time_us`, `elapsed_minutes` derivation
- `verify_forging_output_tool` usage
- `query_forging_data_tool` usage and sample-row interpretation

**Does not cover:**
- Batch processing workflow (see `/dataset-forging`)
- Input data format (see `/dataset-forging-input-format`)
- Session discovery (see the assets plugin's `/session-discovery`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)
- Upstream behavior feather schemas (see `/behavior-results`)
- Upstream cindra output schemas (see `/cindra:single-recording-results` and
  `/cindra:multi-recording-results`)

**Note:** `/cindra:*` refers to the **cindra** plugin from the
[cindra marketplace](https://github.com/Sun-Lab-NBB/cindra).

---

## Available tools

### Verification tool

| Tool                             | Purpose                                                                                                                               |
|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `verify_forging_output_tool`     | Loads the dataset marker, checks every session's `data.feather` and copied descriptor, checks per-animal surgery copies, reads tracker |

**Parameters:**

| Parameter      | Type  | Default    | Description                                                                                               |
|----------------|-------|------------|-----------------------------------------------------------------------------------------------------------|
| `dataset_path` | `str` | (required) | Absolute path to the dataset root directory (`{project_root}/{dataset_name}/`, containing `dataset.yaml`) |

**Return structure:**

```text
verified:            Boolean — True when every file is valid AND at least one session exists AND at least one animal exists
dataset_path:        Echo of the input dataset root
dataset_name:        Name loaded from dataset.yaml
files[]:             Per-session verification results:
  session_name:      The session's name
  animal:            The session's owning animal name
  file:              Absolute path to {session_path}/data.feather
  valid:             True if the feather file AND the companion descriptor are both valid
  columns:           Column names in the DataFrame (when the feather loaded)
  row_count:         Total row count (when the feather loaded)
  error:             Error message (when the feather is invalid or missing)
  descriptor:        Nested verification of the copied session_descriptor.yaml:
    file:            Absolute path to {session_path}/session_descriptor.yaml
    valid:           True if the file parses as MesoscopeExperimentDescriptor
    error:           Error message (when missing or unparseable)
total_files:         Number of sessions in the dataset (== number of per-session checks)
animals[]:           Per-animal verification of the copied surgery_metadata.yaml:
  animal:            The animal name
  file:              Absolute path to {dataset_root}/{animal}/surgery_metadata.yaml
  valid:             True if the file parses as SurgeryData
  error:             Error message (when missing or unparseable)
total_animals:       Number of animals in the dataset (== number of surgery checks)
tracker:             {jobs[], summary} loaded from forging_tracker.yaml; {} if absent;
                     {"error": "..."} if unreadable
error:               Present only when dataset_path is invalid or dataset.yaml cannot be loaded
```

The tool walks the dataset's session list (from `DatasetData.sessions`) and checks
each session's `data.feather` via the shared `analyze_feather_file` helper with
`max_sample_rows=0`, so feather verification only costs metadata reads (no full table
materialization). A missing `data.feather` is reported as `valid: False` with
`error: "data.feather not found."`. The companion `session_descriptor.yaml` copied
alongside is separately loaded via `MesoscopeExperimentDescriptor.from_yaml`, and any
feather-only-valid entry is demoted to `valid: False` when the descriptor is missing
or unparseable. For every unique animal in `DatasetData.animals`, the per-animal
`{animal}/surgery_metadata.yaml` is loaded via `SurgeryData.from_yaml` and reported in
the `animals[]` list. Any invalid feather, descriptor, or surgery file flips the
top-level `verified` flag to False.

The dataset-level forging tracker (`forging_tracker.yaml`) is always attempted
alongside. If the tracker reports failed or scheduled jobs, that indicates the
forging run is incomplete regardless of which feathers happen to exist on disk —
re-invoke `/dataset-forging`.

### Query tool

| Tool                        | Purpose                                                                          |
|-----------------------------|----------------------------------------------------------------------------------|
| `query_forging_data_tool`   | Reads feather files, returns row counts, columns, inter-row timing, sample rows  |

**Parameters:**

| Parameter         | Type        | Default    | Description                                                                     |
|-------------------|-------------|------------|---------------------------------------------------------------------------------|
| `feather_files`   | `list[str]` | (required) | Absolute paths to `data.feather` files (typically from `verify_forging_output_tool`) |
| `max_sample_rows` | `int`       | `10`       | Number of sample rows to include per file                                        |

**Return structure:**

```text
results[]:                    Per-file analysis:
  file:                       Absolute path to the feather file
  summary:
    total_rows:               Total row count
    first_timestamp_us:       First timestamp (microseconds since UTC epoch), when a time column exists
    last_timestamp_us:        Last timestamp
    duration_us:              Total duration in microseconds
    duration_seconds:         Total duration in seconds
    columns:                  List of column names
  inter_row_timing:           mean/median/std/min/max intervals; populated only when the feather
                              has >= 2 rows AND contains one of the recognized time columns
                              (`timestamp_us`, `time_us`, or `frame_time_us`)
  sample_rows[]:              First N rows (binary and array data are omitted for readability)
  error:                      Present only when the file cannot be read
total_files:                  Number of files analyzed
```

Sample rows are compacted for display: the shared `analyze_feather_file` helper
omits array-typed and binary-typed column values in samples, so the fluorescence
columns appear as placeholders. To actually inspect fluorescence values, load the
file with Polars directly (`pl.read_ipc(path, memory_map=True)`) — the query tool is
intended for schema and timing summaries, not for array content.

### Cross-dataset overview

`get_forging_batch_status_overview_tool` is documented in `/dataset-forging`. Use it
here when you want a cross-dataset summary of tracker state before drilling into any
single dataset's outputs.

---

## Recommended query order

1. **`get_forging_batch_status_overview_tool`** (from `/dataset-forging`) — aggregate
   status across every dataset under a project root. Tells you which datasets have
   completed forging.
2. **`verify_forging_output_tool`** — per-dataset verification against the expected
   per-session `data.feather` set and the dataset tracker.
3. **`query_forging_data_tool`** — drill into specific sessions for schema, timing,
   and sample rows.

---

## Output directory structure

Each forged dataset writes into two locations:

```text
{project_root}/
├── {animal_A}/
│   ├── {session_1}/
│   │   ├── data.feather                ← per-session forged output
│   │   └── session_descriptor.yaml  ← copied from raw_data/ at assembly
│   └── {session_2}/
│       ├── data.feather
│       └── session_descriptor.yaml
└── {dataset_name}/                     ← dataset hierarchy
    ├── dataset.yaml                    ← dataset marker (DatasetData)
    ├── forging_tracker.yaml            ← forging ProcessingTracker
    └── {animal_A}/
        └── surgery_metadata.yaml           ← copied once per animal at dataset creation
```

Key facts:

- **Per-session output:** `{session_path}/data.feather`. The path comes from
  `DatasetSession.session_path.joinpath("data.feather")` inside the pipeline. This is
  directly under the session root, not under `processed_data/`. A copy of the
  session's `session_descriptor.yaml` is written next to it so the forged session
  carries experimenter context (animal weight, water dispensed/consumed, completion
  status, notes) without reaching back into the raw session.
- **Dataset hierarchy:** `{project_root}/{dataset_name}/` stores the metadata
  (`dataset.yaml`), the processing tracker (`forging_tracker.yaml`), and one
  `{animal}/surgery_metadata.yaml` per animal copied from each animal's latest session.
  `clean_forging_output_tool` removes this directory but does not touch the
  per-session `data.feather` or `session_descriptor.yaml` files.
- **File format:** uncompressed Arrow IPC, memory-mappable via
  `pl.read_ipc(source, memory_map=True)`.
- **Cross-session layout:** every session in a dataset emits exactly one
  `data.feather` (plus the descriptor copy) with the same feather column schema
  modulo conditional columns.

---

## Output schema

A single forged `data.feather` is produced per session. Columns come from three
upstream sources that are horizontally concatenated in this order: cindra
fluorescence → behavior → runtime/experiment. After concatenation,
`_mask_non_run_experiment_data` rewrites `cue`, `trial`, and `trial_type` values for
rows whose `system_state` is not `"run"`.

All timestamp columns are **microseconds since UTC epoch** (`UInt64`), already
resolved against the runtime DataLogger onset upstream — no further onset offset is
needed.

### Cindra fluorescence block (always present)

| Column                             | Dtype           | Description                                                                    |
|------------------------------------|-----------------|--------------------------------------------------------------------------------|
| `frame`                            | `UInt32`        | 1-indexed frame number (reassigned after TTL alignment)                        |
| `time_us`                          | `UInt64`        | Frame timestamp (rising edge of the mesoscope TTL pulse), microseconds UTC     |
| `elapsed_minutes`                  | `Float32`       | Minutes elapsed since the first frame's `time_us`, rounded to two decimals      |
| `single_day_cell_fluorescence`     | `Array(Float32)`| Single-day cell traces, shape `(frames, rois)`, cell-filtered                  |
| `single_day_neuropil_fluorescence` | `Array(Float32)`| Single-day neuropil traces, shape `(frames, rois)`, cell-filtered              |
| `single_day_subtracted_fluorescence`| `Array(Float32)`| Single-day neuropil-subtracted traces, shape `(frames, rois)`, cell-filtered  |
| `single_day_spikes`                | `Array(Float32)`| Single-day spike inference, shape `(frames, rois)`, cell-filtered              |
| `multi_day_cell_fluorescence`      | `Array(Float32)`| Multi-day tracked cell traces, shape `(frames, rois)`, already cell-filtered   |
| `multi_day_neuropil_fluorescence`  | `Array(Float32)`| Multi-day tracked neuropil traces                                              |
| `multi_day_subtracted_fluorescence`| `Array(Float32)`| Multi-day tracked subtracted traces                                            |
| `multi_day_spikes`                 | `Array(Float32)`| Multi-day tracked spike inference                                              |

### Behavior block (time columns dropped)

| Column          | Dtype                        | Conditional                                              | Description                                                         |
|-----------------|------------------------------|----------------------------------------------------------|---------------------------------------------------------------------|
| `brake`         | `UInt8`                      | `brake_data.feather` present (mesoscope w/ brake)        | 1 when measured brake torque exceeds `minimum_brake_strength`, else 0 |
| `screens`       | `UInt8`                      | `screen_data.feather` present (mesoscope only)           | Screen state code (0 = off, 1 = on)                                 |
| `torque_N_cm`   | `Float32`                    | `torque_data.feather` present (non-run-training only)    | Torque in N·cm; forced to 0 when `system_state == "run"`            |
| `distance_cm`   | `Float32`                    | `encoder_data.feather` present (encoder-equipped)        | Cumulative distance; forward-filled outside run, clamped to 0 before the first run |
| `speed_cm_s`    | `Float32`                    | `encoder_data.feather` present                           | Running speed; forced to 0 when `system_state != "run"`             |
| `lick`          | `UInt8`                      | always                                                   | Lick state (0 / 1)                                                  |
| `water_uL`      | `Float32`                    | always                                                   | Dispensed water volume in microliters (discrete-interpolated)       |
| `reward`        | `Enum["no","tone","yes"]`    | always                                                   | Reward classification (see masking rules)                           |
| `system_state`  | `Enum[hardware_state_codes]` | always                                                   | System state name resolved via `MesoscopeHardwareState.system_state_codes` |

### Runtime / experiment block

| Column                | Dtype                                    | Conditional                                | Description                                                     |
|-----------------------|------------------------------------------|--------------------------------------------|-----------------------------------------------------------------|
| `trial`               | `UInt16`                                 | always                                     | Sequential trial ID (1-indexed); masked to `65535` outside run  |
| `trial_type`          | `Enum[trial_types + "undefined"]`        | always                                     | Trial type; masked to `"undefined"` outside run                 |
| `cue`                 | `UInt8`                                  | always                                     | VR wall cue code; masked to `255` outside run                   |
| `in_trigger_zone`     | `UInt8`                                  | always                                     | 1 when the reference distance is inside a trigger zone, else 0  |
| `runtime_state`       | `Enum[experiment_states + "idle"]`       | always                                     | Runtime state; `0 → "idle"` hardcoded default                   |
| `reinforcing_guided`  | `UInt8`                                  | reinforcing guidance toggled during session | 1 when reinforcing guidance is active, else 0                  |
| `aversive_guided`     | `UInt8`                                  | aversive guidance toggled during session   | 1 when aversive guidance is active, else 0                      |

### Masking rules

`_mask_non_run_experiment_data` rewrites specific columns wherever `system_state` is
not `"run"` (covers `idle` and `rest` states):

| Column       | Masked value    | Dtype   |
|--------------|-----------------|---------|
| `cue`        | `255`           | `UInt8` |
| `trial`      | `65535`         | `UInt16` |
| `trial_type` | `"undefined"`   | Enum member |

These sentinels sit outside the legitimate cue-code and trial-id ranges, so sessions
with many hundreds of trials can still be masked unambiguously. Downstream consumers
should filter on `system_state == "run"` before using `cue`, `trial`, or `trial_type`.

Additionally:
- `torque_N_cm` is forced to `0.0` wherever `system_state == "run"`, because the
  torque sensor is disabled in the run state.
- `distance_cm` is forward-filled across non-run regions and clamped to `0.0` before
  the first run transition.
- `speed_cm_s` is forced to `0.0` wherever `system_state != "run"`.

---

## Fluorescence array interpretation

The eight fluorescence columns are Polars `Array(Float32)` columns, meaning every row
contains a fixed-length vector of per-ROI values for that frame. The effective shape
of each column is `(frames, rois)`:

- `frames` equals the total number of rows in the feather (equal across all
  fluorescence columns).
- `rois` equals the number of cell ROIs after filtering.

ROI filtering semantics:

| Column family        | Source file                                   | ROI filter                                                |
|----------------------|-----------------------------------------------|-----------------------------------------------------------|
| `single_day_*`       | cindra single-recording `.npy`                | `cell_classification[:, 0] == 1` (applied at forging time) |
| `multi_day_*`        | cindra multi-day `.npy`                       | None (multi-day pipeline already emits cell-filtered arrays) |

Because the single-day and multi-day arrays are filtered in different ways, their ROI
counts can differ within the same session. The frame count is shared (it comes from
the cindra single-day shape and drives reference-time construction).

To materialize fluorescence values:

```python
import polars as pl
df = pl.read_ipc("/path/to/session/data.feather", memory_map=True)
single_day_cells = df["single_day_cell_fluorescence"].to_numpy()  # (frames, rois)
```

---

## Frame alignment

The forging pipeline aligns cindra fluorescence frames to mesoscope TTL rising
edges, recorded by the microcontroller during acquisition and available in
`mesoscope_frame_data.feather`:

1. Rising and falling edges are extracted from `ttl_state` diffs and joined per pulse.
2. Pulses whose duration falls outside
   `1000 / sampling_rate ± 20 ms` are filtered out. The sampling rate comes from
   `combined_metadata.npz["sampling_rate"][0]`.
3. If the count of valid pulses exceeds the cindra frame count, only the last
   `frame_count` pulses are kept (aberrant frames are assumed to come from before the
   main experiment runtime).
4. Frame IDs are reassigned from 1 as `UInt32`. `time_us` is the microsecond UTC
   timestamp of the pulse rising edge. `elapsed_minutes` is
   `(time_us - time_us.min()) / 60_000_000`, rounded to two decimals as `Float32`.

Every other data source (behavior columns from feathers sampled on microcontroller
time, runtime state from the log archive) is interpolated onto this frame-aligned
`time_us` vector. Trial, trial type, cue, and trigger-zone columns are interpolated
onto the corresponding `distance_cm` reference vector rather than onto `time_us`.

---

## Interpretation guidance

### Common queries

- **"How many trials were run?"** — `df.filter(pl.col("trial") != 65535).select(pl.col("trial").max())`
- **"How far did the animal run in the task?"** — `df.select(pl.col("distance_cm").max())`
  (already forward-filled across non-run states).
- **"What fraction of frames were in a trigger zone during a given trial?"** —
  `df.filter(pl.col("trial") == k).select(pl.col("in_trigger_zone").mean())`
- **"Which ROIs are cells in this session?"** — implicit; single-day fluorescence
  columns are already cell-filtered. ROI index is consistent across all
  `single_day_*` columns; multi-day columns share their own ROI index.

### Gotchas

- Fluorescence columns are `Array(Float32)` — use `.to_numpy()` to materialize them.
  `query_forging_data_tool` samples omit array values for readability; use Polars
  directly for array inspection.
- `trial`, `trial_type`, and `cue` are sentinel-masked outside `system_state == "run"`
  — always filter on `system_state` before aggregating these columns.
- The single-day and multi-day fluorescence ROI counts can differ. Do not assume
  column shapes are identical across `single_day_*` and `multi_day_*` families.
- `torque_N_cm`, `distance_cm`, and `speed_cm_s` are post-processed (see "Masking
  rules"). Do not reuse them expecting raw sensor values.
- Conditional columns (`brake`, `screens`, `torque_N_cm`, `distance_cm`, `speed_cm_s`,
  `reinforcing_guided`, `aversive_guided`) may be absent for sessions whose hardware
  or runtime state did not populate them. Inspect the columns list via
  `verify_forging_output_tool` before querying.

---

## Related skills

| Skill                                  | Relationship                                                              |
|----------------------------------------|---------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`       | Prerequisite: MCP server connectivity                                     |
| `/dataset-forging`                     | Upstream: produces the outputs documented here                            |
| `/dataset-forging-input-format`        | Reference: inputs that shape this schema                                  |
| `/behavior-results`                    | Reference: upstream behavior feather schemas consumed at forging time     |
| `/cindra:single-recording-results`     | Reference: upstream cindra single-recording output schemas                |
| `/cindra:multi-recording-results`      | Reference: upstream cindra multi-day output schemas                       |

---

## Verification checklist

```text
Dataset Forging Results Audit:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Ran get_forging_batch_status_overview_tool to locate completed datasets
- [ ] Ran verify_forging_output_tool per dataset; confirmed every data.feather, companion session_descriptor.yaml, and per-animal surgery_metadata.yaml is valid
- [ ] Cross-checked forging_tracker.yaml tracker state — no FAILED or lingering SCHEDULED jobs
- [ ] Queried at least one representative session via query_forging_data_tool to confirm schema
- [ ] Verified expected conditional columns are present (brake / torque / guidance as applicable)
- [ ] Confirmed fluorescence arrays load with the expected (frames, rois) shape via Polars
- [ ] Escalated any discrepancies to /dataset-forging for reset / retry
```
