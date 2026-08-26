---
name: behavior-results
description: >-
  Reference for behavior-processing output formats, feather file discovery, and output
  verification. Use when evaluating behavior-processing results, when the user asks about
  runtime state, trial data, camera timestamps, or microcontroller feather contents, or when
  auditing completed sessions for downstream analysis.
user-invocable: false
---

# Processing results

Complete output data format documentation for the behavior processing pipeline. Covers the
`runtime_data/` and `microcontroller_data/` output layout, per-job-type feather schemas, verification
via MCP tool, data querying, and interpretation guidance.

---

## Scope

**Covers:**
- Output directory structure and the `runtime_data/` / `microcontroller_data/` layout
- Runtime output schemas (system state, runtime state, guidance, cue, trial, trigger zone)
- Camera output roster (per-camera timestamp feathers produced by the separate video pipeline and written
  to `processed_data/video_data/`)
- Microcontroller output roster (per-module feather files present after processing)
- `verify_behavior_processing_output_tool` usage
- `query_behavior_data_tool` usage and sample-row interpretation
- Interpretation guidance for state feathers, timing feathers, trial feathers

**Does not cover:**
- Input data format (see `forging:behavior-input-format`)
- Batch processing workflow (see `forging:behavior-processing`)
- Processing-pipeline doctrine: prepare-then-execute, trackers, worker budget (see `forging:data-processing-design`)
- The separate video pipeline that extracts the camera timestamp feathers (see `forging:camera-timestamp-extraction`)
- The agnostic module-feather parsing primitives (see `forging:microcontroller-primitives`)
- Per-module `(module_type, module_id)` conversions, event codes, and calibration fields (see
  `mesoscope:mesoscope-vr-module-parsing`)
- Session discovery (see `assets:session-discovery`)
- MCP server connectivity (see `forging:forging-mcp-environment-setup`)

