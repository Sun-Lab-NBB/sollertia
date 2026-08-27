---
name: batch-processing
description: >-
  Orchestrates batch processing through the sollertia-forgery MCP server for all six batch pipelines. Covers batch
  preparation, job execution, monitoring, cancellation, tracker reset, output cleaning, and the prepared-batch
  registry. Use when running a checksum, runtime, microcontroller, video, two_photon, or forging batch, when
  monitoring or canceling a run, when re-running failed jobs, or when recovering a lost batch identifier.
user-invocable: false
---

# Batch processing

Runs the prepare-then-execute workflow that turns processing-unit roots into processed output. One generic tool set
serves every pipeline `dispatch.BATCH_PIPELINES` declares, selected by a `pipeline` string. This skill is the
**exclusive** owner of `prepare_batch_tool`, `execute_jobs_tool`, `get_processing_status_tool`,
`cancel_processing_tool`, `reset_processing_jobs_tool`, `clean_processing_output_tool`, `list_prepared_batches_tool`,
and `forget_prepared_batches_tool`. No other skill in the marketplace may document or call these eight tools.

- [tool-responses.md](references/tool-responses.md) carries each tool's complete return-key tree, every conditional key,
  and the condition producing it.

---

## Scope

**Covers:**
- Batch preparation over one pipeline and a list of processing-unit roots
- Job execution on the local process pool and on the configured compute server
- Progress monitoring, blocked-job reporting, the recorded closure outcome, and status presentation
- Cancellation, tracker reset, output cleaning, and error routing
- The durable prepared-batch registry and batch identifier recovery

**Does not cover:**
- Session and dataset root discovery. Owned by `assets:session-discovery`.
- Job planning, per-unit job estimates, and the declared resource model. Owned by `/job-planning`.
- The project manifest, the project job artifact, and manifest generation state. Owned by `/project-state`.
- Dataset hierarchy creation and dataset state. Owned by `/dataset-definition`.
- The `forging` pipeline's own prerequisites and dataset semantics. Owned by `/dataset-forging`.
- Remote project discovery and scheduler job reads. Owned by `/remote-execution`.
- Compute server credentials and transport settings. Owned by `/server-configuration`.
- What must exist on disk before a job can run. Owned by `/processing-input-format`.
- Output schemas and how to interpret them. Owned by `/processing-results`.
- MCP server connectivity and the `slf omp` runtime diagnostic. Owned by `/forging-mcp-environment-setup`.

**Handoff rules:** this skill drives the batch. Invoke the owning skill for anything decided by the unit roots, the
plan, the dataset hierarchy, or the output schema, then return here to prepare and execute.

---

## Agent requirements

You MUST drive every batch operation through the sollertia-forgery MCP tools. Do not import `sollertia_forgery`
functions directly and do not shell out to `slf`. Where the tools are unavailable, invoke
`/forging-mcp-environment-setup` to diagnose connectivity. The `slf` CLI is the human path and `/cli-reference` owns it.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

`assets:session-discovery` is the exclusive producer of the `session_paths` lists every batch consumes. You MUST obtain
unit roots from it, never assemble one yourself, and confirm both the selection and its single owning project with the
user.

You MUST treat `prepare_batch_tool` as an expensive write. It plans each unit, creates and aligns that unit's
processing trackers, and rewrites the project's plan and state artifacts before it registers anything, so call it once
per intended batch and never as a poll.

You MUST carry the returned `batch_id` forward, since `execute_jobs_tool` accepts batch identifiers and no path. Where
an identifier was lost, recover it with `list_prepared_batches_tool` rather than preparing the same work again.

---

## Available tools

### Preparation and execution tools

`prepare_batch_tool` materializes the host's artifacts for the named units, joins the plan table against the state
table, and records one dispatchable batch document under a fresh 16-character hexadecimal identifier.

```python
prepare_batch_tool(
    pipeline: str,
    session_paths: list[str],
    options: dict[str, Any] | None = None,
    host: str = "local",
    *,
    replan: bool = False,
    include_job_descriptors: bool = False,
) -> dict[str, Any]
```

