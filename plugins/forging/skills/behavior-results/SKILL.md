---
name: behavior-results
description: >-
  Complete reference for behavior processing output data formats, feather file discovery, output
  verification, and data querying. Use when evaluating behavior processing results, when the user
  asks about runtime state data, trial data, camera timestamps, or microcontroller feather contents,
  or when auditing completed sessions for downstream analysis.
user-invocable: true
---

# Processing results

Complete output data format documentation for the behavior processing pipeline. Covers the
`behavior_data/` output layout, per-job-type feather schemas, verification via MCP tool, data
querying, and interpretation guidance.

---

## Scope

**Covers:**
- Output directory structure and `behavior_data/` layout
- Runtime output schemas (system state, runtime state, guidance, cue, trial, trigger zone)
- Camera output schema (hardlinked from `/video:log-processing` output)
- Microcontroller output schemas (per-module feather contents)
- `verify_behavior_processing_output_tool` usage
- `query_behavior_data_tool` usage and sample-row interpretation
- Interpretation guidance for state feathers, timing feathers, trial feathers

**Does not cover:**
- Input data format (see `/behavior-input-format`)
- Batch processing workflow (see `/behavior-processing`)
- Session discovery (see the assets plugin's `/session-discovery`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)

**Note:** `/video:*` and `/communication:*` refer to the **video** and **communication** plugins
from the [ataraxis marketplace](https://github.com/Sun-Lab-NBB/ataraxis).

---

## Available tools

### Verification tool

| Tool                                         | Purpose                                                             |
|----------------------------------------------|---------------------------------------------------------------------|
| `verify_behavior_processing_output_tool`     | Scans `behavior_data/`, validates feather files, reports tracker    |

**Parameters:**

| Parameter      | Type  | Default    | Description                                                                                                                                                     |
|----------------|-------|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `session_path` | `str` | (required) | Absolute path to the session root directory. The tool loads `SessionData` to resolve `processed_data_path` and inspects `{processed_data_path}/behavior_data/`. |

**Return structure:**

```text
verified:            Boolean — True when all files are readable AND at least one file exists
session_path:        Echo of the input session root
data_path:           Absolute path to the behavior_data/ subdirectory (always under processed_data/)
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

The tool loads the session's :class:`SessionData` marker, walks
`{session.processed_data_path}/behavior_data/` recursively with `rglob("*.feather")`, and loads each
file via the shared `analyze_feather_file` helper with `max_sample_rows=0`, so verification only
costs metadata reads (no full table materialization). The output location is not configurable —
behavior outputs always live under the session's processed data hierarchy.

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
row values or summarize inter-row timing.

### Overview tool (referenced from `/behavior-processing`)

`get_batch_status_overview_tool` is documented in `/behavior-processing`. Use it here when you need a
cross-session summary of tracker state before drilling into any single session's outputs.

---

## Recommended query order

1. **`get_batch_status_overview_tool`** (from `/behavior-processing`) — aggregate status across all
   sessions under a root directory, so you know which sessions have finished processing.
2. **`verify_behavior_processing_output_tool`** — per-session verification against the expected
   feather set and the tracker (takes a session path).
3. **`query_behavior_data_tool`** — drill into specific files for schemas, timing, and sample rows.

---

## Output directory structure

Behavior processing writes all output under a `behavior_data/` subdirectory that always lives under
the session's `processed_data_path`, co-located with the upstream `camera_timestamps/` and
`microcontroller_data/` produced by axvs and axci. This location is static and cannot be overridden:

```text
{session_root}/
└── processed_data/
    └── behavior_data/
        ├── behavior_processing_tracker.yaml          # per-session job tracker
        ├── system_state_data.feather                 # runtime job — always
        ├── runtime_state_data.feather                # runtime job — always
        ├── reinforcing_guidance_state_data.feather   # runtime job — experiments only
        ├── aversive_guidance_state_data.feather      # runtime job — experiments only
        ├── vr_cue_data.feather                       # runtime job — experiments only
        ├── vr_trigger_zone_data.feather              # runtime job — experiments only
        ├── trial_data.feather                        # runtime job — experiments only
        ├── face_camera_timestamps.feather            # camera job — if source 51 present
        ├── body_camera_timestamps.feather            # camera job — if source 62 present
        ├── encoder_data.feather                      # microcontroller (2,1)
        ├── mesoscope_frame_data.feather              # microcontroller (1,1)
        ├── brake_data.feather                        # microcontroller (3,1)
        ├── valve_data.feather                        # microcontroller (5,1)
        ├── gas_puff_data.feather                     # microcontroller (5,2)
        ├── lick_data.feather                         # microcontroller (4,1)
        ├── torque_data.feather                       # microcontroller (6,1)
        └── screen_data.feather                       # microcontroller (7,1)
```

All feather files are uncompressed Arrow IPC, so they are memory-mappable and directly loadable via
`pl.read_ipc(source, memory_map=True)`. Which files are present depends on the session type, the
presence of upstream inputs, and which modules were eligible under the session's hardware state
(see `/behavior-input-format`).

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
`/behavior-processing` error routing.

### Camera feathers

Camera outputs are **hardlinks** to the upstream axvs `camera_{source_id}_timestamps.feather` files.
The schema is whatever axvs produces (typically a `frame_time_us` column), and the file contents are
identical to the upstream feather — no byte duplication on disk.

| Source ID | Output filename                    |
|-----------|------------------------------------|
| 51        | `face_camera_timestamps.feather`   |
| 62        | `body_camera_timestamps.feather`   |

Because the camera feathers are raw axvs output, treat them as an axvs-owned artifact — the schema
evolves with axvs releases, not with sollertia-forgery. Inspect the actual column set via
`query_behavior_data_tool` before relying on specific column names. Hardlinking means that if the
user deletes the upstream axvs feather before behavior processing re-runs, the behavior output
inode remains valid as long as the filesystem hasn't reclaimed the blocks.

### Microcontroller feathers

Each microcontroller job transforms a single axci module feather into a domain-specific output. The
column schemas below reflect the current `_MODULE_REGISTRY` parse functions.

**`encoder_data.feather`** — module `(2, 1)` — converts CCW/CW pulse events to cumulative distance

| Column                 | Dtype     | Description                  |
|------------------------|-----------|------------------------------|
| `time_us`              | `UInt64`  | Event timestamp              |
| `traveled_distance_cm` | `Float64` | Cumulative traveled distance |

CCW rotation is treated as positive displacement, CW as negative. Calibrated via
`hardware_state.cm_per_pulse`.

**`mesoscope_frame_data.feather`** — module `(1, 1)` — Mesoscope TTL frame timestamps

Presence gated by `hardware_state.recorded_mesoscope_ttl == True`. Schema is determined by
`_parse_ttl_data` in `sollertia_forgery.processing.microcontrollers`. Inspect via
`query_behavior_data_tool` if the column list is unknown at query time.

**`brake_data.feather`** — module `(3, 1)` — brake strength events

Calibrated via `hardware_state.maximum_brake_strength` / `minimum_brake_strength`.

**`valve_data.feather`** — module `(5, 1)` — water valve events

Calibrated via `hardware_state.valve_scale_coefficient` and `valve_nonlinearity_exponent`.

**`gas_puff_data.feather`** — module `(5, 2)` — gas puff delivery events

Presence gated by `hardware_state.delivered_gas_puffs == True`.

**`lick_data.feather`** — module `(4, 1)` — lick sensor events

Calibrated via `hardware_state.lick_threshold`.

**`torque_data.feather`** — module `(6, 1)` — torque sensor events

Calibrated via `hardware_state.torque_per_adc_unit`.

**`screen_data.feather`** — module `(7, 1)` — screen on/off state transitions

Initial state defined by `hardware_state.screens_initially_on`.

> **Schema note:** Column names for modules other than the encoder are defined inside their parse
> functions in `sollertia_forgery.processing.microcontrollers` and are not captured in this skill
> because they evolve with the firmware. Always confirm the actual schema via
> `query_behavior_data_tool` (or `verify_behavior_processing_output_tool.files[*].columns`) before
> relying on specific column names in downstream analysis.

---

## Processing tracker

`{session_root}/processed_data/behavior_data/behavior_processing_tracker.yaml` tracks job lifecycle
per session. The `verify_behavior_processing_output_tool` `tracker` return exposes these per-job
fields inside its `jobs[]` list:

| Field           | Meaning                                                                             |
|-----------------|-------------------------------------------------------------------------------------|
| `job_id`        | xxHash64 hex digest of `job_name` (or `job_name:specifier` when a specifier is set) |
| `job_name`      | One of `runtime_processing`, `camera_processing`, `microcontroller_processing`      |
| `specifier`     | Job-specific identifier (source ID or `{cid}-{type}-{id}`)                          |
| `status`        | `SCHEDULED`, `RUNNING`, `SUCCEEDED`, or `FAILED`                                    |
| `error_message` | Populated only when `status == FAILED`                                              |

Additional per-job fields — `started_at` and `completed_at` — are persisted to the YAML file but
NOT surfaced through the MCP tool. Inspect the YAML directly if you need the wall-clock timing
data. Alongside `jobs[]`, the tracker dict also returns a `summary` block with aggregate `total`,
`succeeded`, `failed`, `running`, and `scheduled` counts.

Use the tracker to cross-reference which jobs produced which feather files. A missing feather file
with a `SUCCEEDED` tracker entry indicates a silent write failure (rare) — invoke
`/behavior-processing` to clean and re-run that session.

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
`encoder_data.feather` when mapping trial onsets to time. `vr_trigger_zone_data.feather` is aligned
one-to-one with `trial_data.feather`, so you can zip the two without a key join.

### Camera timestamps

The camera feathers are byte-identical to the upstream axvs outputs. Use them to align behavior
events with imaging frames; the `frame_time_us` column is already in UTC microseconds so it can
be compared directly to any `time_us` column in this output.

---

## Error routing

| Symptom                                                   | Diagnosis                                                                    | Action                                                 |
|-----------------------------------------------------------|------------------------------------------------------------------------------|--------------------------------------------------------|
| `verified: false` with `total_files: 0`                   | Behavior processing never ran or was cleaned                                 | Run `/behavior-processing`                             |
| `verified: false` with file `error: ...`                  | Corrupt feather output                                                       | Clean session output and re-run                        |
| Tracker has `FAILED` jobs                                 | Job failures during execution                                                | See `/behavior-processing` error routing               |
| Tracker has `SUCCEEDED` but file missing                  | Rare: post-write deletion or filesystem eviction                             | Clean and re-run the affected session                  |
| Expected camera feather missing                           | Source ID not in `_CAMERA_OUTPUT_NAMES`, or upstream axvs did not produce it | Check upstream axvs outputs, extend registry if needed |
| Expected module feather missing                           | Module ineligible (hardware state field unset) or unknown `(type, id)`       | Check hardware state YAML and `_MODULE_REGISTRY`       |
| Runtime experiment feathers missing on experiment session | Missing `experiment_configuration.yaml` or cue decomposition failure         | Fix experiment config or inspect runtime failure       |

---

## Related skills

| Skill                                | Relationship                                                 |
|--------------------------------------|--------------------------------------------------------------|
| `/forging-mcp-environment-setup`     | Prerequisite: MCP server connectivity                        |
| assets plugin `/session-discovery`   | Upstream: session discovery produces confirmed session paths |
| `/behavior-input-format`             | Reference: upstream inputs that became these outputs         |
| `/behavior-processing`               | Upstream: produces the data described here                   |

---

## Verification checklist

```text
Processing Results:
- [ ] behavior_data/ subdirectory present under every session's processed_data/
- [ ] verify_behavior_processing_output_tool returned verified: true per session
- [ ] Tracker reports SUCCEEDED for every expected (job_name, specifier) pair
- [ ] Runtime state feathers present for all sessions
- [ ] Experiment feathers present for MESOSCOPE_EXPERIMENT sessions only
- [ ] Camera feathers present for every camera source ID expected
- [ ] Microcontroller feathers present for every eligible module
- [ ] Sample rows and timing statistics spot-checked via query_behavior_data_tool
- [ ] Any missing files cross-referenced against hardware state / experiment config / upstream outputs
```
