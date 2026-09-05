---
name: batch-processing
description: >-
  Orchestrates batch processing through the sollertia-forgery MCP server for all six batch pipelines. Covers batch
  preparation, job execution, monitoring, cancellation, tracker reset, output cleaning, and the prepared-batch registry.
  Use when running a checksum, runtime, microcontroller, video, two_photon, or forging batch, when monitoring or
  canceling a run, when re-running failed jobs, or when recovering a lost batch identifier.
user-invocable: false
---

# Batch processing

Runs the prepare-then-execute workflow that turns processing-unit roots into processed output. One generic tool set
serves every pipeline `dispatch.BATCH_PIPELINES` declares, selected by a `pipeline` string. This skill is the
**exclusive** owner of `prepare_batch_tool`, `execute_jobs_tool`, `get_processing_status_tool`,
`cancel_processing_tool`, `retire_remote_batches_tool`, `reset_processing_jobs_tool`, `clean_processing_output_tool`,
`list_prepared_batches_tool`, and `forget_prepared_batches_tool`. No other skill in the marketplace may document or
call these nine tools.

- [tool-responses.md](references/tool-responses.md) carries each tool's complete return-key tree and every condition.

---

## Scope

**Covers:**
- Batch preparation over one pipeline and a list of processing-unit roots
- Job execution on the local process pool and on the configured compute server
- Progress monitoring, blocked-job reporting, the recorded closure outcome, and status presentation
- Cancellation, remediation of a remote batch and its ledger entry, tracker reset, cleaning, and error routing
- The durable prepared-batch registry and batch identifier recovery

**Does not cover:**
- Unit-root discovery, with session roots owned by `assets:session-discovery` and dataset roots by
  `/dataset-definition`.
- Job planning, per-unit job estimates, and the declared resource model. Owned by `/job-planning`.
- The project manifest, the project job artifact, and manifest generation state. Owned by `/project-state`.
- Dataset hierarchy creation and dataset state. Owned by `/dataset-definition`.
- The `forging` pipeline's own prerequisites and dataset semantics. Owned by `/dataset-forging`.
- Remote project discovery, scheduler job reads, and what each resolved state, verdict, and remediation means. Owned
  by `/remote-execution`.
- Compute server credentials and transport settings. Owned by `/server-configuration`.
- What must exist on disk before a job can run, owned by `/processing-input-format`, and the output schemas it
  produces, owned by `/processing-results`.
- MCP server connectivity and the `slf omp` runtime diagnostic. Owned by `/forging-mcp-environment-setup`.

**Handoff rules:** this skill drives the batch. Invoke the owning skill for anything decided by the unit roots, the
plan, the dataset hierarchy, or the output schema, then return here to prepare and execute.

---

## Agent requirements

You MUST drive every batch operation through the sollertia-forgery MCP tools. Do not import `sollertia_forgery`
functions or shell out to `slf`, the human path `/cli-reference` owns. Where the tools are unavailable, invoke
`/forging-mcp-environment-setup`, which owns the response envelope and the staged-read contract.

`assets:session-discovery` is the exclusive producer of the session roots the five per-session pipelines consume, and
`/dataset-definition` of the dataset roots a `forging` batch consumes. You MUST obtain unit roots from one of them,
never assemble one yourself, and confirm both the selection and its owning project with the user.

You MUST treat `prepare_batch_tool` as an expensive write. It plans each unit, creates and aligns that unit's processing
trackers, and rewrites the project's plan and state artifacts before registering anything, so call it once per intended
batch and never as a poll. You MUST then carry the returned `batch_id` forward, since `execute_jobs_tool` accepts
identifiers and no path, recovering a lost one with `list_prepared_batches_tool` rather than preparing the work again.

---

## Available tools

### Preparation and execution tools

`prepare_batch_tool` materializes the host's artifacts for the named units, joins the plan table against the state
table, and records one dispatchable batch document under a fresh 16-character hexadecimal identifier.

