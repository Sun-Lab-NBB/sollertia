---
name: behavior-processing
description: >-
  Orchestrates batch behavior processing via the sollertia-forgery MCP server (batch
  preparation, job execution, progress monitoring, cancellation, retry, cleanup). Use when
  processing confirmed session paths through the runtime / microcontroller pipeline
  or managing behavior-processing jobs across sessions.
user-invocable: false
---

# Behavior processing

Orchestrates the batch behavior processing workflow: prepare execution manifests, dispatch jobs for
background execution, monitor progress, and hand off to downstream skills for output verification and
analysis.

---

## Scope

**Covers:**
- Batch preparation from confirmed session paths (output locations resolved statically)
- Background job execution with budget-bounded concurrency
- Progress monitoring (per-job status and timing)
- Cancellation and cleanup
- Failed job reset and retry
- Batch status overview across many sessions
- Tracker regeneration behavior
- Resource allocation and the worker budget model

**Does not cover:**
- The prepare-then-execute and worker-budget doctrine in full (see `forging:data-processing-design`)
- Session discovery and filtering (see `assets:session-discovery`)
- Input file formats or cross-library handoff (see `forging:behavior-input-format`)
- Output verification, schemas, or interpretation (see `forging:behavior-results`)
- The separate camera-timestamp extraction stage (see `forging:camera-timestamp-extraction`)
- The agnostic microcontroller feather primitives (see `forging:microcontroller-primitives`)
- Per-module conversions, event codes, and schemas (see `mesoscope:mesoscope-vr-module-parsing`)
- MCP server connectivity (see `forging:forging-mcp-environment-setup`)
- The internals of the in-process axci microcontroller-log binding (see `ataraxis@communication:log-processing`)

**Handoff rules:** If MCP tools are unavailable, invoke `forging:forging-mcp-environment-setup`. If the user has not
yet run session discovery, invoke `assets:session-discovery` first. After all jobs complete successfully, hand
off to `forging:behavior-results` to verify and analyze outputs.