| Parameter                 | Type                     | Default    | Description                                                                                  |
|---------------------------|--------------------------|------------|----------------------------------------------------------------------------------------------|
| `pipeline`                | `str`                    | (required) | One of the six batch pipelines. `manifest` is a processing pipeline but not a batch pipeline |
| `session_paths`           | `list[str]`              | (required) | Unit roots. Session roots for five pipelines, dataset roots for `forging`. One project only  |
| `options`                 | `dict[str, Any] \| None` | `None`     | Stamped onto every descriptor. `regenerate_checksum` is the only key any pipeline reads      |
| `host`                    | `str`                    | `"local"`  | `"local"` or `"remote"`. Under `"remote"` every path names a location on the server          |
| `replan`                  | `bool`                   | `False`    | Re-estimates the cores and memory the units' plan caches already hold                        |
| `include_job_descriptors` | `bool`                   | `False`    | Adds the raw `jobs` list. Dispatch reads descriptors from the record, not from the response  |

Every call issues a new identifier and writes a new document, so two preparations of overlapping work leave two batches
and executing both dispatches the same jobs twice. Succeeded jobs are omitted, so re-preparing after a partial run
queues only what is outstanding, and `success: true` with `total_jobs: 0` means every job either succeeded or is
blocked. `total_units` counts the units named rather than the units that produced jobs, so read `units[*].error` every
time. An unresolved unit reports either that the state table records no job of this pipeline for it, or that the plan
table carries no figures for its outstanding jobs.

`execute_jobs_tool` reads the recorded documents, reconciles their jobs against what is already running, clears the
tracker record of everything it is about to dispatch, and starts the run. It returns immediately.

```python
execute_jobs_tool(
    batch_ids: list[str],
    *,
    core_budget_override: int = -1,
    memory_budget_mb: int = -1,
    walltime_minutes: int = -1,
) -> dict[str, Any]
```

| Parameter              | Type        | Default    | Description                                                                                                              |
|------------------------|-------------|------------|--------------------------------------------------------------------------------------------------------------------------|
| `batch_ids`            | `list[str]` | (required) | Identifiers `prepare_batch_tool` returned. `batch_ids[0]` names the server-side directory                                |
| `core_budget_override` | `int`       | `-1`       | Cores a local batch may commit. Non-positive auto-resolves to the logical cores minus two                                |
| `memory_budget_mb`     | `int`       | `-1`       | Memory a local batch may commit. Non-positive auto-resolves to 85 percent of host memory or 1024 MB, whichever is larger |
| `walltime_minutes`     | `int`       | `-1`       | Wall time each remote allocation requests. Non-positive resolves to 480 minutes                                          |

There is no `host` parameter. A batch runs where it was prepared, read from the recorded document, and batches prepared
against different hosts are refused rather than split. The budget arguments apply to a local run alone and
`walltime_minutes` to a remote one. An explicit `memory_budget_mb` is honored with no floor, so a small value starves
the run, and a repeated identifier is not de-duplicated, so it contributes its jobs twice.

**Note:** `started: true` proves dispatch and nothing about outcomes. Locally it means a daemon manager thread began,
remotely it means the scheduler accepted the allocations, so never report a run as finished from this response. The
local branch also runs unguarded, so a failure to resolve the core allocations or the host memory surfaces as a raw tool
error rather than an error payload. Tracker records are cleared before that point, so a batch failing there has already
lost the records it was about to rerun.

### Monitoring and management tools

`get_processing_status_tool` is two readers under one signature. The local reader reports this process's pool and the
processing trackers, the remote reader reports the submission ledger and the scheduler.

```python
get_processing_status_tool(
    host: str = "local",
    batch_ids: list[str] | None = None,
    status_filter: str | None = None,
    session_paths: list[str] | None = None,
    job_ids: list[str] | None = None,
    job_names: list[str] | None = None,
    pipelines: list[str] | None = None,
    limit: int | None = None,
    start_row: int = 0,
    *,
    include_items: bool = False,
    detailed: bool = False,
) -> dict[str, Any]
```

