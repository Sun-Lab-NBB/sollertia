---
name: project-state
description: >-
  Documents the project state artifacts of sollertia-forgery, the session manifest and the job table published
  beside it, covering their exact column schemas, the one call that writes both, the three widening reads that
  query them, and the query order that establishes whether a batch of jobs succeeded. Use when generating or
  reading a project manifest, when asking which sessions finished a pipeline, when investigating why a job
  failed, or when verifying the outcome of a processing batch.
user-invocable: false
---

# Project state

Documents the two project state artifacts, the session-rowed manifest and the job-rowed table published beside it,
covering their exact column schemas, the single call that writes both, the three widening reads that query them, the
generation tracker, and the query order that establishes whether a batch of jobs succeeded.

This skill is the **exclusive** owner of `generate_project_manifest_tool`, `read_project_manifest_tool`,
`read_project_jobs_tool`, and `get_manifest_status_tool`. No other skill in the marketplace may document or call these
four tools.

---

## Scope

**Covers:**
- The manifest and the job artifact, their paths, and the one generation call that writes both under one lock
- The complete manifest schema, one row per session, every column with its Polars dtype
- The complete job artifact schema, one row per tracked per-session job, every column with its Polars dtype
- The animal and session key that joins the manifest, the job table, the plan projection, and dataset state
- Filters, breakdown axes, and detail fields of both read tools, and the paths they report for a remote project
- The generation tracker, its single `manifest_generation` job, and the two status vocabularies its reader reports
- The query order that establishes whether a batch succeeded, which is this plugin's answer to output verification

**Does not cover:**
- Preparing, executing, canceling, resetting, and cleaning processing jobs. Owned by `/batch-processing`.
- The plan projection, the resource model, and job sizing. Owned by `/job-planning`.
- Forging job state, which each dataset records on its own artifact. Owned by `/dataset-definition`.
- The per-session files each pipeline writes under `processed_data`. Owned by `/processing-results`.
- The scheduler's own record of a submitted job. Owned by `/remote-execution`.
- The stored access record a remote host resolves through. Owned by `/server-configuration`.
- The `slf manifest` command surface a user runs by hand. Owned by `/cli-reference`.
- Every acquisition-system-specific file, column, and session type. Owned by `mesoscope:mesoscope-vr-processing-schema`.

**Handoff rules:** a question about what a running batch is doing right now goes to `/batch-processing`, whose status
tool reads the live trackers. A question about what a finished batch left behind is answered here, because both
artifacts are snapshots taken between execution graphs. A question about the contents of one written feather is analysis
Python rather than an MCP call, and `/processing-results` owns the file layout that Python reads.

---

## Agent requirements

- Every tool here takes the project's root data directory as an absolute path. `assets:project-hierarchy` enumerates the
  projects under the data root, and `assets:working-directory` owns the data root itself.
- Both read tools read a stored snapshot and never rescan the session hierarchy, so regenerate before reading whenever
  jobs have run since the last generation.
- Generation is a whole-project operation. It takes no session list, no animal filter, and no pipeline argument.
- Never round-trip the `project_path` a read reports back into a tool as a server path, because a remote read reports
  the local mirror. Take server paths from `discover_remote_project_tool`, which `/remote-execution` owns.

> The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
> documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## Available tools

### Generation tool

| Tool                             | Purpose                                                                  |
|----------------------------------|--------------------------------------------------------------------------|
| `generate_project_manifest_tool` | Rebuilds the manifest and the job artifact from one walk of the sessions |

**Parameters:**

| Parameter      | Type  | Default    | Description                                                                    |
|----------------|-------|------------|--------------------------------------------------------------------------------|
| `project_path` | `str` | (required) | Absolute path to the project root data directory, a server path under `remote` |
| `host`         | `str` | `"local"`  | `"local"` for this machine, `"remote"` for the configured compute server       |