```python
prepare_batch_tool(pipeline: str, session_paths: list[str], options: dict[str, Any] | None = None,
                   host: str = "local", *, replan: bool = False,
                   include_job_descriptors: bool = False) -> dict[str, Any]
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
blocked. `total_units` counts the units named rather than those that produced jobs, so read `units[*].error` every time:
an unresolved unit reports either no state-table job of this pipeline, or no plan figures for its outstanding jobs.

`execute_jobs_tool` reads the recorded documents, reconciles their jobs against what is already running, clears the
tracker record of everything it is about to dispatch, and starts the run. It returns immediately. Remotely it also
withholds any job that resolves as running with no allocation to adopt, and every job downstream of one, reporting
them under `withheld_jobs` and leaving their trackers untouched.

```python
execute_jobs_tool(batch_ids: list[str], *, core_budget_override: int = -1, memory_budget_mb: int = -1,
                  walltime_minutes: int = -1) -> dict[str, Any]
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
remotely that the scheduler accepted the allocations, so never report a run as finished from this response. The local
branch also runs unguarded, so a failure to resolve the core allocations or the host memory surfaces as a raw tool error
rather than an error payload, after tracker records were already cleared for the rerun.

### Monitoring and management tools

`get_processing_status_tool` is two readers under one signature. The local reader reports this process's pool and the
processing trackers, the remote reader reports the submission ledger and the scheduler.

```python
get_processing_status_tool(host: str = "local", batch_ids: list[str] | None = None, status_filter: str | None = None,
                           session_paths: list[str] | None = None, job_ids: list[str] | None = None,
                           job_names: list[str] | None = None, pipelines: list[str] | None = None,
                           limit: int | None = None, start_row: int = 0, *, include_items: bool = False,
                           detailed: bool = False) -> dict[str, Any]
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

A remote read resolves as well as reports. It regenerates and reads each covered project's state artifacts, reads
scheduler accounting and the queue, resolves every covered allocation once, and closes each batch whose entries all
prescribe the plain `drop` remediation. Each listed allocation carries a `scheduler_state` of `held`, `settled`, or
`gone`, the `tracker_status` its job recorded, a `verdict` of `running`, `finished`, `failed`, `abandoned`, or
`stranded`, and the `remediation` it prescribes, while each batch carries a `progress` verdict of `progressing`,
`stalled`, or `awaiting_closure` with a `verdicts` count, `stranded_allocations`, `unresolvable_allocations`, and a
`remedy`. The response carries `stalled_batch_ids`, `uncovered_batch_ids` naming any batch another process recorded
while the read ran, and `scheduler_read_error`. `active` counts the `running` verdict rather than the `held` scheduler
state, so a held tracker claim or an executor outside the scheduler sets it even where every recorded allocation reads
`settled` or `gone`, and no stalled batch sets it, so read that list. `/remote-execution` owns those values and
`retire_remote_batches_tool` acts on them.

**Note:** `summary`, `status`, and `breakdown` describe the whole covered batch and never respect a per-job filter, and
only `jobs`, `rows`, and `matched_rows` do. Remotely, `batch_ids` narrows the covered batches, so it narrows `active`,
`summary`, and `breakdown` with them. A local read with no live state ignores every filter, reporting the recorded
outcomes of the named batches where any exist and otherwise only that no batch is running in this process, which is also
what a restarted server returns while outcome files sit on disk. Blocked jobs arrive as a `blocked_jobs` count plus a
`blocked_reason` string, and their tracker record still reads scheduled, so list them with `status_filter="scheduled"`.

`cancel_processing_tool` stops a run. Locally it is cooperative, remotely it kills queued and running allocations alike
and lets the scheduler cascade the cancellation onto their dependents.

```python
cancel_processing_tool(host: str = "local", batch_ids: list[str] | None = None) -> dict[str, Any]
```

| Parameter   | Type                | Default   | Description                                                                          |
|-------------|---------------------|-----------|--------------------------------------------------------------------------------------|
| `host`      | `str`               | `"local"` | `"local"` for this process's pool, `"remote"` for the scheduler                      |
| `batch_ids` | `list[str] \| None` | `None`    | Outstanding remote batches to cancel, omit for all. Silently ignored under `"local"` |

**Note:** `canceled: true` does not mean work stopped. In-flight local jobs run to completion, the remote
`canceled_jobs` figure counts every allocation the call named including ones already finished, and cancellation leaves
tracker records where they stand. A remote cancel additionally resolves every named batch the way a status read does and
closes each one whose entries all prescribe the plain `drop` remediation, so read the outcome rather than the
outstanding listing. A batch still holding a live allocation, or a job whose tracker still claims the run, stays
outstanding, and the tool below remediates it.

`retire_remote_batches_tool` applies each allocation's resolved remediation and then drops the named batches from the
submission ledger, which is the only way to release a remote batch the scheduler can no longer settle.
`/remote-execution` owns the ledger, the verdicts, and the state table sending a caller here.

```python
retire_remote_batches_tool(batch_ids: list[str], *, force: bool = False,
                           drop_without_outcome: bool = False) -> dict[str, Any]