| Parameter                           | Type                | Default   | Meaning under `host="local"`                                       | Meaning under `host="remote"`                       |
|-------------------------------------|---------------------|-----------|--------------------------------------------------------------------|-----------------------------------------------------|
| `host`                              | `str`               | `"local"` | This process's pool                                                | The outstanding allocations                         |
| `batch_ids`                         | `list[str] \| None` | `None`    | Names closed batches whose outcome to report. Not a listing filter | Restricts to these outstanding batches, and filters |
| `status_filter`                     | `str \| None`       | `None`    | Lowercase, one of `scheduled`, `running`, `succeeded`, `failed`    | An uppercase scheduler state such as `FAILED`       |
| `session_paths`                     | `list[str] \| None` | `None`    | Filters the per-job key `session_path`                             | Filters the per-job key `unit_path`                 |
| `job_ids`, `job_names`, `pipelines` | `list[str] \| None` | `None`    | Filter `job_id`, `job_name`, and `pipeline` respectively           | Same                                                |
| `limit`                             | `int \| None`       | `None`    | Page size, as the response contract defines it                     | Same                                                |
| `start_row`                         | `int`               | `0`       | Match index to begin at, negative clamps to zero                   | Same                                                |
| `include_items`                     | `bool`              | `False`   | Lists jobs when no filter is named                                 | Same                                                |
| `detailed`                          | `bool`              | `False`   | Adds resources, timing, provenance, and error text                 | Adds requested resources and the server log paths   |

Timing is `detailed=True` here, which adds `started_at`, `completed_at`, and `elapsed_seconds` to each listed job.
There is no timing tool and no output-verification tool. Verification runs through these breakdowns, through
`read_project_jobs_tool` under `/project-state`, and through `read_dataset_state_tool` under `/dataset-definition`.

**Note:** `summary`, `status`, and `breakdown` always describe the whole batch and never respect a filter. Only `jobs`,
`rows`, and `matched_rows` do. A local read with no live state ignores every filter argument, reporting the recorded
outcomes of the named batches where any exist and otherwise only that no batch is running in this process. Restarting
the MCP server loses the in-process run, so that second shape is what a restart produces while outcome files sit on
disk. Blocked jobs arrive as an integer `blocked_jobs` count plus a `blocked_reason` string rather than a list, and
their tracker record still reads scheduled, so list them with `status_filter="scheduled"`.

`cancel_processing_tool` stops a run. Locally it is cooperative, remotely it kills queued and running allocations alike
and lets the scheduler cascade the cancellation onto their dependents.

```python
cancel_processing_tool(host: str = "local", batch_ids: list[str] | None = None) -> dict[str, Any]
```

| Parameter   | Type                | Default   | Description                                                                          |
|-------------|---------------------|-----------|--------------------------------------------------------------------------------------|
| `host`      | `str`               | `"local"` | `"local"` for this process's pool, `"remote"` for the scheduler                      |
| `batch_ids` | `list[str] \| None` | `None`    | Outstanding remote batches to cancel, omit for all. Silently ignored under `"local"` |

**Note:** `canceled: true` does not mean work stopped. In-flight local jobs run to completion, and the remote
`canceled_jobs` figure counts every allocation the call named, including allocations that had already finished.
Cancellation leaves tracker records where they stand. A remote cancel additionally re-queries the scheduler, records a
closure outcome, and retires the batches from the ledger, so read the outcome rather than the outstanding listing.

`reset_processing_jobs_tool` returns tracked jobs to the scheduled state without running anything.

```python
reset_processing_jobs_tool(
    pipeline: str, unit_paths: list[str], job_ids: list[str] | None = None, host: str = "local"
) -> dict[str, Any]
```