**Return structure:**
```text
project_path:            Project root the generation ran against
manifest_path:           {project_root}/{project_stem}_manifest.feather
host:                    "local" or "remote"
jobs_path:               {project_root}/{project_stem}_jobs.feather
total_jobs:              Rows the job artifact holds, 0 when the file is absent
elapsed_seconds:         Generation wall time in seconds, rounded to the millisecond
total_sessions:          Manifest row count                                     | local only
total_animals:           Distinct animal identifiers                            | local only
animals:                 Natural-sorted list of animal identifiers              | local only
session_types:           Sessions counted per session type                      | local only
acquisition_systems:     Sessions counted per acquisition system                | local only
complete_count:          Sessions whose complete column holds 1                 | local only
pipeline_status_counts:  {pipeline: {done, not_done}} per pipeline identifier   | local only
columns:                 Manifest column names                                  | local only
total_rows:              Identical to total_sessions                            | local only
```

A remote generation omits the summary block, because the manifest stays on the server, and it counts `total_jobs` off
the server. `pipeline_status_counts` is keyed by the pipeline identifier rather than by the manifest column, so the
checksum pipeline's entry is keyed `checksum` while the column it counts is `integrity`.

**One call writes two artifacts.** `managing.manifest.generate_project_manifest` walks the sessions once, holds one lock
on `{project_stem}_manifest.feather.lock`, and publishes the job artifact through `write_project_jobs` first, then the
manifest. Both renames are atomic but ordered, and readers take no lock, so a reader landing between them at worst holds
job rows for a session the manifest does not list yet. A join on animal and session drops exactly those rows. The
reverse order would show a manifest row whose jobs are absent, which reads as a session nothing has processed.

Run generation between execution graphs. A snapshot taken while jobs run is already stale by the time it is read.

### Manifest read tool

| Tool                         | Purpose                                                            |
|------------------------------|--------------------------------------------------------------------|
| `read_project_manifest_tool` | Reads sessions out of the stored manifest in three widening stages |

**Parameters:**

| Parameter       | Type                    | Default    | Description                                                     |
|-----------------|-------------------------|------------|-----------------------------------------------------------------|
| `project_path`  | `str`                   | (required) | Absolute path to the project root, a server path under `remote` |
| `host`          | `str`                   | `"local"`  | `"local"` or `"remote"`                                         |
| `animal`        | `str / None`            | `None`     | Restricts the listing to one animal                             |
| `session_type`  | `str / None`            | `None`     | Restricts the listing to one session type, filtering `type`     |
| `system`        | `str / None`            | `None`     | Restricts the listing to one acquisition system                 |
| `pipeline_done` | `dict[str, int] / None` | `None`     | Maps a 0/1 column to `1` for done or `0` for outstanding        |
| `limit`         | `int / None`            | `None`     | Sessions to list. See the response contract                     |
| `start_row`     | `int`                   | `0`        | Match index at which the listing begins                         |
| `include_items` | `bool`                  | `False`    | Lists sessions when no filter is named                          |
| `detailed`      | `bool`                  | `False`    | Adds `notes` to each listed session                             |

**Return structure:**
```text
project_path:    Project root that was read, the local mirror for a remote project
manifest_path:   Absolute path to the manifest that was read
total_sessions:  Manifest row count, spanning the project regardless of the filters
breakdown:       Counts per held value for animal, type, system, complete, integrity,
                 runtime, microcontroller, video, and two_photon
sessions[]:      animal, session, session_path, date, type, system, complete, integrity,
                 runtime, microcontroller, video, two_photon    | filtered or include_items=True
  notes:         Free-text experimenter notes                   | requested by detailed=True
rows, matched_rows, start_row, next_start_row                   | alongside sessions
```

`pipeline_done` accepts a `dict` of `str` to `int`, keyed by any 0/1 column the breakdown names, meaning the five
pipeline columns and `complete`. Its entries merge into the same selector dictionary the three named filters use, so a
`pipeline_done` key of `animal`, `type`, or `system` overrides the dedicated parameter. An empty dict narrows nothing
and leaves the call bare.

The five pipeline columns are gross done indicators. They answer which sessions are ready and nothing finer, so route
every question about which job failed, and why, to `read_project_jobs_tool`.

### Job read tool

| Tool                     | Purpose                                                                         |
|--------------------------|---------------------------------------------------------------------------------|
| `read_project_jobs_tool` | Reads tracked per-session jobs out of the job artifact in three widening stages |

**Parameters:**