**Note:** `ataraxis@communication:*` refers to the **communication** plugin
from the [ataraxis marketplace](https://github.com/Sun-Lab-NBB/ataraxis).

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for all processing operations. Do not call the
underlying pipeline entry points (such as the per-job `run_behavior_processing_pipeline`) directly —
those bypass the background execution manager and the progress/timing monitoring surface.

You MUST have confirmed session paths from `assets:session-discovery` before calling
`prepare_behavior_processing_batch_tool`. Do not guess, infer, or discover paths from within this
skill.

Behavior outputs are written to `{session.processed_data_path}/behavior_data/` for every session. This
batch writes only the runtime and microcontroller feathers here; the pipeline resolves this location from
the session marker at dispatch time. Camera timestamp feathers also reside in `behavior_data/`, but they
are written by the separate camera-timestamp extraction stage, not by this batch (see
`forging:camera-timestamp-extraction`).

You MUST respect the single-execution-session constraint: only one batch may run at a time per
`sl-mcp` process. Cancel any active session before starting a new batch.

---

## Available tools

### Preparation and execution

| Tool                                     | Purpose                                                            |
|------------------------------------------|--------------------------------------------------------------------|
| `prepare_behavior_processing_batch_tool` | Creates execution manifest without starting execution (idempotent) |
| `execute_behavior_processing_jobs_tool`  | Dispatches prepared jobs for background execution                  |

**`prepare_behavior_processing_batch_tool` parameters:**

| Parameter         | Type          | Default      | Description                                                                                |
|-------------------|---------------|--------------|--------------------------------------------------------------------------------------------|
| `session_paths`   | `list[str]`   | (required)   | Absolute paths to session root directories (from `assets:session-discovery`) |

Each session's `behavior_data/` subdirectory is created under its
`{session.processed_data_path}/behavior_data/`, containing the processing tracker
(`behavior_processing_tracker.yaml`) and all processed feather files. The tool does NOT accept an
output directory parameter — the location is static.

**`execute_behavior_processing_jobs_tool` parameters:**

| Parameter       | Type         | Default    | Description                                                                                                                                                                                                                            |
|-----------------|--------------|------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `jobs`          | `list[dict]` | (required) | Job descriptors from the prepare manifest. Each must have `session_path`, `tracker_path`, `job_id`, `job_name`, `specifier`. The output location is NOT part of the descriptor — workers resolve it from SessionData at dispatch time. |
| `worker_budget` | `int`        | `-1`       | Total CPU cores for the session; `-1` for automatic resolution via `resolve_worker_count`. Directly controls memory footprint and concurrency.                                                                                         |

### Monitoring

| Tool                                   | Purpose                                                       |
|----------------------------------------|---------------------------------------------------------------|
| `get_behavior_processing_status_tool`  | Per-job status of the active execution session                |
| `get_behavior_processing_timing_tool`  | Per-job timing and session-level throughput                   |

Neither tool takes parameters — both read from the in-memory execution state plus on-disk trackers.

### Lifecycle

| Tool                                    | Purpose                                                           |
|-----------------------------------------|-------------------------------------------------------------------|
| `cancel_behavior_processing_tool`       | Clears pending queue; active jobs complete naturally              |
| `reset_behavior_processing_jobs_tool`   | Resets specific or all jobs in a tracker to `SCHEDULED` for retry |
| `clean_behavior_processing_output_tool` | Deletes `behavior_data/` subdirectories for full re-processing    |
| `get_batch_status_overview_tool`        | Aggregate status across all sessions under a root directory       |

**`reset_behavior_processing_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                                                 |
|----------------|--------------------|------------|-----------------------------------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to `behavior_processing_tracker.yaml`                         |
| `job_ids`      | `list[str] / None` | `None`     | Hexadecimal job IDs to reset; if omitted, every job in the tracker is reset |

**`clean_behavior_processing_output_tool` parameters:**

| Parameter       | Type        | Default    | Description                                                                        |
|-----------------|-------------|------------|------------------------------------------------------------------------------------|
| `session_paths` | `list[str]` | (required) | Absolute paths to session root directories whose behavior output should be cleaned |

Loads each session's SessionData marker to resolve `processed_data_path`, then deletes
`{processed_data_path}/behavior_data/` and all of its contents (feather outputs + tracker YAML).
After cleanup, pass the same session paths back to `prepare_behavior_processing_batch_tool` to
reinitialize from scratch.

**`get_batch_status_overview_tool` parameters:**

| Parameter        | Type  | Default    | Description                                           |
|------------------|-------|------------|-------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to search recursively for tracker YAMLs |

---

## Pipeline architecture

```text
session_data.yaml + raw_data/behavior_data/1_log.npz       → runtime_processing         → *_data.feather

raw_data/behavior_data/{cid}_log.npz --(axci in-process)--> processed_data/microcontroller_data/controller_*_module_*.feather
                                                            → microcontroller_processing → encoder_data.feather, etc.
```

Key architectural facts:

- **Job types:** exactly two — `runtime_processing` and `microcontroller_processing`. There is no
  `camera_processing` behavior job; camera-timestamp extraction is a separate stage (see
  `forging:camera-timestamp-extraction`).
- **Job specifiers:**
  - `runtime_processing` → always the fixed source ID `"1"` (exactly one per session)
  - `microcontroller_processing` → `"{controller_id}-{module_type}-{module_id}"`
- **In-process microcontroller extraction:** at execute time the behavior worker extracts the raw
  microcontroller logs in-process via the axci log-processing binding (`_extract_microcontroller_logs`),
  reading raw `{controller_id}_log.npz` archives from `raw_data/behavior_data/` (driven by the session's
  `extraction_configuration.yaml`) and writing the per-module feathers into
  `processed_data/microcontroller_data/` itself. No separate axci run is required — axci is a hard
  in-process dependency, not a prerequisite job. See `ataraxis@communication:log-processing` for the
  binding's internals.
- **Prepare-time asymmetry:** prepare-time discovery (`discover_behavior_jobs`) only globs for
  already-present `controller_*_module_*.feather` files and does NOT extract. A first prepare on a
  never-processed session may therefore find zero microcontroller jobs until an execute (or a CLI run)
  has produced the module feathers.
- **Tracker filename:** `behavior_processing_tracker.yaml`, written under
  `{session.processed_data_path}/behavior_data/`. The path is static — the caller does not choose it.
- **ProcessingTracker lifecycle:** `SCHEDULED` → `RUNNING` → `SUCCEEDED` / `FAILED`, persisted as YAML.
- **Single execution session constraint:** one batch per `sl-mcp` process. Cancel before starting another.
- **Remote execution mode:** each worker subprocess runs `run_behavior_processing_pipeline(job_id=...)`,
  which executes exactly one `(job_name, specifier)` pair without re-running all jobs for the session.
- **Tracker regeneration:** foreign or stale tracker entries are detected at prepare/execute time,
  surfaced as warnings, and reset before execution. A tracker built from an older session layout
  will be rebuilt automatically on the next prepare call.
- **Reserved cores:** 2 cores are reserved system-wide (`RESERVED_CORES = 2`); the worker budget
  applies to the remaining cores.
- **Output layout:** every session writes its runtime and microcontroller behavior outputs to
  `{session_root}/processed_data/behavior_data/` so that downstream tooling can locate them
  deterministically from the session root. The separate camera-timestamp extraction stage also lands its
  `{name}_timestamps.feather` outputs in `behavior_data/`, but that stage runs independently with its own
  `camera_timestamps/camera_processing_tracker.yaml`, not under this batch.

---

## Processing workflow

### Execution model

The workflow uses the **prepare-then-execute** batch model shared by every batch slf pipeline:
prepare initializes trackers and builds job descriptors idempotently, then execute dispatches jobs
into a budget-bounded process pool with a single active execution session at a time. This skill does
not re-explain that doctrine — see `forging:data-processing-design` for the full prepare-then-execute,
job-identity, tracker, and worker-budget contract, and for the cross-stage seams between slf-owned and
delegated dependency stages.

### Pre-processing checklist

```text
- [ ] Confirmed session paths from assets:session-discovery
- [ ] extraction_configuration.yaml present in raw_data/behavior_data/ (drives in-process microcontroller extraction)
- [ ] Raw controller {controller_id}_log.npz archives present in raw_data/behavior_data/
- [ ] Runtime 1_log.npz archive present in raw_data/behavior_data/
- [ ] Hardware state YAML present in raw_data/ for every session in the batch
- [ ] Experiment configuration YAML present for MESOSCOPE_EXPERIMENT sessions
- [ ] Worker budget decision made with user (default -1 for auto-resolution)
- [ ] No other execution session currently active
```

**STOP**: If any checkbox is incomplete, do not proceed. Complete the missing steps first. See
`forging:behavior-input-format` for per-file details.

### Workflow steps

1. **Receive confirmed inputs** — Get the `session_paths` list from `assets:session-discovery`. Do
   not re-derive.

2. **Prepare batch** — Call `prepare_behavior_processing_batch_tool` with the confirmed session
   paths list. Inspect the result:
   - Top-level `success: true` and a populated `sessions` map → continue.
   - Per-session `error: "Discovery failed: ..."` → session-specific problem (likely missing
     hardware state or malformed inputs). Surface to user; continue with remaining sessions.
   - Per-session `error: "No processable behavior jobs discovered..."` → no runtime archive was found,
     or prepare-time discovery found zero microcontroller module feathers. Because prepare only globs for
     already-present `controller_*_module_*.feather` files (it does not extract), a never-processed
     session may report no microcontroller jobs until an execute (or CLI run) has produced them. Confirm
     the runtime `1_log.npz` archive and the `extraction_configuration.yaml` and raw
     `{controller_id}_log.npz` archives are present under `raw_data/behavior_data/`, then re-prepare.
   - `invalid_paths` list populated → session paths that did not exist. Surface to user.

3. **Present discovered jobs** — For each session in the manifest, show the job count broken down
   by type (runtime / microcontroller). Format suggestion:

   ```text
   **Batch Preparation** — 3 sessions, 18 jobs

   | Session                       | Runtime | MCU | Total |
   |-------------------------------|---------|-----|-------|
   | animal_001_2026-03-04_lick_01  | 1       | 5   | 6     |
   | animal_001_2026-03-05_run_01   | 1       | 5   | 6     |
   | animal_002_2026-03-06_exp_01   | 1       | 5   | 6     |
   ```

4. **Confirm resource allocation** — Present the default worker budget (`-1` = auto-resolve using
   `resolve_worker_count` with `reserved_cores=2`). Explain that the budget bounds:
   - Total memory footprint (each worker is a separate process)
   - Maximum number of concurrent jobs
   Ask whether the user wants to override the default. Reduce to limit memory footprint on
   constrained systems.

5. **Flatten jobs** — Collect every job descriptor from `sessions[*].jobs` into a single flat
   `list[dict]`. Every descriptor must contain `session_path`, `tracker_path`, `job_id`, `job_name`,
   and `specifier` — the tool returns these already; do not modify keys. There is no
   `output_directory` key; the worker resolves the output path from SessionData at dispatch time.

6. **Execute jobs** — Call `execute_behavior_processing_jobs_tool` with the flat job list and the
   confirmed `worker_budget`. Check the result:
   - `started: true` → continue to monitoring.
   - `error: "An execution session is already active..."` → previous run is still running; either
     wait or cancel.
   - `invalid_jobs` list populated → job descriptors with missing keys or missing tracker files;
     surface to user.

7. **Monitor progress** — Call `get_behavior_processing_status_tool` periodically until the session
   completes. Optionally call `get_behavior_processing_timing_tool` for elapsed time and throughput.
   Present status as a formatted table (see "Status formatting" below).

8. **Handle completion** — When `active: false` and all jobs are in terminal states:
   - All `SUCCEEDED` → hand off to `forging:behavior-results`.
   - Some `FAILED` → see "Error routing" and offer reset/retry.
   - Mix of `SUCCEEDED` and `UNKNOWN` → tracker corruption; recommend clean + re-prepare.

---

## Resource management

The execution tool uses **budget-based worker allocation** via a single `worker_budget` parameter.
Behavior jobs are generally lightweight (microcontroller parsers partition a single feather into
in-memory arrays; runtime processing streams one NPZ archive), so the budget primarily controls how many
jobs run in parallel rather than tuning per-job parallelism. Microcontroller jobs additionally incur the
in-process axci extraction cost if extraction has not already populated the module feathers. Behavior
processing uses the plain `RESERVED_CORES` plus `max_parallel_jobs`
variant of the worker-budget doctrine — see `forging:data-processing-design` for the full concurrency
contract and how this variant differs from the per-job-overhead and saturating-allocation variants.

- `worker_budget=-1` → `resolve_worker_count` picks the available cores after subtracting the reserved
  cores (`RESERVED_CORES = 2`).
- Each pending job is dispatched to a free worker as the budget allows. The dispatch order preserves
  the (runtime, microcontroller) ordering produced by discovery.
- Reduce `worker_budget` to cap memory on constrained hosts. A budget of `1` effectively runs the
  batch sequentially.

Because each job executes exactly one `(job_name, specifier)` pair in a fresh subprocess, there is no
shared state between workers — you cannot reduce memory by running multiple jobs in a single worker.

---

## Status formatting

When presenting batch status, format as a table:

```text
**Behavior Processing Status**

Summary: 14/18 jobs complete | 2 running | 2 queued | 0 failed

| Session                       | Job                             | Status    | Duration |
|-------------------------------|---------------------------------|-----------|----------|
| animal_001_2026-03-04_lick_01  | runtime_processing:1            | SUCCEEDED | 12.5s    |
| animal_001_2026-03-04_lick_01  | microcontroller_processing:101-2-1 | SUCCEEDED | 3.1s     |
| animal_001_2026-03-05_run_01   | runtime_processing:1            | RUNNING   | 4.8s     |
| animal_002_2026-03-06_exp_01   | runtime_processing:1            | SCHEDULED | --       |
```

For multi-session overview via `get_batch_status_overview_tool`:

```text
**Batch Overview** — /data/projects/my_project

| Session root                                         | Status    | Succeeded | Failed | Running | Scheduled |
|------------------------------------------------------|-----------|-----------|--------|---------|-----------|
| /data/projects/my_project/animal_001/2026-03-04_lick_01 | completed | 6         | 0      | 0       | 0         |
| /data/projects/my_project/animal_001/2026-03-05_run_01  | running   | 3         | 0      | 2       | 1         |
| /data/projects/my_project/animal_002/2026-03-06_exp_01  | scheduled | 0         | 0      | 0       | 6         |
```

---

## Re-running failed jobs

1. Identify failed jobs from `get_behavior_processing_status_tool` output (inspect `error_message`).
2. Call `reset_behavior_processing_jobs_tool` with the tracker path and the failed `job_ids`.
3. Re-prepare the batch to pick up the reset jobs (prepare is idempotent and returns the current
   tracker state).
4. Call `execute_behavior_processing_jobs_tool` again with the refreshed job descriptors.

To re-process an entire session from scratch, call `clean_behavior_processing_output_tool` to delete
the session's `behavior_data/` subdirectory, then re-prepare and re-execute. This is the right move
when a tracker is corrupt or when the user wants to change output file contents (e.g., after updating
a module parser in the Mesoscope-VR module registry, gated by `mesoscope:mesoscope-vr-module-parsing`).

---

## Error routing

### Preparation errors

| Error                                                       | Resolution                                                           |
|-------------------------------------------------------------|----------------------------------------------------------------------|
| `Discovery failed: ...`                                     | Session metadata or hardware state is malformed; inspect the session |
| `No processable behavior jobs discovered for this session.` | No runtime archive found, or prepare found no `controller_*_module_*.feather` yet (extraction has not run), or session type not eligible |
| `invalid_paths: [...]`                                      | One or more session paths do not exist on disk                       |

### Execution errors

| Error                                      | Resolution                                                              |
|--------------------------------------------|-------------------------------------------------------------------------|
| `An execution session is already active.`  | Wait for the current session or `cancel_behavior_processing_tool` first |
| `No valid jobs to execute.`                | Verify job descriptors have all required keys                           |
| `Tracker file not found: ...`              | Re-prepare the batch to regenerate trackers                             |
| Jobs stuck in `RUNNING` after manager died | Manager thread crashed; cancel + clean + re-prepare                     |

### Per-job failure routing

| Error pattern                                        | Action                                                               |
|------------------------------------------------------|----------------------------------------------------------------------|
| Runtime archive not found / file read errors         | Verify `1_log.npz` exists under `raw_data/behavior_data/`            |
| Extraction configuration not found                   | Ensure `extraction_configuration.yaml` present in `raw_data/behavior_data/` |
| Hardware state YAML not found                        | Ensure `*hardware_state*.yaml` present in `raw_data/`                |
| Experiment configuration YAML not found (experiment) | Ensure `*experiment_configuration*.yaml` present in `raw_data/`      |
| Module `(type, id)` does not match registry          | Unsupported module — see `mesoscope:mesoscope-vr-module-parsing`     |
| Cue sequence decomposition failure                   | Malformed cue sequence in runtime archive; inspect payload           |
| Polars / Arrow read errors on module feather         | Corrupt feather from in-process axci extraction; clean + re-prepare (see `ataraxis@communication:log-processing` for binding internals) |
| MCP tools unavailable                                | Invoke `forging:forging-mcp-environment-setup`                       |
| Out of memory                                        | Reduce `worker_budget`                                               |
| Corrupt tracker                                      | `clean_behavior_processing_output_tool` → re-prepare                 |

---

## Related skills

| Skill                                     | Relationship                                                      |
|-------------------------------------------|------------------------------------------------------------------|
| `forging:forging-mcp-environment-setup`   | Prerequisite: MCP server connectivity                            |
| `forging:data-processing-design`          | Doctrine: the prepare-then-execute and worker-budget pattern     |
| `assets:session-discovery`                | Upstream: session discovery and filtering                        |
| `forging:behavior-input-format`           | Reference: input file layout and cross-library handoff           |
| `forging:behavior-results`                | Downstream: output discovery, verification, and interpretation   |
| `forging:camera-timestamp-extraction`     | Separate stage: the manifest-driven camera-timestamp extraction  |
| `forging:microcontroller-primitives`      | Stage: the agnostic microcontroller feather-parsing primitives   |
| `mesoscope:mesoscope-vr-module-parsing`   | Specialization: per-module conversions, event codes, and schemas |
| `forging:project-manifest`                | Downstream: regenerate manifest to update behavior status        |
| `ataraxis@communication:log-processing`   | Internals of the in-process axci microcontroller-log binding     |

---

## Verification checklist

```text
Behavior Processing Workflow:
- [ ] Verified MCP server connectivity (invoked forging:forging-mcp-environment-setup if unavailable)
- [ ] Received confirmed session paths from assets:session-discovery
- [ ] Prepared batch via prepare_behavior_processing_batch_tool
- [ ] Presented discovered job counts per session and per type
- [ ] Confirmed worker budget with user
- [ ] Verified no active execution session before dispatch
- [ ] Executed jobs via execute_behavior_processing_jobs_tool
- [ ] Monitored status until all jobs reached terminal state
- [ ] Investigated and retried failed jobs if needed (reset or clean + re-prepare)
- [ ] Handed off successful output to forging:behavior-results
```