| Parameter    | Type                | Default    | Description                                                                        |
|--------------|---------------------|------------|------------------------------------------------------------------------------------|
| `pipeline`   | `str`               | (required) | One of the six batch pipelines. A mismatched pipeline resolves a different tracker |
| `unit_paths` | `list[str]`         | (required) | Unit roots. This is the only tool in the set naming the parameter `unit_paths`     |
| `job_ids`    | `list[str] \| None` | `None`     | Tracker job identifiers from any listing. Omit to reset every job each unit tracks |
| `host`       | `str`               | `"local"`  | `"local"` or `"remote"`                                                            |

Every named unit receives the same identifier set and silently drops an identifier it does not track, so one call
safely covers a whole batch. An empty `job_ids` list reads as omitted and therefore resets everything, the larger
operation rather than the smaller one. `jobs_reset` counts the identifiers named rather than the records cleared and
reads `null` when every job was reset, so `success: true` proves the call completed, not that anything changed.

`clean_processing_output_tool` removes a pipeline's tracker and, where the pipeline owns one, its output directory.

```python
clean_processing_output_tool(pipeline: str, session_paths: list[str], host: str = "local") -> dict[str, Any]
```

| Parameter       | Type        | Default    | Description                                                          |
|-----------------|-------------|------------|----------------------------------------------------------------------|
| `pipeline`      | `str`       | (required) | One of the six batch pipelines                                       |
| `session_paths` | `list[str]` | (required) | Unit roots, dataset roots for `forging`. Server paths under `remote` |
| `host`          | `str`       | `"local"`  | `"local"` or `"remote"`                                              |

Cleaning discards work irreversibly. The running-batch guard covers the local pool alone, so a remote clean is accepted
while allocations are in flight and will fail the jobs reading those paths. A unit that cannot be loaded is skipped and
the rest are cleaned, with the skip echoed to the console, so `total_paths: 0` covers both an empty removal and a batch
of unloadable units.

### Registry tools

`list_prepared_batches_tool` reads the registry of prepared and settled batches, which is the only recovery path for a
lost identifier.

```python
list_prepared_batches_tool(
    batch_ids: list[str] | None = None,
    pipelines: list[str] | None = None,
    host: str | None = None,
    limit: int | None = None,
    start_row: int = 0,
    *,
    detailed: bool = False,
) -> dict[str, Any]
```

| Parameter   | Type                | Default | Description                                                                                      |
|-------------|---------------------|---------|--------------------------------------------------------------------------------------------------|
| `batch_ids` | `list[str] \| None` | `None`  | Restricts to these batches. Every named identifier must be held or the call errors               |
| `pipelines` | `list[str] \| None` | `None`  | Restricts to batches dispatching these pipelines. Unvalidated, so an unknown one matches nothing |
| `host`      | `str \| None`       | `None`  | `"local"` or `"remote"`. `None` applies no host filter, unlike every other tool                  |
| `limit`     | `int \| None`       | `None`  | Page size, as the response contract defines it                                                   |
| `start_row` | `int`               | `0`     | Match index to begin at, negative clamps to zero                                                 |
| `detailed`  | `bool`              | `False` | Adds `options`, `job_names`, and `unit_names`, for prepared records only                         |

The listing is always present and there is no `include_items` parameter. Records return in ascending identifier order,
neither preparation nor chronological order. `outcome_recorded: true` marks a batch closure already settled, and
closure deletes the prepared document, so such a record carries no `unit_count` and no detail fields.

`forget_prepared_batches_tool` removes what the registry holds for the named batches, both the prepared document and
the recorded outcome.

```python
forget_prepared_batches_tool(batch_ids: list[str]) -> dict[str, Any]
```

| Parameter   | Type        | Default    | Description                                                         |
|-------------|-------------|------------|---------------------------------------------------------------------|
| `batch_ids` | `list[str]` | (required) | The batches to remove, as `list_prepared_batches_tool` reports them |

An empty list is an error rather than a wildcard, since the call removes what the registry records for each named
batch. Forgetting an unrun batch destroys its descriptors, and forgetting a settled batch destroys the only durable
record of what its jobs reached, so read the outcome first.

---

## Pipeline architecture