| Parameter       | Type               | Default    | Description                                                     |
|-----------------|--------------------|------------|-----------------------------------------------------------------|
| `project_path`  | `str`              | (required) | Absolute path to the project root, a server path under `remote` |
| `host`          | `str`              | `"local"`  | `"local"` or `"remote"`                                         |
| `animal`        | `str / None`       | `None`     | Restricts the listing to one animal                             |
| `session`       | `str / None`       | `None`     | Restricts the listing to one session                            |
| `pipelines`     | `list[str] / None` | `None`     | Restricts the listing to these pipelines                        |
| `job_names`     | `list[str] / None` | `None`     | Restricts the listing to these job type names                   |
| `status`        | `str / None`       | `None`     | Restricts the listing to one tracker status, such as `"FAILED"` |
| `limit`         | `int / None`       | `None`     | Jobs to list. See the response contract                         |
| `start_row`     | `int`              | `0`        | Match index at which the listing begins                         |
| `include_items` | `bool`             | `False`    | Lists jobs when no filter is named                              |
| `detailed`      | `bool`             | `False`    | Adds the executor, the timestamps, and the recorded error       |

**Return structure:**
```text
project_path:  Project root that was read, the local mirror for a remote project
jobs_path:     Absolute path to the job artifact that was read
total_jobs:    Artifact row count, spanning the project regardless of the filters
breakdown:     Counts per held value for animal, pipeline, job_name, and status
jobs[]:        animal, session, pipeline, job_name, specifier, status, job_id | filtered or include_items=True
  executor_id, error_message, started_at, completed_at                       | requested by detailed=True
rows, matched_rows, start_row, next_start_row                                | alongside jobs
```

`status` takes one uppercase member name. `pipelines` and `job_names` take lists and filter on membership, while
`animal`, `session`, and `status` filter on equality. The listing carries `job_id` before any detail is requested,
because that is the identifier a reset targets, so a listing omitting it would name no object for the follow-up call.

### Generation status tool

| Tool                       | Purpose                                                                            |
|----------------------------|------------------------------------------------------------------------------------|
| `get_manifest_status_tool` | Reports the outcome of the last state-artifact generation from the project tracker |

**Parameters:**

| Parameter      | Type  | Default    | Description                                                     |
|----------------|-------|------------|-----------------------------------------------------------------|
| `project_path` | `str` | (required) | Absolute path to the project root, a server path under `remote` |
| `host`         | `str` | `"local"`  | `"local"` or `"remote"`                                         |

**Return structure:**
```text
project_path:   Project root that was read, the local mirror for a remote project
tracker_path:   {project_root}/manifest_processing_tracker.yaml
manifest_path:  {project_root}/{project_stem}_manifest.feather
jobs_path:      {project_root}/{project_stem}_jobs.feather
exists:         {manifest: bool, jobs: bool}, exactly these two keys
status:         "not_started", "scheduled", "running", "succeeded", or "failed"
error_message:  Text the generation job recorded       | present only on a recorded failure
```

It reads the tracker alone and rescans nothing, so it answers whether the stored artifacts are trustworthy without
paying to rebuild them. The `exists` block reports the mirror for a remote project, so a project whose artifacts were
never generated on the server reports both flags false alongside a `not_started` status.

**The `manifest_generation` job.** The manifest pipeline registers exactly one job. `MANIFEST_JOB_NAME` holds the value
`manifest_generation`, and the specifier is the project directory's stem. The tracker entry is keyed by
`ProcessingTracker.generate_job_id` over that pair, so a mirror stored under a different directory name would resolve no
entry. `remote_state_directory` names the mirror after the project, which keeps the two identifiers equal.

---

## Recommended query order

**The server ships no output-verification tool and no feather-query tool.** Nothing on it opens a written feather and
reports its rows, and nothing walks an output directory to confirm that a file landed. Verification is the breakdown of
the job artifact, read in the order below, and the library itself verifies the same way, checking which pipelines
completed rather than counting output files.

1. **`generate_project_manifest_tool`**: Refresh both artifacts after the batch's last job retires. A snapshot taken
   before the batch reports the state the project held then, and carries no indication that it is stale.
2. **`read_project_jobs_tool`**: Call it bare. The `status` axis of the breakdown counts every tracked job in the
   project by outcome, which is the one number that answers whether the batch succeeded.
3. **`read_project_jobs_tool`** with `status="FAILED"` and `detailed=True`: List the failures with their
   `error_message`, `executor_id`, and timestamps. Follow `next_start_row` until it is null when a page fills.