**Note:** `ataraxis@video:log-processing` and `ataraxis@communication:log-processing` refer to skills in the
**video** and **communication** plugins from the [ataraxis marketplace](https://github.com/Sun-Lab-NBB/ataraxis).
They own, respectively, the upstream camera-log production that the video pipeline turns into camera timestamp
feathers and the upstream module-feather production that the microcontroller jobs parse.

---

## Available tools

### Verification tool

| Tool                                         | Purpose                                                             |
|----------------------------------------------|---------------------------------------------------------------------|
| `verify_behavior_processing_output_tool`     | Scans both output directories, validates feathers, reports trackers |

**Parameters:**

| Parameter      | Type  | Default    | Description                                                                                                                                                     |
|----------------|-------|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `session_path` | `str` | (required) | Absolute path to the session root directory. The tool loads `SessionData` to resolve `processed_data_path` and inspects `{processed_data_path}/runtime_data/` and `{processed_data_path}/microcontroller_data/`. |

**Return structure:**

```text
verified:            Boolean — True when all files are readable AND at least one file exists
session_path:        Echo of the input session root
data_path:           Absolute path to the inspected output subdirectory (always under processed_data/)
files[]:             Per-file verification results:
  file:              Absolute path to the feather file
  filename:          Filename (without directory)
  valid:             Boolean — True if the file could be loaded via Polars IPC
  columns:           List of column names in the DataFrame (when valid)
  row_count:         Total row count (when valid)
  error:             Error message (when invalid)
total_files:         Number of feather files found
tracker:             Dict with `jobs[]` (per-job fields below) and `summary` (aggregate counts);
                     {} if no tracker file is present; {"error": "..."} if unreadable
```

The tool loads the session's SessionData marker, lists `{session.processed_data_path}/runtime_data/` and
`{session.processed_data_path}/microcontroller_data/` non-recursively with `glob("*.feather")`, and loads each
file via the shared `analyze_feather_file` helper with `max_sample_rows=0`, so verification only costs metadata
reads (no full table materialization). The output location is not configurable — behavior outputs always live
under the session's processed data hierarchy.

### Query tool

| Tool                           | Purpose                                                                      |
|--------------------------------|------------------------------------------------------------------------------|
| `query_behavior_data_tool`     | Reads feather files, returns row counts, columns, timing stats, sample rows  |

**Parameters:**

| Parameter         | Type        | Default    | Description                                                |
|-------------------|-------------|------------|------------------------------------------------------------|
| `feather_files`   | `list[str]` | (required) | Absolute paths to feather files (from verification output) |
| `max_sample_rows` | `int`       | `10`       | Number of sample rows to include per file                  |

**Return structure:**

```text
results[]:                   Per-file analysis results:
  file:                      Absolute path to the feather file
  summary:
    total_rows:              Total row count
    first_timestamp_us:      First timestamp (microseconds since UTC epoch), when a time column exists
    last_timestamp_us:       Last timestamp
    duration_us:             Total duration in microseconds
    duration_seconds:        Total duration in seconds
    columns:                 List of column names
  inter_row_timing:          mean/median/std/min/max intervals; populated only when the feather
                             has >= 2 rows AND contains one of the recognized time columns
                             (`timestamp_us`, `time_us`, or `frame_time_us`)
  sample_rows[]:             First N rows (binary data omitted for readability)
  error:                     Present only when the file cannot be read
total_files:                 Number of files analyzed
```

`query_behavior_data_tool` delegates to the same `analyze_feather_file` helper as the verification
tool but requests the configured number of sample rows. Use it whenever you need to eyeball actual
row values or summarize inter-row timing. Note that behavior outputs use `time_us` (and camera feathers
`frame_time_us`), not `timestamp_us`, so the timing statistics are driven by the `time_us` / `frame_time_us`
candidates for this output set.

### Overview tool (referenced from `forging:behavior-processing`)

`get_batch_status_overview_tool` is documented in `forging:behavior-processing`. Use it here when you need a
cross-session summary of tracker state before drilling into any single session's outputs.

---

## Recommended query order

1. **`get_batch_status_overview_tool`** (from `forging:behavior-processing`) — aggregate status across all
   sessions under a root directory, so you know which sessions have finished processing.
2. **`verify_behavior_processing_output_tool`** — per-session verification against the expected
   feather set and the tracker (takes a session path).
3. **`query_behavior_data_tool`** — drill into specific files for schemas, timing, and sample rows.

---

## Output directory structure

Behavior processing writes its output into two subdirectories of the session's `processed_data_path`: the
runtime job writes its feathers into `runtime_data/`, and the microcontroller jobs write theirs into
`microcontroller_data/`. The camera `{name}_timestamps.feather` files are NOT behavior-processing outputs and do
not live in either directory: the separate video pipeline writes them into `processed_data/video_data/` (see the
camera-feathers note below). These locations are static and cannot be overridden:

```text
{session_root}/
└── processed_data/
    ├── runtime_data/
    │   ├── runtime_processing_tracker.yaml           # runtime job tracker
    │   ├── system_state_data.feather                 # runtime job — always
    │   ├── runtime_state_data.feather                # runtime job — always
    │   ├── reinforcing_guidance_state_data.feather   # runtime job — experiments only
    │   ├── aversive_guidance_state_data.feather      # runtime job — experiments only
    │   ├── vr_cue_data.feather                       # runtime job — experiments only
    │   ├── vr_trigger_zone_data.feather              # runtime job — experiments only
    │   └── trial_data.feather                        # runtime job — experiments only
    ├── microcontroller_data/
    │   ├── microcontroller_processing_tracker.yaml   # microcontroller job tracker
    │   ├── encoder_data.feather                      # microcontroller (2,1)
    │   ├── mesoscope_frame_data.feather              # microcontroller (1,1)
    │   ├── brake_data.feather                        # microcontroller (3,1)
    │   ├── valve_data.feather                        # microcontroller (5,1)
    │   ├── gas_puff_data.feather                     # microcontroller (5,2)
    │   ├── lick_data.feather                         # microcontroller (4,1)
    │   ├── torque_data.feather                       # microcontroller (6,1)
    │   └── screen_data.feather                       # microcontroller (7,1)
    └── video_data/
        └── {name}_timestamps.feather                 # written by the separate video pipeline
```

All feather files are uncompressed Arrow IPC, so they are memory-mappable and directly loadable via
`pl.read_ipc(source, memory_map=True)`. Which behavior feather files are present depends on the session type, the
presence of upstream inputs, and which modules were eligible under the session's hardware state
(see `forging:behavior-input-format`). The `{name}_timestamps.feather` files are produced by the separate video
pipeline (job `camera_timestamp_extraction`, owned by `forging:camera-timestamp-extraction`) and written into
`processed_data/video_data/`; each is named from the colloquial source name registered in the acquisition-time camera
manifest (for example, a `face_camera` source produces `face_camera_timestamps.feather`), so the exact roster of
camera files is session-specific — see `forging:camera-timestamp-extraction`.

---

## Output schemas

All timestamp columns in this section are **microseconds since UTC epoch** (`UInt64`), already
resolved against the runtime DataLogger onset by `LogArchiveReader`. No further onset offset is
needed.

### Runtime state feathers (all sessions)

**`system_state_data.feather`** — VR system state transitions

| Column         | Dtype    | Description                                  |
|----------------|----------|----------------------------------------------|
| `time_us`      | `UInt64` | Timestamp                                    |
| `system_state` | `UInt8`  | Opaque state code from the acquisition rig   |

**`runtime_state_data.feather`** — session runtime state transitions

| Column          | Dtype    | Description                                  |
|-----------------|----------|----------------------------------------------|
| `time_us`       | `UInt64` | Timestamp                                    |
| `runtime_state` | `UInt8`  | Opaque runtime state code                    |

### Runtime experiment feathers (MESOSCOPE_EXPERIMENT only)

**`reinforcing_guidance_state_data.feather`** — only written when guidance events were recorded

| Column                        | Dtype    | Description                     |
|-------------------------------|----------|---------------------------------|
| `time_us`                     | `UInt64` | Timestamp                       |
| `reinforcing_guidance_state`  | `UInt8`  | Reinforcing guidance state code |

**`aversive_guidance_state_data.feather`** — only written when guidance events were recorded

| Column                    | Dtype    | Description                  |
|---------------------------|----------|------------------------------|
| `time_us`                 | `UInt64` | Timestamp                    |
| `aversive_guidance_state` | `UInt8`  | Aversive guidance state code |

**`vr_cue_data.feather`** — per-cue VR wall cue sequence

| Column                 | Dtype     | Description                                         |
|------------------------|-----------|-----------------------------------------------------|
| `vr_cue`               | `UInt8`   | Wall cue code from the experiment configuration     |
| `traveled_distance_cm` | `Float64` | Cumulative distance at which the cue appears (cm)   |

**`vr_trigger_zone_data.feather`** — per-trial trigger zone boundaries

| Column                   | Dtype     | Description                                |
|--------------------------|-----------|--------------------------------------------|
| `trigger_zone_start_cm`  | `Float64` | Start of the trigger zone (cumulative cm)  |
| `trigger_zone_end_cm`    | `Float64` | End of the trigger zone (cumulative cm)    |

**`trial_data.feather`** — per-trial type and start distance

| Column                 | Dtype     | Description                                                      |
|------------------------|-----------|------------------------------------------------------------------|
| `trial_type_index`     | `Int32`   | Index into the experiment configuration's trial structures list  |
| `traveled_distance_cm` | `Float64` | Cumulative distance at which the trial starts                    |

The runtime pipeline decomposes VR wall cue sequences into trials using the
`MesoscopeExperimentConfiguration.trial_structures` mapping. If decomposition fails, the runtime
job fails with a `RuntimeError` identifying the offending sequence position — see
`forging:behavior-processing` error routing. The decomposition algorithm itself (cue-sequence motif
matching, trial geometry) is owned by `mesoscope:mesoscope-vr-trial-decomposition`.

### Camera feathers

The camera `{name}_timestamps.feather` files are produced by the SEPARATE video pipeline
(`run_video_processing_pipeline`, job `camera_timestamp_extraction`, CLI `mesoscope process video`), NOT by the
behavior pipeline. That pipeline extracts frame acquisition timestamps from each raw VideoSystem log archive and
writes a fresh feather into `processed_data/video_data/` (its sink is `session.processed_data.video_data_path`), so
the files sit apart from the behavior outputs, and their job state lives in the SEPARATE video tracker
(`session.processed_data.video_tracker_path`, which resolves to `video_data/video_processing_tracker.yaml`), not a
behavior tracker. A camera feather is never a behavior-tracker job. Each output is NOT a hardlink to any
upstream axvs output, and there is no hardcoded source-ID-to-name registry: the output filename for each source comes
from the colloquial name registered in the acquisition-time camera manifest, so a `face_camera` source yields
`face_camera_timestamps.feather`. See `forging:camera-timestamp-extraction` (and upstream
`ataraxis@video:log-processing`) for how source IDs become named feathers, where the inputs live, and the manifest
contract.

| Column          | Dtype    | Description                                                       |
|-----------------|----------|-------------------------------------------------------------------|
| `frame_time_us` | `UInt64` | Per-frame acquisition timestamp (microseconds since the UTC epoch) |

The `frame_time_us` column is already in UTC microseconds, so it can be compared directly to any `time_us` column in
the rest of this output set.

### Microcontroller feathers

Each microcontroller job transforms a single axci module feather into one domain-specific output feather, keyed by
the module's `(module_type, module_id)` pair. For verification, this skill carries only the **output-file roster**:
the filename to expect on disk for each module key, and which hardware-state field gates the module's presence. The
per-module output column schemas, event codes, unit conversions, and calibration fields are owned by
`mesoscope:mesoscope-vr-module-parsing` — consult that skill (or `query_behavior_data_tool`) for column-level detail.

| Module key | Output filename               | Presence gated by                                          |
|------------|-------------------------------|------------------------------------------------------------|
| `(2, 1)`   | `encoder_data.feather`        | `cm_per_pulse`                                             |
| `(1, 1)`   | `mesoscope_frame_data.feather`| `recorded_mesoscope_ttl`                                   |
| `(3, 1)`   | `brake_data.feather`          | `maximum_brake_strength`, `minimum_brake_strength`        |
| `(5, 1)`   | `valve_data.feather`          | `valve_scale_coefficient`, `valve_nonlinearity_exponent`  |
| `(5, 2)`   | `gas_puff_data.feather`       | `delivered_gas_puffs`                                      |
| `(4, 1)`   | `lick_data.feather`           | `lick_threshold`                                          |
| `(6, 1)`   | `torque_data.feather`         | `torque_per_adc_unit`                                     |
| `(7, 1)`   | `screen_data.feather`         | `screens_initially_on`                                    |

The module registry that drives this roster lives in `mesoscope_vr/microcontrollers.py`; the agnostic discovery,
filename-parsing, and event-partitioning primitives every parser builds on (`find_module_feathers`,
`parse_module_feather_name`, `partition_events`, `get_event_data`, `merge_event_streams`) live in
`cross_system/microcontroller.py` (see `forging:microcontroller-primitives`). The module feathers these jobs parse
are themselves produced upstream by axci (see `ataraxis@communication:log-processing`). A module whose gating
hardware-state field is unset is skipped during processing, so its feather is legitimately absent.

> **Schema note:** Per-module output column names are defined by the parse functions in
> `mesoscope_vr/microcontrollers.py` and may evolve with the firmware. Always confirm the actual schema via
> `query_behavior_data_tool` (or `verify_behavior_processing_output_tool.files[*].columns`) before
> relying on specific column names in downstream analysis. The authoritative per-module schema reference is
> `mesoscope:mesoscope-vr-module-parsing`.

---

## Processing tracker

Two trackers record job lifecycle per session, each written beside the output it covers:
`{session_root}/processed_data/runtime_data/runtime_processing_tracker.yaml` for the runtime job, and
`{session_root}/processed_data/microcontroller_data/microcontroller_processing_tracker.yaml` for the
microcontroller jobs. Each tracker carries only its own pipeline's jobs, so a session's full job roster is the
union of the two. The `verify_behavior_processing_output_tool` `tracker` return exposes these per-job
fields inside its `jobs[]` list:

| Field           | Meaning                                                                             |
|-----------------|-------------------------------------------------------------------------------------|
| `job_id`        | xxHash64 hex digest of `job_name` (or `job_name:specifier` when a specifier is set) |
| `job_name`      | `runtime_processing` or `microcontroller_processing`, per the tracker read          |
| `specifier`     | `1` for runtime, `{cid}-{type}-{id}` for microcontroller                            |
| `status`        | `SCHEDULED`, `RUNNING`, `SUCCEEDED`, or `FAILED`                                    |
| `error_message` | Populated only when `status == FAILED`                                              |

Additional per-job fields — `started_at` and `completed_at` — are persisted to the YAML file but
NOT surfaced through the MCP tool. Inspect the YAML directly if you need the wall-clock timing
data. Alongside `jobs[]`, the tracker dict also returns a `summary` block with aggregate `total`,
`succeeded`, `failed`, `running`, and `scheduled` counts.

Use the tracker to cross-reference which jobs produced which feather files. A missing feather file
with a `SUCCEEDED` tracker entry indicates a silent write failure (rare) — invoke
`forging:behavior-processing` to clean and re-run that session.

---

## Interpretation guidance

### State feathers

`system_state_data` and `runtime_state_data` are transition logs — one row per state change, not a
per-frame dump. Treat each row as a state-open event; the next row is the transition out. The last
row remains open until the end of the session.

```python
# Reconstruct durations for a state feather:
import polars as pl

df = pl.read_ipc("system_state_data.feather", memory_map=True)
durations = df.with_columns(
    end_us=pl.col("time_us").shift(-1)
).with_columns(
    duration_us=pl.col("end_us") - pl.col("time_us"),
)
```

Don't read a gap in any single feather as diagnostic on its own. Every per-event feather can go
quiet for legitimate reasons: state feathers are sparse by design, encoder data stops flowing
while the brake is engaged, lick / valve / gas-puff feathers only record events. Interpret gaps
holistically across the full set of feathers the session produced.

A key property to exploit: the Mesoscope-VR runtime and its microcontrollers *detect their own
interruptions* and log them as explicit state transitions. Emergency pauses, Unity terminations,
and idle transitions all write rows into `system_state_data.feather` and
`runtime_state_data.feather` the moment they fire, so a real acquisition pause is represented as
an idle / paused state row — not as a hole. Disambiguation then reduces to: if a `time_us` gap in
a per-event feather is covered by an idle or paused row in the state feathers, the gap is a real
acquisition interruption; if no state row covers it, the gap is just an absence of events in that
feather.

### Encoder feather

`traveled_distance_cm` is cumulative, not per-step. To recover per-event displacement, compute the
first-difference over `time_us`:

```python
import polars as pl

df = pl.read_ipc("encoder_data.feather", memory_map=True)
df = df.with_columns(step_cm=pl.col("traveled_distance_cm").diff())
```

Sign flips in `step_cm` indicate direction reversals. CCW is positive by convention.

### Trial feathers

`trial_data.feather` is trial-start-indexed; use `traveled_distance_cm` to join against
`encoder_data.feather` when mapping trial onsets to time. A `vr_trigger_zone_data.feather` row is emitted only for
trials that reached their trigger-zone start (the runtime appends a trigger-zone row only when
`trigger_start_absolute <= trial_distances[index]`), so the trigger-zone feather has at most one row per trial. The
two feathers are one-to-one ONLY when every trial reached its trigger zone (no early truncation); a trial truncated
before its trigger-zone start produces a `trial_data` row but NO `vr_trigger_zone_data` row, leaving the feathers at
different lengths. Verify the row counts are equal before zipping the two, or align by `traveled_distance_cm` rather
than a positional zip.

### Camera timestamps

The camera feathers are written into `processed_data/video_data/` by the separate video pipeline (job
`camera_timestamp_extraction`, tracked under the video tracker, not a behavior tracker), freshly extracted from
the raw VideoSystem logs rather than hardlinked from upstream axvs output. Use them to align behavior events with
imaging frames; the `frame_time_us` column is already in UTC microseconds so it can be compared directly to any
`time_us` column in this output.

---

## Error routing

| Symptom                                                   | Diagnosis                                                                    | Action                                                 |
|-----------------------------------------------------------|------------------------------------------------------------------------------|--------------------------------------------------------|
| `verified: false` with `total_files: 0`                   | Behavior processing never ran or was cleaned                                 | Run `forging:behavior-processing`                             |
| `verified: false` with file `error: ...`                  | Corrupt feather output                                                       | Clean session output and re-run                        |
| Tracker has `FAILED` jobs                                 | Job failures during execution                                                | See `forging:behavior-processing` error routing               |
| Tracker has `SUCCEEDED` but file missing                  | Rare: post-write deletion or filesystem eviction                             | Clean and re-run the affected session                  |
| Expected camera feather missing                           | Source not registered in the camera manifest, or no raw `{source_id}_log.npz` archive present; diagnosed against the video tracker, NOT a behavior tracker | Check the camera manifest, raw camera logs, and `video_data/video_processing_tracker.yaml` (see `forging:camera-timestamp-extraction`) |
| Expected module feather missing                           | Module ineligible (gating hardware-state field unset) or unregistered `(module_type, module_id)` | Check the hardware state YAML and the module roster (see `mesoscope:mesoscope-vr-module-parsing`) |
| Runtime experiment feathers missing on experiment session | Missing `experiment_configuration.yaml` or cue decomposition failure         | Fix experiment config or inspect runtime failure       |

---

## Related skills

| Skill                                   | Relationship                                                              |
|-----------------------------------------|--------------------------------------------------------------------------|
| `forging:forging-mcp-environment-setup` | Prerequisite: MCP server connectivity                                    |
| `assets:session-discovery`              | Upstream: session discovery produces confirmed session paths             |
| `forging:behavior-input-format`         | Reference: upstream inputs that became these outputs                      |
| `forging:behavior-processing`           | Upstream: produces the data described here                               |
| `forging:data-processing-design`        | Doctrine: prepare-then-execute, trackers, and worker-budget concurrency  |
| `forging:camera-timestamp-extraction`   | Owner of the separate video pipeline that writes camera feathers into `video_data/` |
| `forging:microcontroller-primitives`    | Owner of the agnostic module-feather discovery and partitioning helpers  |
| `mesoscope:mesoscope-vr-module-parsing` | Owner of per-module column schemas, event codes, and calibration fields  |
| `mesoscope:mesoscope-vr-trial-decomposition` | Owner of the cue-sequence-to-trial decomposition algorithm          |

---

## Verification checklist

```text
Processing Results:
- [ ] runtime_data/ and microcontroller_data/ subdirectories present under every session's processed_data/
- [ ] verify_behavior_processing_output_tool returned verified: true per session
- [ ] Tracker reports SUCCEEDED for every expected (job_name, specifier) pair
- [ ] Runtime state feathers present for all sessions
- [ ] Experiment feathers present for MESOSCOPE_EXPERIMENT sessions only
- [ ] Camera `{name}_timestamps.feather` present for every camera source in the manifest (produced by the separate
      video pipeline into processed_data/video_data/; their job state lives in the video tracker)
- [ ] Microcontroller feathers present for every eligible module key in the roster
- [ ] Sample rows and timing statistics spot-checked via query_behavior_data_tool
- [ ] Any missing files cross-referenced against hardware state / experiment config / upstream outputs
```