```text
unit roots (assets:session-discovery)
  -> prepare_batch_tool     plans units, aligns trackers, rewrites the project plan and state artifacts, issues batch_id
  -> execute_jobs_tool      one process pool locally, one scheduler allocation per job remotely
  -> get_processing_status  tracker records while the run is live, the recorded outcome once closure settles it
```

Prepared batches are records under `remote_state/prepared_batches/` in the sollertia working directory and they outlive
the MCP server process. The in-process run state does not, so a restart loses the live pool while every identifier
survives. The six batch pipelines below are the accepted `pipeline` strings, with stages in the order the jobs run.

| Pipeline          | Unit kind | Stages in order, with the constant declaring each name                                                                                                                                                                                   | Specifier convention                                                                                          | Distinctive                                                                                                                                                            |
|-------------------|-----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `checksum`        | session   | `checksum_resolution` (`managing.checksum.CHECKSUM_JOB_NAME`)                                                                                                                                                                            | the unit's session name                                                                                       | The only pipeline reading an option, `regenerate_checksum`. Owns no output directory, so cleaning removes the tracker and leaves the stored checksum baseline in place |
| `runtime`         | session   | `runtime_processing` (`runtime.pipeline.RUNTIME_JOB_NAME`)                                                                                                                                                                               | the acquisition system's runtime log source identifier                                                        | One job per session, resolved from the registry binding the acquisition system donates                                                                                 |
| `microcontroller` | session   | `microcontroller_data_extraction` (upstream `CONTROLLER_EXTRACTION_JOB_NAME`), then `module_parsing` (`microcontrollers.pipeline.PARSE_JOB_NAME`)                                                                                        | controller identifier as a string, then that identifier, the module type, and the module id joined by hyphens | Extraction runs in process through ataraxis-communication-interface, parsing through a system-donated parser                                                           |
| `video`           | session   | `camera_timestamp_extraction` (upstream `CAMERA_EXTRACTION_JOB_NAME`), then `camera_timestamp_rename` (`video.pipeline.RENAME_JOB_NAME`), with `pose_tracking` (`TRACKING_JOB_NAME`) and `motion_energy` (`ENERGY_JOB_NAME`) independent | camera source identifier for extraction and motion energy, empty for rename and tracking                      | Three independent chains in one pipeline, so one blocked chain leaves the others dispatchable                                                                          |
| `two_photon`      | session   | `binarization`, then `registration`, then `processing`, then `combination` (cindra's `SingleRecordingJobNames`)                                                                                                                          | empty, then `plane_{n}`, then `plane_{n}`, then empty                                                         | The only pipeline carrying a priming step, which preparation runs before its jobs are discovered                                                                       |
| `forging`         | dataset   | `multiday_discovery` (`forging.pipeline.MULTIDAY_DISCOVERY_JOB_NAME`), then `multiday_extraction` (`MULTIDAY_EXTRACTION_JOB_NAME`), then `session_data_assembly` (`FORGING_JOB_NAME`)                                                    | animal identifier, then session name, then session name                                                       | `session_paths` names dataset roots. Cleaning removes the whole assembled dataset hierarchy. `/dataset-forging` owns its specifics                                     |

A job identifier is derived from the job name and the specifier alone, so the same stage of two units shares one
identifier and every per-unit operation is scoped by the unit path paired with that identifier.

---

## Processing workflow

### Execution model

1. **Prepare** materializes the host's artifacts and records a dispatchable batch document without running any job. It
   is not idempotent in identity, since each call issues an identifier and rewrites plan caches and project artifacts.

2. **Execute** dispatches the recorded batch. A local run holds one batch at a time in a daemon-threaded pool, and a
   remote run submits one scheduler allocation per job in dependency order and may hold many batches at once.

### Workflow steps

1. **Orient before starting.** Where the project may already have been processed, call `list_prepared_batches_tool`
   first. Its `breakdown` names the pipelines and hosts the registry holds, and `outcome_recorded` marks settled work.

2. **Resolve the unit roots.** Obtain session roots, or dataset roots for `forging`, through `assets:session-discovery`,
   then confirm the selection and the single owning project with the user.

3. **Confirm the host.** A local batch reads this filesystem and a remote batch reads paths on the compute server, so
   confirm the server configuration through `/server-configuration` before a remote run.

4. **Read the plan.** Invoke `/job-planning` to size the work first, since preparation adopts the figures the plan
   caches already hold unless `replan=True` deliberately re-estimates them.

5. **Prepare the batch.** Call `prepare_batch_tool` with the confirmed pipeline, unit roots, and host. Record the
   returned `batch_id`, reconcile `total_units` against `units[*].error` and `total_jobs` against `total_blocked_jobs`,
   then report every unresolved unit and every blocked job before executing.

6. **Confirm the budgets.** Present the default local budgets, which resolve to the logical cores minus two and to 85
   percent of host memory, and the default remote wall time of 480 minutes. Never estimate a figure yourself.

7. **Execute the batch.** Call `execute_jobs_tool` with the recorded identifiers and confirmed budgets. Read back
   `pool_size`, the resolved `core_budget` and `memory_budget_mb`, and `job_allocations` for a local run, or
   `submissions`, `adopted_jobs`, and `batch_directory` for a remote one.

8. **Monitor progress.** Poll `get_processing_status_tool` on the same host, since nothing in this tool set blocks or
   waits. Priorities, tracker resets, and outcome refreshes precede the first admission, so a first read can look idle.

9. **Handle completion.** A local run has finished once a read reports the closed-batch message and carries `outcomes`,
   and a remote run once no allocation remains outstanding. Read each outcome's `succeeded`, `failed`, `blocked`, and
   `outstanding` counts separately, because `complete` is a strict equality against a total that includes blocked jobs.

10. **Verify the output.** On success, invoke `/processing-results` for the output layout and `/project-state` for the
    project-wide job breakdown. On failure, follow Error routing.

---

## Status formatting

Present a live batch as one header block and one job table. Fill the rows from a `get_processing_status_tool` call
with `include_items=True` and `detailed=True`. The header identity comes from the preparation record, not this read.

```text
**Batch processing status**

Batch 4f2a9c1e77b30d58 | pipeline video | host local | status processing | active true | canceled false
Summary: 8 total | 4 succeeded | 1 running | 3 scheduled | 0 failed | 2 blocked

| Job id           | Job name                    | Specifier | Unit   | Status    | Elapsed |
|------------------|-----------------------------|-----------|--------|-----------|---------|
| 3f9c1a2b4d5e6f70 | camera_timestamp_extraction | 0         | unit_a | succeeded | 41.2 s  |
| 8b1e0d7c6a5f4e32 | camera_timestamp_extraction | 1         | unit_a | running   | 12.8 s  |
| c47a5e9b18d20f63 | camera_timestamp_rename     | --        | unit_a | scheduled | --      |
| 5d3f8a0c2b71e4d9 | motion_energy               | 0         | unit_b | succeeded | 96.5 s  |
```

The header `status` is the roll-up label, resolved from the counts by a fixed priority, so the labels are not mutually
exclusive descriptions of the batch.

| Label         | Condition                                                                                  |
|---------------|--------------------------------------------------------------------------------------------|
| `failed`      | At least one job failed. Outranks every other label, so the batch may still hold successes |
| `completed`   | The batch holds jobs and every one of them succeeded                                       |
| `processing`  | At least one job is running                                                                |
| `not_started` | The batch holds jobs and every one of them is still scheduled                              |
| `in_progress` | Everything else, including a batch holding no job at all                                   |

Render a dash where a key is absent, since a listing drops any empty field and `specifier`, `error_message`,
`executor_id`, and the timing fields vanish from a row rather than reading as null. Never report `failed` as a failed
batch without naming the succeeded count. A remote batch carries no roll-up label and no `canceled` flag, its status
values are uppercase scheduler states, and its unit column is `unit_path`.

---

## Re-running failed jobs

The straightforward retry is to prepare the pipeline again and execute the new batch, which queues only the work that is
still outstanding.

1. **Identify the failures.** Call `get_processing_status_tool` with `status_filter="failed"`, `include_items=True`,
   and `detailed=True`, then read each row's `error_message`. For a remote batch, follow the `output_log` and
   `error_log` paths the detailed row carries to the allocation's own diagnostics on the server.

2. **Decide between reset and clean.** Use `reset_processing_jobs_tool` where the failure cause was external and the
   partial output is harmless. Use `clean_processing_output_tool` where the output itself is suspect, remembering that
   cleaning `forging` removes the whole assembled dataset while cleaning `checksum` removes only the tracker.

3. **Re-prepare and re-execute.** Call `prepare_batch_tool` again for the same pipeline and units, confirm the new
   `total_jobs` matches the work you intended to retry, then call `execute_jobs_tool` with the new identifier.

A job reported blocked is not a failure and never reached an execution backend. A run reports a job blocked rather than
dispatched when it can neither queue the upstream stage nor confirm that the stage already succeeded, and blocking
propagates to the dependents. Run the upstream stage first, then execute the dependents again.

---

## Error routing

Errors carry `success: false` and a human-readable `error` string alone, with no error code and no structured detail,
so match on `success` and read the string. The one exception is `No valid jobs to execute.`, which also carries
`invalid_jobs`. Two rejections are shared, one by every tool taking a pipeline and one by every tool taking a host.

```text
Unsupported batch pipeline '{pipeline}'. Available: checksum, forging, microcontroller, runtime, two_photon, video.
Unsupported host '{host}'. Available: local, remote.
```

| Message                                                                                                                                                                   | Tool                           | Remedy                                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| `Unable to prepare the {host} '{pipeline}' batch. {exception}`                                                                                                            | `prepare_batch_tool`           | Read the wrapped exception, which names the missing plan table, the multi-project batch, or the absent server configuration |
| `Unable to read the prepared batches {batch_ids}. {exception}`                                                                                                            | `execute_jobs_tool`            | A registry file is unreadable or malformed. Re-list, then re-prepare                                                        |
| `No prepared batch exists for identifier(s) {missing}. Prepare the pipeline again to register its jobs.`                                                                  | `execute_jobs_tool`            | Recover the identifier with `list_prepared_batches_tool`                                                                    |
| `No batch was named.`                                                                                                                                                     | `execute_jobs_tool`            | The identifier list was empty                                                                                               |
| `Unable to execute batches prepared against the hosts {hosts} together. …`                                                                                                | `execute_jobs_tool`            | Dispatch one host's batches at a time                                                                                       |
| `No dispatchable jobs. Every prepared job is blocked or already succeeded.`                                                                                               | `execute_jobs_tool`            | Read the preparation's `blocked_jobs` list and run the upstream stage                                                       |
| `No valid jobs to execute.`                                                                                                                                               | `execute_jobs_tool`            | The response carries `invalid_jobs`, each entry naming the field that failed                                                |
| `A batch is already running. Wait for it to finish or cancel it before starting another.`                                                                                 | `execute_jobs_tool`            | One local pool holds one batch. Wait, or cancel                                                                             |
| `Unable to submit the remote batch. {exception}`, `Unable to query the remote batches. {exception}`, and `Unable to cancel the remote batches. {exception}`               | execute, status, and cancel    | Each wraps a connection, scheduler, or closure failure on the remote host                                                   |
| `Unable to report the named batch(es) {uncovered}. …` and `No outstanding remote batch has identifier(s) {unknown}. …`                                                    | `get_processing_status_tool`   | A live local run answers only for the batches it covers, and a remote read only for the outstanding ones                    |
| `Unknown status '{status_filter}'. Available: failed, running, scheduled, succeeded.` locally, `Unknown scheduler state '{status_filter}'. Available: {states}.` remotely | `get_processing_status_tool`   | Local labels are lowercase, remote scheduler states are uppercase                                                           |
| `Unable to read the recorded outcomes of {named}. {exception}`                                                                                                            | `get_processing_status_tool`   | An outcome file is unreadable. Re-list the registry                                                                         |
| `No batch is running.`                                                                                                                                                    | `cancel_processing_tool`       | Nothing is live in this process                                                                                             |
| `No remote batch is outstanding. …`                                                                                                                                       | `cancel_processing_tool`       | Read the recorded outcome instead of the outstanding listing                                                                |
| `The named batches hold no allocation to cancel.`                                                                                                                         | `cancel_processing_tool`       | Confirm the identifiers through a remote status read first                                                                  |
| `Unable to reset the {host} '{pipeline}' jobs. {exception}` and `Unable to clean the {host} '{pipeline}' output. {exception}`                                             | reset and clean                | Read the wrapped exception, which for a remote call names the failing server-side command                                   |
| `A batch is currently running. Wait for it to finish or cancel it before cleaning output.`                                                                                | `clean_processing_output_tool` | The guard covers the local pool alone                                                                                       |
| `No prepared batch has identifier(s) {unknown}. Held: {held}.`                                                                                                            | `list_prepared_batches_tool`   | The message enumerates what the registry does hold                                                                          |
| `Unable to read the prepared batches under '{directory}'. {exception}`                                                                                                    | `list_prepared_batches_tool`   | The registry directory is unreadable                                                                                        |
| `Unable to forget a batch without an identifier. …`                                                                                                                       | `forget_prepared_batches_tool` | Name the batches, since there is no wildcard                                                                                |
| `Unable to forget the batches {batch_ids} under '{directory}'. {exception}`                                                                                               | `forget_prepared_batches_tool` | Removal was partial and its report is lost. Re-list to see what remains                                                     |

---

## Related skills

| Skill                                      | Relationship                                                                      |
|--------------------------------------------|-----------------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`           | Prerequisite: server connectivity, the response contract, and the `slf omp` check |
| `assets:session-discovery`                 | Upstream: the exclusive producer of the `session_paths` lists                     |
| `/job-planning`                            | Upstream: unit plans, job estimates, and the declared resource model              |
| `/dataset-definition`                      | Upstream: the dataset hierarchy a `forging` batch consumes                        |
| `/server-configuration`                    | Upstream: the compute server credentials a remote batch needs                     |
| `/processing-input-format`                 | Reference: what each pipeline requires on disk before a job can run               |
| `/dataset-forging`                         | Reference: the `forging` pipeline's own prerequisites and dataset semantics       |
| `/remote-execution`                        | Adjacent: remote project discovery and scheduler job reads                        |
| `/processing-results`                      | Downstream: output layouts and how to interpret them                              |
| `/project-state`                           | Downstream: the project manifest and job artifacts a batch refreshes              |
| `/cli-reference`                           | Reference: the human-facing `slf` command surface                                 |
| `/pipeline`                                | Context: where batch processing sits in the end-to-end workflow                   |
| `mesoscope:mesoscope-vr-processing-schema` | Reference: the acquisition-system donations that fill the registry seams          |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill

Batch workflow, tool-settled (run get_processing_status_tool on the batch, then read_project_jobs_tool):
- [ ] sollertia-forgery MCP server is connected
- [ ] Unit roots came from assets:session-discovery and every unit belongs to one project
- [ ] Batch prepared via prepare_batch_tool and its batch_id recorded
- [ ] Every entry of units[*] reconciled, with each unresolved unit reported to the user
- [ ] total_blocked_jobs reconciled, with each blocked job's upstream stage named
- [ ] Jobs dispatched via execute_jobs_tool using the recorded batch_ids
- [ ] Every job reads succeeded, or reads failed with an error_message that was investigated

Batch workflow, agent-judged:
- [ ] Pipeline and host confirmed with the user before preparation
- [ ] Resource budgets confirmed with the user before execution
- [ ] Blocked jobs distinguished from failed jobs in every report
- [ ] Failed jobs retried through reset or clean, with the destructive cost of clean stated first
- [ ] Output verified through /processing-results before the batch is called complete
```