4. **`read_project_manifest_tool`** with `pipeline_done` set to `0` for the batch's pipeline: List the sessions still
   outstanding, which covers a session that registered no job as well as one whose jobs failed.
5. **`get_manifest_status_tool`**: Run it when a read errors on a missing artifact, or when the counts disagree with
   what the batch reported, since a failed generation leaves the previous snapshot in place under the same names.

Reading the rows of an output feather is analysis Python against the paths `/processing-results` documents, never an MCP
call. Jobs still in flight belong to `get_processing_status_tool`, which `/batch-processing` owns, and the forging
pipeline's own jobs belong to `read_dataset_state_tool`, which `/dataset-definition` owns.

---

## Output data reference

### Artifact locations

```text
{project_root}/
├── {project_stem}_manifest.feather
├── {project_stem}_manifest.feather.lock
├── {project_stem}_jobs.feather
└── manifest_processing_tracker.yaml
```

Both table names derive from the project directory's stem rather than its name, through
`managing.manifest.project_manifest_path` and `managing.jobs.project_jobs_path`. The `.lock` sibling is the file lock
the generator holds while it writes, so it appears in a listing without being an artifact. Both tables are written as
uncompressed Arrow IPC so a reader memory-maps rather than decodes them, and both are published through `atomic_write`,
a temporary file renamed over the destination, because the readers that memory-map them take no lock of their own.

### Manifest schema

`managing.manifest._PROJECT_MANIFEST_SCHEMA`, one row per session, `natural_sort` ordered by animal then session:

| Column            | Dtype      | Description                                                                             |
|-------------------|------------|-----------------------------------------------------------------------------------------|
| `animal`          | `String`   | Animal identifier, text so every project artifact joins on it without a cast            |
| `date`            | `Datetime` | Acquisition time, timezone-aware UTC, null when the session name is not a timestamp     |
| `session`         | `String`   | Session name, which is the UTC acquisition timestamp string                             |
| `session_path`    | `String`   | `<animal>/<session>`, relative to the project root so it resolves against any data root |
| `type`            | `String`   | The session's `SessionTypes` value                                                      |
| `system`          | `String`   | The recording system's `AcquisitionSystems` value                                       |
| `notes`           | `String`   | Free-text experimenter notes read from the session descriptor                           |
| `complete`        | `UInt8`    | `1` when the descriptor's `incomplete` flag is false                                    |
| `integrity`       | `UInt8`    | `1` when the checksum pipeline finished                                                 |
| `two_photon`      | `UInt8`    | `1` when the two-photon pipeline finished                                               |
| `runtime`         | `UInt8`    | `1` when the acquisition runtime pipeline finished                                      |
| `microcontroller` | `UInt8`    | `1` when the microcontroller pipeline finished                                          |
| `video`           | `UInt8`    | `1` when the camera pipeline finished                                                   |

These thirteen columns are the whole schema, and none of them is list-typed. The checksum pipeline reports under
`integrity`, the one entry of `_PIPELINE_STATUS_COLUMNS` whose column name differs from its pipeline identifier, and
every other pipeline's column name equals its identifier. `_assert_status_column_coverage` runs at import, so a pipeline
added to `SESSION_PIPELINES` without a status column fails the import rather than dropping out of the table.

### Job artifact schema

`managing.jobs._PROJECT_JOBS_SCHEMA`, one row per tracked job, `natural_sort` ordered by animal, session, pipeline, job
name, and specifier, with nulls last:

| Column          | Dtype    | Description                                                                          |
|-----------------|----------|--------------------------------------------------------------------------------------|
| `animal`        | `String` | Animal that performed the session recording this job                                 |
| `session`       | `String` | Name of the session recording this job                                               |
| `pipeline`      | `String` | The `ProcessingPipelines` value that produced the job                                |
| `job_id`        | `String` | Hexadecimal tracker registry key, derived from the job name and the specifier        |
| `job_name`      | `String` | Stage identifier within the pipeline                                                 |
| `specifier`     | `String` | Separates instances of one stage within a session, empty when the stage carries none |
| `status`        | `String` | The `ProcessingStatus` member name                                                   |
| `executor_id`   | `String` | Executor label, a scheduler job id or a process id, null when none was recorded      |
| `error_message` | `String` | Failure text, null for a job that recorded no failure                                |
| `started_at`    | `UInt64` | UTC microsecond epoch at which the job started running, null when it never started   |
| `completed_at`  | `UInt64` | UTC microsecond epoch at which the job succeeded or failed, null otherwise           |