```

| Parameter              | Type        | Default    | Description                                                                             |
|------------------------|-------------|------------|-----------------------------------------------------------------------------------------|
| `batch_ids`            | `list[str]` | (required) | Outstanding batches, as a remote status read reports them. An empty list errors         |
| `force`                | `bool`      | `False`    | Also remediates batches holding a `running` allocation, cancelling every held one first |
| `drop_without_outcome` | `bool`      | `False`    | Also drops the entries when closure cannot snapshot what their jobs recorded            |

**Note:** there is no `host` parameter, since only a remote batch has a ledger entry, and no wildcard, since the drop is
irreversible. The call resolves every allocation exactly as the status read does, then cancels under `force` alone,
resets each `stranded` job to `SCHEDULED` on its own tracker, snapshots each named batch through the same closure a
settled batch takes, and drops the entries. **That cancellation is not confined to this ledger**: it also covers the
allocation each job's tracker claims, so `force` can kill an allocation another machine submitted; `/remote-execution`
owns why, and the user confirms it. Only `stranded` writes a tracker, so a recorded success or failure survives
untouched. Each flag waives one guarantee and names only itself in its refusal: `force` disturbs a `running` allocation,
`drop_without_outcome` discards the last record naming the run. A failed tracker read, accounting read, cancellation,
reset, or ledger lock is reported rather than waived, since each leaves the world unchanged. This clears the submission
ledger alone, where `forget_prepared_batches_tool` clears the prepared documents and outcomes, and the empty-ledger
refusal is tested first, so an empty `batch_ids` meets that one.

`reset_processing_jobs_tool` returns tracked jobs to the scheduled state without running anything.

```python
reset_processing_jobs_tool(pipeline: str, unit_paths: list[str], job_ids: list[str] | None = None,
                           host: str = "local") -> dict[str, Any]
