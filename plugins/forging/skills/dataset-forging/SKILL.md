---
name: dataset-forging
description: >-
  Orchestrates batch dataset forging (per-session `data.feather` assembly) via the
  sollertia-forgery MCP server (dataset resolution, batch prep, execution, progress, cancel,
  retry, cleanup). Use when assembling analysis-ready feathers from processed behavior and
  cindra outputs or managing forging jobs across a dataset.
user-invocable: false
---

# Dataset forging

Orchestrates the batch dataset-forging workflow: resolve the dataset hierarchy, prepare
execution manifests, dispatch jobs for background execution, monitor progress, and hand
off to downstream skills for output verification and querying.

---

## Scope

**Covers:**
- Dataset resolution semantics (create / load / recreate) and `force_recreate` handling
- Batch preparation, background job execution, and the worker-budget model
- Progress monitoring, cancellation, cleanup, failed-job reset, and cross-dataset overview

**Does not cover:**
- Session discovery and filtering (see the assets plugin's `/session-discovery`)
- Upstream input file formats and cross-library handoff (see `/dataset-forging-input-format`)
- Output verification, schemas, or interpretation (see `/dataset-forging-results`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)
- Upstream behavior processing (see `/behavior-processing`)
- Upstream cindra processing (see `/cindra:single-recording-processing` and
  `/cindra:multi-recording-processing`)

**Handoff rules:** If MCP tools are unavailable, invoke `/forging-mcp-environment-setup`.
If the user has not yet run session discovery, invoke the assets plugin's `/session-discovery` first. After
all jobs complete successfully, hand off to `/dataset-forging-results` to verify and
query outputs.

**Note:** `/cindra:*` refers to the **cindra** plugin from the
[cindra marketplace](https://github.com/Sun-Lab-NBB/cindra).

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for all forging operations. Do not import
`sollertia_forgery.forging.pipeline` directly or invoke the `sl-forge` CLI — those
bypass the background execution manager and the progress/timing monitoring surface.

You MUST have confirmed session names and a project root from the assets plugin's `/session-discovery`
before calling `prepare_forging_batch_tool`. Do not guess, infer, or discover paths from
within this skill.

You MUST use only `MESOSCOPE_EXPERIMENT` sessions. Forging refuses any dataset whose
first resolved session has a different session type. Lick-training and run-training
sessions cannot be forged even though they produce behavior feathers. Eligibility is
enforced by `_create_dataset` inside the pipeline.

You MUST respect the single-execution-session constraint: only one batch may run at a
time per `sl-mcp` process. Cancel any active session before starting a new batch.

Per-session forged output is written to `{session_root}/data.feather` — directly under
the session's own directory, NOT under `processed_data/`. The session's
`session_descriptor.yaml` is also copied from `raw_data/` into the session root
alongside the feather so the forged session carries experimenter context without
reaching back into the raw data. The dataset-level metadata (`dataset.yaml`) and
tracker (`forging_tracker.yaml`) live at `{project_root}/{dataset_name}/`, and each
animal's `surgery_metadata.yaml` is copied once to `{project_root}/{dataset_name}/{animal}/`.

---

## Available tools

### Preparation and execution

| Tool                          | Purpose                                                                   |
|-------------------------------|---------------------------------------------------------------------------|
| `prepare_forging_batch_tool`  | Resolves datasets, initializes trackers, returns a job manifest (idempotent) |
| `execute_forging_jobs_tool`   | Dispatches prepared jobs for background execution                          |

**`prepare_forging_batch_tool` parameters:**

| Parameter  | Type              | Default    | Description                               |
|------------|-------------------|------------|-------------------------------------------|
| `datasets` | `list[dict[str, Any]]` | (required) | Dataset specifications (schema below) |

Each dataset specification dictionary has these keys:

| Key              | Type        | Required | Description                                                                           |
|------------------|-------------|----------|---------------------------------------------------------------------------------------|
| `name`           | `str`       | yes      | The unique dataset name. Becomes the directory name under `project_root`.             |
| `project_root`   | `str`       | yes      | Absolute path to the project root containing animal and session data directories.     |
| `session_names`  | `list[str]` | no       | Session names to include. Omit or pass an empty list to work with an existing dataset.|
| `force_recreate` | `bool`      | no       | Defaults to False. Set True to delete and rebuild the dataset when session sets diverge. |

The tool is idempotent: if the dataset already exists and the provided session set
matches (or is empty), the tracker is reused without reinitialization. A mismatched
session set without `force_recreate=True` is reported as an `invalid_datasets` entry.

Return shape:

```text
success:            Always True (per-dataset errors are reported in invalid_datasets)
datasets:           Map keyed by dataset name. Each value has:
  dataset_path:     Absolute path to {project_root}/{dataset_name}/
  tracker_path:     Absolute path to {dataset_path}/forging_tracker.yaml
  dataset_name:     Echo of the input name
  project_root:     Echo of the input project_root
  jobs[]:           Enriched job descriptors (fields: job_id, job_name, specifier,
                    status, session_name, dataset_name, project_root, tracker_path,
                    optional error_message)
  summary:          Counts for total / succeeded / failed / running / scheduled
total_datasets:     Count of successfully resolved datasets
total_jobs:         Flattened job count across all datasets
invalid_datasets[]: Present only when one or more dataset specs failed to resolve
```

**`execute_forging_jobs_tool` parameters:**

| Parameter       | Type               | Default    | Description                                                                                         |
|-----------------|--------------------|------------|-----------------------------------------------------------------------------------------------------|
| `jobs`          | `list[dict[str,str]]` | (required) | Flattened job descriptors from the prepare manifest.                                               |
| `worker_budget` | `int`              | `-1`       | Total CPU cores for the session; `-1` auto-resolves via `resolve_worker_count` (`RESERVED_CORES=2`). |

Each job descriptor must contain `tracker_path`, `job_id`, `dataset_name`,
`project_root`, and `session_name`. These keys come directly from the prepare manifest
— do not modify them. The worker resolves the per-session output path
(`{session_root}/data.feather`) internally from the dataset metadata.

Return shape:

```text
started:        True if the background manager was started
total_jobs:     Number of pending jobs actually dispatched
worker_budget:  The resolved worker budget (after -1 auto-resolution)
invalid_jobs:   Present only when one or more descriptors were rejected
error:          Present only when no session could start
                  - "An execution session is already active..." → cancel or wait
                  - "No valid jobs to execute." → every descriptor was invalid
```

### Monitoring

| Tool                       | Purpose                                                |
|----------------------------|--------------------------------------------------------|
| `get_forging_status_tool`  | Per-job status of the active execution session         |
| `get_forging_timing_tool`  | Per-job timing and session-level throughput            |

Neither tool takes parameters. Both read in-memory execution state plus the on-disk
trackers. When no session has ever been started, they return `{active: False, message: ...}`.

`get_forging_timing_tool` reports `started_at` / `completed_at` microsecond timestamps,
live `elapsed_seconds` for running jobs, and `duration_seconds` for completed jobs.
When at least one job has finished, `session.throughput_jobs_per_hour` is populated
from the earliest start time.

### Lifecycle

| Tool                                     | Purpose                                                                     |
|------------------------------------------|-----------------------------------------------------------------------------|
| `cancel_forging_tool`                    | Clears the pending queue; active jobs complete naturally                    |
| `reset_forging_jobs_tool`                | Resets specific or all jobs in a tracker to SCHEDULED for retry             |
| `clean_forging_output_tool`              | Deletes dataset hierarchies (tracker, metadata, per-session feathers)       |
| `get_forging_batch_status_overview_tool` | Aggregate status across every dataset discovered under a project root       |

**`reset_forging_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                                            |
|----------------|--------------------|------------|------------------------------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to the dataset's `forging_tracker.yaml`                  |
| `job_ids`      | `list[str] \| None` | `None`     | Hexadecimal job IDs to reset; if omitted, every job in the tracker is reset |

Returns `{reset: True, jobs_reset: N, jobs: [...], summary: {...}}` on success or
`{error: "..."}` when the tracker path is missing or unreadable.

**`clean_forging_output_tool` parameters:**

| Parameter       | Type        | Default    | Description                                                               |
|-----------------|-------------|------------|---------------------------------------------------------------------------|
| `dataset_paths` | `list[str]` | (required) | Absolute paths to `{project_root}/{dataset_name}/` directories to delete |

Refuses to run while an execution session is active — cancel first. Deletes the full
dataset directory tree (tracker + dataset metadata + per-animal `surgery_metadata.yaml`
copies). Per-session `data.feather` and the copied `session_descriptor.yaml` are
NOT removed, because those files live under the session root, not the dataset
directory. After cleanup, pass the same dataset specs back to
`prepare_forging_batch_tool` to reinitialize from scratch.

**`get_forging_batch_status_overview_tool` parameters:**

| Parameter        | Type  | Default    | Description                                                   |
|------------------|-------|------------|---------------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to search recursively for `forging_tracker.yaml` files |

Walks the root with `rglob("forging_tracker.yaml")`. Each tracker's parent directory
is the dataset root and its name is the dataset name. Returns per-dataset summaries
plus cross-dataset aggregates.

**Cancellation behavior:** `cancel_forging_tool` drains the pending queue atomically
and marks the state canceled. Active workers complete their current session normally.
The call is idempotent. The payload reports `cleared_count` (pending jobs removed) and
`active_jobs_at_cancel` (workers still running at cancel time). Use
`get_forging_status_tool` afterward to watch the last active jobs finish.

---

## Pipeline architecture

```text
project_root/
├── animal_A/
│   ├── session_1/
│   │   ├── raw_data/
│   │   │   ├── hardware_state.yaml
│   │   │   ├── experiment_configuration.yaml
│   │   │   ├── session_descriptor.yaml
│   │   │   ├── surgery_metadata.yaml
│   │   │   └── ...
│   │   ├── processed_data/
│   │   │   ├── behavior_data/          ← from /behavior-processing
│   │   │   └── mesoscope_data/
│   │   │       ├── <single-recording>/ ← from /cindra:single-recording-processing
│   │   │       └── multiday/{dataset_name}/  ← from /cindra:multi-recording-processing
│   │   ├── data.feather                ← FORGED OUTPUT (this pipeline)
│   │   └── session_descriptor.yaml  ← FORGED COPY (this pipeline)
│   └── session_2/...
└── {dataset_name}/                     ← FORGED DATASET HIERARCHY (this pipeline)
    ├── dataset.yaml
    ├── forging_tracker.yaml
    └── animal_A/
        └── surgery_metadata.yaml           ← FORGED COPY (one per animal)
```

Key architectural facts:

- **Job name:** `session_data_assembly`. Exactly one job per session in the dataset.
- **Job specifier:** the session name.
- **Tracker filename:** `forging_tracker.yaml`, written to
  `{project_root}/{dataset_name}/`.
- **ProcessingTracker lifecycle:** `SCHEDULED` → `RUNNING` → `SUCCEEDED` / `FAILED`,
  persisted as YAML.
- **Single execution session constraint:** one batch per `sl-mcp` process. Cancel
  before starting another.
- **Remote execution mode:** each worker subprocess runs
  `run_forging_pipeline(name=..., session_names=(), project_root=..., job_id=...)` so
  that only the single session identified by `job_id` is assembled.
- **Output layout:** per-session `{session_root}/data.feather` plus a copy of
  `session_descriptor.yaml` from `raw_data/`; the dataset hierarchy at
  `{project_root}/{dataset_name}/` stores `dataset.yaml`, `forging_tracker.yaml`,
  and one `{animal}/surgery_metadata.yaml` copy per animal (taken from each animal's
  most recent session at dataset creation time).
- **Reserved cores:** two cores are reserved system-wide (`RESERVED_CORES = 2`); the
  worker budget applies to the remaining cores. Each worker is a separate process.
- **Session eligibility:** only `MESOSCOPE_EXPERIMENT` sessions — the pipeline raises
  at dataset creation time if the first resolved session has a different type. Every
  subsequent session in the batch must share the first session's `session_type` and
  `acquisition_system`; a mismatch raises during dataset creation.
- **Surgery data requirement:** every animal represented in the dataset must carry a
  `surgery_metadata.yaml` under the most recent session's `raw_data/`. The file is copied
  once per animal into the dataset hierarchy; a missing file raises during dataset
  creation.
- **Experiment descriptor requirement:** every session must carry an
  `session_descriptor.yaml` under `raw_data/`. The file is copied alongside
  `data.feather` at the end of each session assembly; a missing file raises at
  assembly time before any computation is performed.

---

## Dataset resolution semantics

`prepare_forging_batch_tool` delegates to `resolve_dataset` under the hood. The
resolution rules are:

| Directory state                  | `session_names` provided | `force_recreate` | Outcome                                               |
|----------------------------------|--------------------------|------------------|-------------------------------------------------------|
| Dataset does not exist           | non-empty                | any              | Creates a fresh dataset from the provided sessions    |
| Dataset does not exist           | empty / omitted          | any              | `invalid_datasets` entry (nothing to create from)     |
| Dataset exists                   | omitted / empty          | any              | Loads existing dataset; no verification               |
| Dataset exists, sessions match   | non-empty, matches       | any              | Loads existing dataset; tracker reused                |
| Dataset exists, sessions differ  | non-empty, diverges      | `False`          | `invalid_datasets` entry — caller must decide         |
| Dataset exists, sessions differ  | non-empty, diverges      | `True`           | Deletes the hierarchy, recreates from provided sessions |

Use `force_recreate=True` whenever you extend, shrink, or modify the session set of an
existing dataset. The existing tracker, dataset metadata, and per-animal surgery
copies are discarded, but the per-session `{session_root}/data.feather` and
`{session_root}/session_descriptor.yaml` files already on disk are not touched
(only the dataset hierarchy under `{project_root}/{dataset_name}/` is removed). To
also wipe per-session output, overwrite it naturally on the next run or remove each
`data.feather` manually.

---

## Processing workflow

The workflow is **prepare-then-execute**: `prepare_forging_batch_tool` resolves dataset
hierarchies and builds job descriptors without running any computation (idempotent on
repeat calls with the same session set); `execute_forging_jobs_tool` spawns a background
manager that dispatches jobs into a `ProcessPoolExecutor` bounded by the worker budget.
Only one execution session can be active at a time.

### Pre-processing checklist

```text
- [ ] Confirmed session names and project_root from the assets plugin's /session-discovery
- [ ] Every session in the batch is MESOSCOPE_EXPERIMENT
- [ ] /behavior-processing has completed for every session (behavior feathers present)
- [ ] /cindra:single-recording-processing has completed for every session
- [ ] /cindra:multi-recording-processing has completed with the SAME dataset name
- [ ] hardware_state.yaml and experiment_configuration.yaml present under raw_data/
- [ ] Worker budget decision made with user (default -1 for auto-resolution)
- [ ] No other forging execution session currently active
```

**STOP**: If any checkbox is incomplete, do not proceed. Complete the missing steps
first. See `/dataset-forging-input-format` for per-file details on upstream prerequisites.

### Workflow steps

1. **Receive confirmed inputs** — Get session names, project root, and the target
   dataset name(s) from the user. Session names come from the assets plugin's `/session-discovery`.

2. **Prepare batch** — Call `prepare_forging_batch_tool` with the list of dataset
   specifications. Inspect the result:
   - Top-level `success: true` and a populated `datasets` map → continue.
   - `invalid_datasets[].error == "Missing required 'name' or 'project_root' key."`
     → spec is malformed; fix and retry.
   - `invalid_datasets[].error` starting with `"Unable to use the existing ... dataset."`
     → session set diverges; ask the user whether to set `force_recreate=True`.
   - `invalid_datasets[].error` mentioning session-type → session is not
     `MESOSCOPE_EXPERIMENT`; remove it from the batch.
   - `invalid_datasets[].error` starting with `"Unable to resolve the directory for session"`
     → session name does not resolve under `project_root`; cross-check with
     the assets plugin's `/session-discovery`.

3. **Present discovered jobs** — For each dataset in the manifest, show the session
   count and any pre-existing SUCCEEDED / FAILED counts. Format suggestion:

   ```text
   **Batch Preparation** — 2 datasets, 14 jobs

   | Dataset               | Sessions | Succeeded | Failed | Scheduled |
   |-----------------------|----------|-----------|--------|-----------|
   | animal_001_week1       | 7        | 0         | 0      | 7         |
   | animal_002_week1       | 7        | 2         | 1      | 4         |
   ```

4. **Confirm resource allocation** — Present the default worker budget (`-1` =
   auto-resolve using `resolve_worker_count` with `reserved_cores=2`). Explain that the
   budget bounds:
   - Total memory footprint (each worker is a separate process)
   - Maximum number of concurrent jobs
   Ask whether the user wants to override the default. Reduce to limit memory footprint
   — assembly loads fluorescence arrays and behavior feathers fully in memory per
   worker.

5. **Flatten jobs** — Collect every descriptor from `datasets[*].jobs` into a single
   flat `list[dict]`. Each descriptor already contains `tracker_path`, `job_id`,
   `dataset_name`, `project_root`, and `session_name` — do not modify or add keys.

6. **Execute jobs** — Call `execute_forging_jobs_tool` with the flat job list and the
   confirmed `worker_budget`. Inspect the result:
   - `started: true` → continue to monitoring.
   - `error: "An execution session is already active..."` → previous run is still
     running; either wait or cancel.
   - `invalid_jobs` list populated → job descriptors with missing keys or missing
     tracker files; surface to user.

7. **Monitor progress** — Call `get_forging_status_tool` periodically until the session
   completes. Optionally call `get_forging_timing_tool` for elapsed time and throughput.
   Present status as a formatted table (see "Status formatting" below).

8. **Handle completion** — When `active: false` and all jobs are in terminal states:
   - All `SUCCEEDED` → hand off to `/dataset-forging-results`.
   - Some `FAILED` → see "Error routing" and offer reset/retry.
   - Mix of `SUCCEEDED` and `UNKNOWN` → tracker corruption; recommend clean +
     re-prepare.

---

## Resource management

The execution tool uses budget-based worker allocation via a single `worker_budget`
parameter. Forging jobs are moderately memory-heavy — each worker memory-maps cindra
fluorescence arrays plus the session's behavior feathers, concatenates them, and writes
one `data.feather`. The budget therefore controls both concurrency and memory footprint.

- `worker_budget=-1` → `resolve_worker_count` picks available cores minus
  `RESERVED_CORES=2`.
- Jobs dispatch in the order `prepare_forging_batch_tool` returns them (per-dataset,
  then per-session within the dataset).
- Reduce the budget for large multi-day datasets (many ROIs or frames). A budget of `1`
  runs the batch sequentially.

Each job executes one `(session_data_assembly, session)` pair in a fresh subprocess, so
workers share no state.

---

## Status formatting

When presenting per-session status:

```text
**Forging Status** — dataset `animal_001_week1`

Summary: 5/7 jobs complete | 1 running | 1 queued | 0 failed

| Session                       | Status    | Duration |
|-------------------------------|-----------|----------|
| animal_001_2026-03-04_exp_01   | SUCCEEDED | 42.7s    |
| animal_001_2026-03-05_exp_01   | SUCCEEDED | 39.1s    |
| animal_001_2026-03-06_exp_01   | RUNNING   | 12.4s    |
| animal_001_2026-03-07_exp_01   | SCHEDULED | --       |
```

For multi-dataset overview via `get_forging_batch_status_overview_tool`:

```text
**Forging Overview** — /data/projects/my_project

| Dataset          | Status    | Succeeded | Failed | Running | Scheduled |
|------------------|-----------|-----------|--------|---------|-----------|
| animal_001_week1  | completed | 7         | 0      | 0       | 0         |
| animal_002_week1  | running   | 3         | 1      | 2       | 1         |
| animal_003_week1  | scheduled | 0         | 0      | 0       | 7         |
```

---

## Re-running failed jobs

1. Identify failed jobs from `get_forging_status_tool` output (inspect `error_message`).
2. Call `reset_forging_jobs_tool` with the dataset's `tracker_path` and the failed
   `job_ids` list.
3. Re-prepare the batch (idempotent) to pick up the reset jobs.
4. Call `execute_forging_jobs_tool` again with the refreshed job descriptors.

To rebuild a dataset from scratch (for example after changing the session set):
1. Call `clean_forging_output_tool` with `[{project_root}/{dataset_name}]` to delete
   the dataset hierarchy.
2. Call `prepare_forging_batch_tool` with the new session list. Because the directory
   no longer exists, the dataset is created fresh.

To rebuild only the session set of an existing dataset without deleting first:
1. Call `prepare_forging_batch_tool` with the new `session_names` and
   `force_recreate=True`. The hierarchy is deleted and recreated in-place.

---

## Error routing

### Preparation errors (`invalid_datasets[].error`)

| Error pattern                                                            | Resolution                                                   |
|--------------------------------------------------------------------------|--------------------------------------------------------------|
| `Missing required 'name' or 'project_root' key.`                         | Fix the dataset spec                                         |
| Validation error from `validate_directory`                               | `project_root` does not exist or is not a directory          |
| `Unable to use the existing '{name}' dataset. The provided session list does not match...` | Set `force_recreate=True` or submit the matching list        |
| `Unable to define dataset '{name}'. The dataset does not exist under '{project_root}' and no sessions were provided...` | Provide `session_names`                                      |
| `Unable to resolve the directory for session '{s}' under '{project_root}'.` | Session name not unique or absent under the project          |
| `Unable to define dataset '{name}'. Dataset creation is currently supported only for mesoscope experiment sessions...` | Remove the non-mesoscope session or split the batch          |
| `Unable to define dataset '{name}'. All sessions in a dataset must share the same session type...` | Split the batch by session type                              |
| `Unable to define dataset '{name}'. All sessions in a dataset must be acquired by the same acquisition system...` | Split the batch by acquisition system                        |
| `Unable to define dataset '{name}'. The latest session '{s}' for animal '{a}' does not contain a 'surgery_metadata.yaml' file...` | Add the missing surgery file to the animal's latest session  |

### Execution errors (`execute_forging_jobs_tool` top-level `error` or `invalid_jobs[]`)

| Error pattern                              | Resolution                                                         |
|--------------------------------------------|--------------------------------------------------------------------|
| `An execution session is already active.`  | Wait for the current session or `cancel_forging_tool` first        |
| `No valid jobs to execute.`                | Every descriptor was rejected; check `invalid_jobs`                |
| `Missing required keys: [...]`             | Use the descriptor from `prepare_forging_batch_tool` verbatim      |
| `Tracker file not found: ...`              | Re-prepare the batch to regenerate the tracker                     |

### Per-job failure routing

| Error pattern                                                   | Action                                                                |
|-----------------------------------------------------------------|-----------------------------------------------------------------------|
| Behavior tracker not found / ambiguous                          | Rerun `/behavior-processing` for the session                          |
| Cindra single-recording tracker not found / ambiguous           | Rerun `/cindra:single-recording-processing` for the session           |
| Cindra multi-day file missing (`cell_fluorescence.npy`, etc.)   | Rerun `/cindra:multi-recording-processing` with the same dataset name |
| Hardware state YAML missing / missing required field            | See `/dataset-forging-input-format` and the assets plugin             |
| Experiment configuration YAML missing                           | See `/dataset-forging-input-format`                                   |
| Experiment descriptor YAML missing                              | See `/dataset-forging-input-format`; add the file under `raw_data/`   |
| Polars / Arrow read errors on a behavior feather                | Rerun `/behavior-processing` — the upstream feather is corrupt        |
| MCP tools unavailable                                           | Invoke `/forging-mcp-environment-setup`                               |
| Out of memory                                                   | Reduce `worker_budget`                                                |
| Corrupt tracker                                                 | `clean_forging_output_tool` → re-prepare                              |

---

## Related skills

| Skill                                           | Relationship                                                                  |
|-------------------------------------------------|-------------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`                | Prerequisite: MCP server connectivity                                         |
| assets plugin `/session-discovery`              | Upstream: session discovery and filtering                                     |
| `/dataset-forging-input-format`                 | Reference: upstream artifacts and session / dataset layout                    |
| `/dataset-forging-results`                      | Downstream: output verification, schemas, and querying                        |
| `/behavior-processing`                          | Upstream: produces behavior feathers consumed by forging                      |
| `/behavior-results`                             | Upstream reference: schema of the behavior feathers consumed here             |
| `/cindra:single-recording-processing`           | Upstream: produces single-recording cindra outputs consumed here              |
| `/cindra:multi-recording-processing`            | Upstream: produces multi-day cindra outputs (dataset name must match)         |
| `/cindra:single-recording-results`              | Upstream reference: schemas of the cindra single-recording outputs            |
| `/cindra:multi-recording-results`               | Upstream reference: schemas of the cindra multi-day outputs                   |

---

## Verification checklist

```text
Dataset Forging Workflow:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Received confirmed session names and project_root from the assets plugin's /session-discovery
- [ ] Confirmed every session is MESOSCOPE_EXPERIMENT
- [ ] Confirmed upstream /behavior-processing and /cindra:* outputs exist
- [ ] Prepared batch via prepare_forging_batch_tool
- [ ] Resolved any invalid_datasets with the user (force_recreate as needed)
- [ ] Presented discovered job counts per dataset
- [ ] Confirmed worker budget with user
- [ ] Verified no active execution session before dispatch
- [ ] Executed jobs via execute_forging_jobs_tool
- [ ] Monitored status until all jobs reached terminal state
- [ ] Investigated and retried failed jobs if needed (reset or clean + re-prepare)
- [ ] Handed off successful output to /dataset-forging-results
```