The artifact holds the five per-session pipelines only, which are `checksum`, `runtime`, `microcontroller`, `video`, and
`two_photon`. A `pipelines=["forging"]` filter therefore fails the value check rather than returning an empty page,
because forging jobs live on each dataset's own state artifact and the generation job lives on the manifest tracker.

`job_id` also joins this table to the plan projection `/job-planning` owns, which carries the prerequisite identifiers
from which a scheduler builds its dependency graph.

### The join key

`animal` is a `String` column in the manifest, in the job artifact, in the plan projection, and in each dataset's state
table. A reader joins any pair of them on the animal and session key without casting either side, and the natural sort
that orders all four puts animal 2 before animal 10 rather than after it. Treat an animal identifier as text everywhere,
including as a filter value, since a numeric literal will not match the stored column.

### Generation tracker

`manifest_processing_tracker.yaml` at the project root records the generation itself, carrying the single
`manifest_generation` job. Generation calls `start_job` inside the lock, `complete_job` once the manifest lands, and
`fail_job` with the exception text before re-raising on any failure, which is what leaves `get_manifest_status_tool`
able to report the outcome afterwards.

---

## Interpretation guidance

### What a pipeline column means

A pipeline column holds `1` when that pipeline's session tracker rolls up to `TrackerStatus.COMPLETED`, which is to say
when every job the tracker registered succeeded. It holds `0` in every other case, including a tracker holding no jobs
and a tracker file that does not exist, both of which resolve to `not_started` rather than to an in-progress label. A
pipeline that resolved any job registers all of them before dispatching any, so an empty registry means never run.

The six 0/1 columns are independent observations, and none of them gates another. `complete` reports the descriptor's
own flag and gates no pipeline column. `integrity` reports the checksum pipeline alone and gates none of the other four.
A session carrying `complete=0`, `integrity=0`, and `video=1` is a legitimate row, and it is the row an orchestrator
needs, because suppressing processing state for an incomplete session would make an unprocessable session
indistinguishable from an unprocessed one.

### Status vocabularies

| Field                                                 | Vocabulary                                     | Values                                                       |
|-------------------------------------------------------|------------------------------------------------|--------------------------------------------------------------|
| `status` in the job artifact, and the `status` filter | `ProcessingStatus` member name                 | `SCHEDULED`, `RUNNING`, `SUCCEEDED`, `FAILED`                |
| `status` from `get_manifest_status_tool`              | lowercased member name, or one `TrackerStatus` | `scheduled`, `running`, `succeeded`, `failed`, `not_started` |

The generation status tool reports `not_started` from `TrackerStatus` when the tracker file is absent, and equally when
the tracker holds no entry under the `manifest_generation` identifier. Every other case reports a lowercased
`ProcessingStatus` name, so a finished generation reports `succeeded`. The `TrackerStatus` values `in_progress`,
`processing`, and `completed` never reach a caller of this tool, though `completed` is what drives the 0/1 columns.

### What an absence means

| Observation                                              | Legitimate meaning                                                        |
|----------------------------------------------------------|---------------------------------------------------------------------------|
| A session is absent from the manifest                    | Its `raw_data` directory is empty, so it is aborted or not yet acquired   |
| `date` is null on a manifest row                         | The session name does not follow the acquisition timestamp format         |
| `error_message` is absent from a listed job              | That job recorded no failure, so the artifact column is null for that row |
| `total_jobs` is `0` immediately after a generation       | No session in the project carries a tracker holding any job               |
| A pipeline column is `0` while its jobs read `SUCCEEDED` | The snapshot predates the last job retiring, so regenerate and read again |
| A remote read reports a path under `remote_state`        | The read mirrored the server's artifacts onto this machine and read those |

### Remote reads and the mirror

A generation with `host="remote"` runs on the server and leaves both artifacts there, reporting server paths. The
artifacts reach this machine the first time a read tool runs with `host="remote"`, which mirrors the project's tables
and its manifest tracker under the working directory's `remote_state` folder and reads that mirror.