```

| Parameter    | Type                | Default    | Description                                                                        |
|--------------|---------------------|------------|------------------------------------------------------------------------------------|
| `pipeline`   | `str`               | (required) | One of the six batch pipelines. A mismatched pipeline resolves a different tracker |
| `unit_paths` | `list[str]`         | (required) | Unit roots. This is the only tool in the set naming the parameter `unit_paths`     |
| `job_ids`    | `list[str] \| None` | `None`     | Tracker job identifiers from any listing. Omit to reset every job each unit tracks |
| `host`       | `str`               | `"local"`  | `"local"` or `"remote"`                                                            |

Every named unit receives the same identifier set and silently drops an identifier it does not track, so one call
safely covers a whole batch. An empty `job_ids` list reads as omitted and therefore resets everything. `jobs_reset`
counts the identifiers named rather than the records cleared and reads `null` when every job was reset, so
`success: true` proves the call completed, not that anything changed.

`clean_processing_output_tool` removes a pipeline's tracker and, where the pipeline owns one, its output directory.

```python
clean_processing_output_tool(pipeline: str, session_paths: list[str], host: str = "local") -> dict[str, Any]
```

| Parameter       | Type        | Default    | Description                                                          |
|-----------------|-------------|------------|----------------------------------------------------------------------|
| `pipeline`      | `str`       | (required) | One of the six batch pipelines                                       |
| `session_paths` | `list[str]` | (required) | Unit roots, dataset roots for `forging`. Server paths under `remote` |
| `host`          | `str`       | `"local"`  | `"local"` or `"remote"`                                              |

Cleaning discards work irreversibly. The running-batch guard covers this process's pool alone, so a remote clean is
accepted while allocations are in flight and will fail the jobs reading those paths. A unit that cannot be loaded is
skipped and the rest are cleaned, so `total_paths: 0` covers both an empty removal and a batch of unloadable units.

**Run one server per working directory.** The guard lives in the server process, and local reconciliation treats every
job whose tracker reads running as a pool that died. A second `slf mcp` server sharing one working directory therefore
clears the first server's records and dispatches those jobs again over the same output paths.

### Registry tools

`list_prepared_batches_tool` reads the registry of prepared and settled batches, which is the only recovery path for a
lost identifier.

```python
list_prepared_batches_tool(batch_ids: list[str] | None = None, pipelines: list[str] | None = None,
                           host: str | None = None, limit: int | None = None, start_row: int = 0, *,
                           detailed: bool = False) -> dict[str, Any]
```

| Parameter   | Type                | Default | Description                                                                                      |
|-------------|---------------------|---------|--------------------------------------------------------------------------------------------------|
| `batch_ids` | `list[str] \| None` | `None`  | Restricts to these batches. Every named identifier must be held or the call errors               |
| `pipelines` | `list[str] \| None` | `None`  | Restricts to batches dispatching these pipelines. Unvalidated, so an unknown one matches nothing |
| `host`      | `str \| None`       | `None`  | `"local"` or `"remote"`. `None` applies no host filter, unlike every other tool                  |
| `limit`     | `int \| None`       | `None`  | Page size, as the response contract defines it                                                   |
| `start_row` | `int`               | `0`     | Match index to begin at, negative clamps to zero                                                 |
| `detailed`  | `bool`              | `False` | Adds `options`, `job_names`, and `unit_names`, for prepared records only                         |

The listing is always present, there is no `include_items` parameter, and records return in ascending identifier order.
`outcome_recorded: true` marks a settled batch, whose prepared document closure deleted, so it carries no `unit_count`
and no detail fields.

`forget_prepared_batches_tool` removes what the registry holds for the named batches, both the prepared document and
the recorded outcome.

```python
forget_prepared_batches_tool(batch_ids: list[str]) -> dict[str, Any]
```

| Parameter   | Type        | Default    | Description                                                         |
|-------------|-------------|------------|---------------------------------------------------------------------|
| `batch_ids` | `list[str]` | (required) | The batches to remove, as `list_prepared_batches_tool` reports them |

An empty list is an error rather than a wildcard. Forgetting an unrun batch destroys its descriptors, and forgetting a
settled batch destroys the only durable record of what its jobs reached, so read the outcome first.

---

## Pipeline architecture

```text
unit roots (assets:session-discovery)
  -> prepare_batch_tool     plans units, aligns trackers, rewrites the project plan and state artifacts, issues batch_id
  -> execute_jobs_tool      one process pool locally, one scheduler allocation per job remotely
  -> get_processing_status  tracker records while the run is live, the recorded outcome once closure settles it