Only the final component of `project_path` names the project on a remote read. A remote read never regenerates anything,
so pair `generate_project_manifest_tool(host="remote")` with the read that follows it whenever the server has run jobs
since the last generation. The mirror keeps the project's name, which is why one reader serves both hosts and why
`get_manifest_status_tool` resolves the same job identifier against it.

---

## Error routing

| Condition                            | Returned error                                                                                                |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------|
| `host` is neither label              | `Unsupported host '<host>'. Available: local, remote.`                                                        |
| The manifest is absent               | `No manifest exists at '<manifest_path>'. Generate it with generate_project_manifest_tool before reading it.` |
| The job artifact is absent           | `No job artifact exists at '<jobs_path>'. Generate it with generate_project_manifest_tool before reading it.` |
| A filter names an unknown column     | `Unknown column '<column>'. Available: <sorted columns>.`                                                     |
| A filter names a value nothing holds | `No session has '<column>' in <values>. Available: <values>.`, with subject `job` for the job read            |
| A remote read cannot mirror          | `Unable to read the <host> project. <exception>`                                                              |
| A local generation fails             | `Unable to generate the state artifacts for '<project_path>'. <exception>`                                    |
| A remote generation fails            | `Unable to generate the remote state artifacts for '<project_root>'. <exception>`                             |

Four underlying failures arrive wrapped in a generation error's trailing exception text:

| Underlying cause                       | What the wrapped exception says                                                                 |
|----------------------------------------|-------------------------------------------------------------------------------------------------|
| The project directory does not exist   | `The specified project directory does not exist.`                                               |
| The project holds no session data      | `The project directory does not contain any session data.`, plus the at-least-one-session floor |
| A session declares an unsupported type | `An unsupported session type '<type>' was encountered for session '<session>'.`                 |
| The manifest lock is held              | A `Timeout` on `<project_stem>_manifest.feather.lock` after 20 seconds                          |

A concurrent generation is the ordinary cause of that timeout. Wait for it rather than retrying at once, since the run
holding the lock is writing the same two artifacts.

Filter comparison is textual, so `pipeline_done={"video": 2}` returns the rejection
`No session has 'video' in ['2']. Available: ['0', '1'].` rather than an empty page. An empty page and a mistyped filter
would otherwise look identical to a caller.

---

## Related skills

| Skill                                      | Relationship                                                                |
|--------------------------------------------|-----------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`           | Prerequisite: server connectivity and the shared response contract          |
| `/batch-processing`                        | Upstream: runs the jobs whose outcomes both artifacts record                |
| `/job-planning`                            | Peer: owns the plan projection this table joins on `job_id`                 |
| `/dataset-definition`                      | Peer: owns the dataset state artifact that records forging jobs             |
| `/processing-results`                      | Reference: the per-session files a succeeded job leaves behind              |
| `/remote-execution`                        | Reference: server-side discovery and the scheduler's own job record         |
| `/cli-reference`                           | Reference: the `slf manifest` path a user runs by hand                      |
| `/pipeline`                                | Context: where project state sits in the end-to-end order                   |
| `assets:project-hierarchy`                 | Upstream: enumerates the projects and animals under the data root           |
| `assets:session-discovery`                 | Upstream: produces the session path lists a batch consumes                  |
| `experiment:data-management`               | Upstream: materializes the `raw_data` a session needs to enter the manifest |
| `mesoscope:mesoscope-vr-processing-schema` | Reference: the acquisition system's own file and column rosters             |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Project state, tool-settled (run generate_project_manifest_tool, read_project_jobs_tool, get_manifest_status_tool):
- [ ] sollertia-forgery MCP server is connected
- [ ] generate_project_manifest_tool ran after the batch's last job retired, and get_manifest_status_tool reports a
      succeeded status for the manifest_generation job
- [ ] The exists block reports true for both the manifest and the job artifact
- [ ] A bare read_project_jobs_tool call was made and its status breakdown read, and every FAILED job was listed with
      detailed=True and its error_message read
- [ ] read_project_manifest_tool with pipeline_done set to 0 for the batch's pipeline lists no session the batch was
      expected to finish

Project state, reader-judged:
- [ ] The five pipeline columns were read as independent 0/1 observations, with no session dismissed because complete
      or integrity holds 0
- [ ] Every join across the manifest, the job artifact, the plan, and dataset state used animal and session as text
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