```

Prepared batches are records under `remote_state/prepared_batches/` in the sollertia working directory and outlive the
MCP server process, while the in-process run state does not, so a restart loses the live pool and keeps every
identifier. The six batch pipelines below are the accepted `pipeline` strings, with stages in the order the jobs run.

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
2. **Execute** dispatches the named prepared batches. A local pool holds one run at a time and a second dispatch is
   refused, though one call may carry several prepared batches into that run against one pair of budgets. A remote run
   submits one scheduler allocation per job in dependency order and may hold many batches at once.

### Workflow steps

1. **Orient before starting.** Where the project may already have been processed, call `list_prepared_batches_tool`
   first. Its `breakdown` names the pipelines and hosts the registry holds, and `outcome_recorded` marks settled work.
2. **Resolve the unit roots.** Obtain session roots through `assets:session-discovery` and dataset roots for `forging`
   through `/dataset-definition`, then confirm the selection and the single owning project with the user.
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
   Remotely, read `stalled_batch_ids` each poll and remediate what it names, since it never settles on its own.
9. **Handle completion.** A local run has finished once a read reports the closed-batch message and carries `outcomes`,
   and a remote run once no allocation remains outstanding. Read each outcome's `succeeded`, `failed`, `blocked`, and
   `outstanding` counts separately, because `complete` is a strict equality against a total that includes blocked jobs.
   A local run whose closure recorded no outcome, because every close raised or its batches had already been retired,
   keeps its live-run state instead, so `active: false` carrying `status`, `summary`, and `breakdown` but no
   closed-batch message is equally a finished run. That `summary` carries neither `blocked` nor `outstanding`, so take
   blocked work from the separate `blocked_jobs` count and confirm the batches through `list_prepared_batches_tool`.
10. **Verify the output.** On success, invoke `/processing-results` for the output layout and `/project-state` for the
    project-wide job breakdown. On failure, follow Error routing.

---

## Status formatting

Present a live batch as one header block and one job table. Fill the rows from a `get_processing_status_tool` call
with `include_items=True` and `detailed=True`. The header identity comes from the preparation record, not this read.
[tool-responses.md](references/tool-responses.md) carries the rendering template both blocks follow.

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
`executor_id`, and the timing fields vanish from a row rather than reading as null. Never report `failed` without naming
the succeeded count. A remote batch carries no roll-up label and no `canceled` flag, uses uppercase scheduler states,
reports its unit as `unit_path`, and carries its `progress` verdict per batch instead.

---

## Re-running failed jobs

The straightforward retry is to prepare the pipeline again and execute the new batch, queuing only what is outstanding.

1. **Identify the failures.** Call `get_processing_status_tool` with `status_filter="failed"`, `include_items=True`,
   and `detailed=True`, then read each row's `error_message`. For a remote batch, follow the `output_log` and
   `error_log` paths the detailed row carries to the allocation's own diagnostics on the server.

2. **Decide between reset and clean.** Use `reset_processing_jobs_tool` where the failure cause was external and the
   partial output is harmless. Use `clean_processing_output_tool` where the output itself is suspect, remembering that
   a `checksum` clean removes only the tracker, while a `forging` clean removes the whole assembled dataset and the
   cross-recording directory that dataset owns inside every source session it names.

3. **Re-prepare and re-execute.** Call `prepare_batch_tool` again for the same pipeline and units, confirm the new
   `total_jobs` matches the work you intended to retry, then call `execute_jobs_tool` with the new identifier.

A job reported blocked is not a failure and never reached an execution backend. A run blocks a job when it can neither
queue the upstream stage nor confirm that it already succeeded, and blocking propagates to the dependents. Run the
upstream stage first, then execute the dependents again.

---

## Error routing

Errors carry `success: false` and a human-readable `error` string alone, with no error code and no structured detail,
so match on `success` and read the string. The one exception is `No valid jobs to execute.`, which also carries
`invalid_jobs`. Two rejections are shared, one by every tool taking a pipeline and one by every tool taking a host.

```text
Unsupported batch pipeline '{pipeline}'. Available: checksum, forging, microcontroller, runtime, two_photon, video.
Unsupported host '{host}'. Available: local, remote.
```

| Message                                                                                                                                                                      | Tool                           | Remedy                                                                                                                                                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Unable to prepare the {host} '{pipeline}' batch. {exception}`                                                                                                               | `prepare_batch_tool`           | Read the wrapped exception, which names the missing plan table, the multi-project batch, or the absent server configuration                                                                                                                     |
| `Unable to read the prepared batches {batch_ids}. {exception}`                                                                                                               | `execute_jobs_tool`            | A registry file is unreadable or malformed. Re-list, then re-prepare                                                                                                                                                                            |
| `No prepared batch exists for identifier(s) {missing}. Prepare the pipeline again to register its jobs.`                                                                     | `execute_jobs_tool`            | Recover the identifier with `list_prepared_batches_tool`                                                                                                                                                                                        |
| `No batch was named.`                                                                                                                                                        | `execute_jobs_tool`            | The identifier list was empty                                                                                                                                                                                                                   |
| `Unable to execute batches prepared against the hosts {hosts} together. …`                                                                                                   | `execute_jobs_tool`            | Dispatch one host's batches at a time                                                                                                                                                                                                           |
| `No dispatchable jobs. Every prepared job is blocked or already succeeded.`                                                                                                  | `execute_jobs_tool`            | Read the preparation's `blocked_jobs` list and run the upstream stage                                                                                                                                                                           |
| `No valid jobs to execute.`                                                                                                                                                  | `execute_jobs_tool`            | The response carries `invalid_jobs`, each entry naming the field that failed                                                                                                                                                                    |
| `A batch is already running in this process. Wait for it to finish or cancel it before starting another.`                                                                    | `execute_jobs_tool`            | One local pool holds one run. Wait, or cancel                                                                                                                                                                                                   |
| `Unable to clear the recorded state of the batch's jobs. {exception}`                                                                                                        | `execute_jobs_tool`            | A concurrent writer held a unit's tracker lock past the timeout, so nothing was dispatched and no record was cleared. Retry once that writer finishes                                                                                           |
| `Unable to submit the remote batch to the compute server's scheduler. …` and `Unable to cancel the allocations the named batches hold …`                                     | execute and cancel             | The submit message means the scheduler rejected a job; every allocation it accepted first stays queued and recorded, so re-running the batch submits what it left. The cancel one means nothing was cancelled and every batch stays outstanding |
| `Unable to read this machine's submission ledger, …` and `Unable to drop the named batches from this machine's submission ledger. …`                                         | status and retire              | This machine's ledger file or its lock. The drop message states that remediating again is safe                                                                                                                                                  |
| `Unable to reach the remote compute server, …`, `Unable to read what the named batches' jobs recorded …`, and `Unable to read the state of the named batches' allocations …` | status and retire              | Restore the connection, the host's state artifacts, or scheduler accounting, then call again                                                                                                                                                    |
| `Unable to retire the batches every allocation of which resolves to a plain drop from the submission ledger. …`                                                              | status and cancel              | The closure that runs inside the call could not write. Call again once the host answers                                                                                                                                                         |
| `Unable to cancel the allocations the scheduler still holds, …` and `Unable to return the stranded jobs to the scheduled state, …`                                           | `retire_remote_batches_tool`   | Nothing was changed and no flag waives either one. Retry once the server answers                                                                                                                                                                |
| `Unable to report the named batch(es) {uncovered}. …` and `No outstanding remote batch has identifier(s) {unknown}. …`                                                       | status, cancel, and retire     | A live local run answers only for the batches it covers, and a remote call only for the outstanding ones                                                                                                                                        |
| `Unknown status '{status_filter}'. Available: failed, running, scheduled, succeeded.` locally, `Unknown scheduler state '{status_filter}'. Available: {states}.` remotely    | `get_processing_status_tool`   | Local labels are lowercase, remote scheduler states are uppercase                                                                                                                                                                               |
| `Unable to read the recorded outcomes of {named}. {exception}`                                                                                                               | `get_processing_status_tool`   | An outcome file is unreadable. Re-list the registry                                                                                                                                                                                             |
| `No batch is running.`                                                                                                                                                       | `cancel_processing_tool`       | Nothing is live in this process                                                                                                                                                                                                                 |
| `No remote batch is outstanding. …`                                                                                                                                          | cancel and retire              | Read the recorded outcome instead of the outstanding listing                                                                                                                                                                                    |
| `Unable to remediate a batch without an identifier. …`                                                                                                                       | `retire_remote_batches_tool`   | Name the batches. Remediation never defaults to the whole ledger                                                                                                                                                                                |
| `Unable to remediate the named remote batch(es). {n} of their allocation(s) resolve as 'running' …`                                                                          | `retire_remote_batches_tool`   | Wait, cancel through `cancel_processing_tool`, or pass `force=True` to cancel each one first                                                                                                                                                    |
| `Unable to snapshot what the jobs of {batches} recorded. …` and `Unable to reach the remote compute server, so neither the state … nor what their jobs recorded …`           | `retire_remote_batches_tool`   | Restore server access and remediate again, or pass `drop_without_outcome=True` to drop the entries regardless                                                                                                                                   |
| `The named batches hold no allocation to cancel.`                                                                                                                            | `cancel_processing_tool`       | Confirm the identifiers through a remote status read first                                                                                                                                                                                      |
| `Unable to reset the {host} '{pipeline}' jobs. {exception}` and `Unable to clean the {host} '{pipeline}' output. {exception}`                                                | reset and clean                | Read the wrapped exception, which for a remote call names the failing server-side command                                                                                                                                                       |
| `A batch is currently running in this process. Wait for it to finish or cancel it before cleaning output.`                                                                   | `clean_processing_output_tool` | The guard covers this process's pool alone                                                                                                                                                                                                      |
| `No prepared batch has identifier(s) {unknown}. Held: {held}.`                                                                                                               | `list_prepared_batches_tool`   | The message enumerates what the registry does hold                                                                                                                                                                                              |
| `Unable to read the prepared batches under '{directory}'. {exception}`                                                                                                       | `list_prepared_batches_tool`   | The registry directory is unreadable                                                                                                                                                                                                            |
| `Unable to resolve the prepared-batch directory. {exception}`                                                                                                                | list and forget                | The platform working directory does not resolve, so neither tool reaches the registry. `/forging-mcp-environment-setup` owns that check                                                                                                         |
| `Unable to forget a batch without an identifier. …`                                                                                                                          | `forget_prepared_batches_tool` | Name the batches, since there is no wildcard                                                                                                                                                                                                    |
| `Unable to forget the batches {batch_ids} under '{directory}'. {exception}`                                                                                                  | `forget_prepared_batches_tool` | Removal was partial and its report is lost. Re-list to see what remains                                                                                                                                                                         |

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
| `/remote-execution`                        | Adjacent: the ledger, the resolved verdicts, and scheduler job reads              |
| `/processing-results`                      | Downstream: output layouts and how to interpret them                              |
| `/project-state`                           | Downstream: the project manifest and job artifacts a batch refreshes              |
| `/cli-reference`                           | Reference: the human-facing `slf` command surface                                 |
| `/pipeline`                                | Context: where batch processing sits in the end-to-end workflow                   |
| `mesoscope:mesoscope-vr-processing-schema` | Reference: Mesoscope-VR file names and columns, and the per-pipeline skill map    |

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
- [ ] Every stalled_batch_ids entry on a remote read was remediated rather than waited on, with no file edited by hand

Batch workflow, agent-judged:
- [ ] Pipeline and host confirmed with the user before preparation
- [ ] Resource budgets confirmed with the user before execution
- [ ] Blocked jobs distinguished from failed jobs in every report
- [ ] Failed jobs retried through reset or clean, with the destructive cost of clean stated first
- [ ] Output verified through /processing-results before the batch is called complete
```
