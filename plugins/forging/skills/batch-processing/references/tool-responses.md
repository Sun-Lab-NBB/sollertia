# Batch tool responses

The complete return-key tree of the eight tools `/batch-processing` owns, with every conditional key and the condition
that produces it. Read this alongside the parameter tables in that skill.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

Two reading rules govern every tree below. A per-item listing is projected, so a field whose value is `None`, `""`,
`[]`, or `{}` is dropped from that row entirely while `0` and `False` survive. Every summary key is computed over the
whole set and ignores every filter, so only `jobs`, `batches`, `rows`, and `matched_rows` respond to one. Where a tree
names the paging group, it stands for `rows`, `matched_rows`, `start_row`, and `next_start_row`, merged at the top
level of the response rather than nested under the listing.

---

## prepare_batch_tool

```text
success              bool, always true on this branch
pipeline             str, the prepared pipeline, read from the recorded document
host                 str, "local" or "remote", read from the resolved execution host
units[]              raw entries, not projected, in one of two shapes:
  resolved:
    unit_path        str
    unit_name        str
    job_count        int, the dispatchable jobs this unit contributed
    blocked_count    int
  unresolved:
    unit_path        str
    unit_name        str
    error            str, either the no-state reason or the unplanned reason
    job_count        0, and NO blocked_count key at all
total_units          int, the units NAMED, not the units that produced jobs
total_jobs           int, the dispatchable jobs only
total_blocked_jobs   int
blocked_jobs[]       projected, one entry per job this run could neither queue nor find already succeeded:
  job_id             str
  pipeline           str
  job_name           str
  specifier          str, omitted when empty
  unit_name          str
  unsatisfied_prerequisite_ids[]   list of str
batch_id             str, 16 lowercase hexadecimal characters
jobs[]               ONLY when include_job_descriptors=True. Raw descriptors, not projected, so every key is
                     present even when empty: job_id, job_name, specifier, status, executor_id, unit_path,
                     unit_name, pipeline, tracker_path, cores, memory_mb, prerequisite_ids, options
```

`status` on a raw descriptor is a tracker status MEMBER NAME, uppercase, unlike the lowercase labels a status read
returns. `tracker_path` is empty on every descriptor of a remotely prepared batch, by design.

---

## execute_jobs_tool

A local dispatch and a remote dispatch return different trees. Branch on `host`.

```text
success              bool
started              true, literally, on every success. It proves dispatch and nothing about outcomes
host                 "local"
total_jobs           int, the dispatched count
core_budget          int, the RESOLVED figure rather than the argument
memory_budget_mb     int, the RESOLVED figure rather than the argument
pool_size            int, the worker processes the pool opened
pipelines[]          sorted list of the dispatched jobs' pipelines
adopted_jobs         [], always empty locally, because a local run never adopts a live job
job_allocations{}    one entry per job TYPE in the batch, not projected, so nulls are present:
  <job_name>:
    cores_per_job              int, the cap applied to an over-wide job of this type
    maximum_parallel           int
    concurrency_limit          int or null, always present
    concurrency_reservation    int or null, always present
invalid_jobs[]       ONLY when a descriptor failed to build: job_id, pipeline, unit_path, error
```

```text
success              bool
started              true
host                 "remote"
batch_id             str, batch_ids[0], which names the server-side script and log directory
batch_ids[]          echoed exactly as the caller passed them, order preserved
total_jobs           int, the SUBMITTED count. Adopted jobs are NOT counted here
walltime_minutes     int, the RESOLVED figure, 480 when the caller passed a non-positive value
pipelines[]          sorted list over the submissions
batch_directory      str, a path ON THE SERVER
adopted_jobs[]       sorted by unit_path then job_id: unit_path, job_id, slurm_job_id
submissions[]        in the order the scheduler accepted them: job_id, slurm_job_id, job_name
invalid_jobs[]       ONLY when a descriptor failed to build, same shape as the local tree
```

A remote run's real size is `total_jobs` plus the length of `adopted_jobs`. Submitting a remote batch also closes and
retires any earlier batch whose allocations settled in the meantime, so unrelated batches leave the outstanding listing
as a side effect of this call.

---

## get_processing_status_tool

Four trees. The host chooses the reader, and the local reader chooses among three shapes by whether a run is installed
in this process and whether the named batches have recorded outcomes.

**Local, with a live run installed.**

```text
success              bool
active               bool, true while the manager thread is alive
canceled             bool
status               str, the roll-up label: failed, completed, processing, not_started, or in_progress
summary
  total              int, the batch's own dispatched job count
  scheduled          int
  running            int
  succeeded          int
  failed             int
breakdown            one entry per axis, each either a value-to-count map or an elision marker carrying
                     distinct_values and elided when the axis holds more than 50 distinct values:
  pipeline, job_name, status, session_path
blocked_jobs         int, a COUNT rather than a list. ONLY when the run blocked jobs
blocked_reason       str. Same condition. Names the remedy and says to list them by filtering to scheduled
jobs[]               ONLY when a filter is named or include_items=True. Projected:
  job_id, pipeline, job_name, specifier, status, session_path
  detailed adds:     cores, memory_mb, elapsed_seconds, executor_id, error_message, started_at,
                     completed_at, options, prerequisite_ids, tracker_path
<paging group>       same condition as jobs
```

The per-job unit key is `session_path` even for a dataset unit, and the matching filter parameter is `session_paths`,
so a `forging` batch is filtered by dataset root through a parameter named for sessions. `status` here is a lowercase
tracker label. `started_at` and `completed_at` are microsecond-precision epoch integers, and `elapsed_seconds` is
rounded to three decimals and measured to the current moment while a job still runs.

**Local, no live run, outcomes found.**

```text
success              bool
active               false
batch_ids[]          the identifiers named, or the last run's identifiers when none were named
outcomes[]           one entry per named identifier that HAS an outcome file, in the order named
message              str, the closed-batch message, stating that the counts come from the recorded closure
```

No `status`, no `summary`, no `breakdown`, no `jobs`, and no paging keys appear on this branch, and every filter
argument is ignored.

**Local, no live run, no outcomes.**

```text
success              bool
active               false
message              "No batch is running in this process."
```

This is also what a restarted MCP server returns while outcome files sit on disk, because the run state is held in
process. Recover by calling `list_prepared_batches_tool` and then naming `batch_ids` explicitly.

**The recorded outcome**, the entry shape `outcomes[]` carries on either host.

```text
batch_id             str
pipeline             str
host                 str
total                int, the dispatched jobs PLUS the preparation-blocked jobs
succeeded            int
failed               int
blocked              int, preparation-blocked jobs plus jobs whose prerequisite failed during the run
outstanding          int, dispatched jobs that neither succeeded nor failed and wait on nothing that failed
complete             bool, a strict equality of succeeded against total, which includes blocked jobs
failed_jobs[]        capped at 50 entries: job_id, job_name, specifier, unit_name, error_message
blocked_jobs[]       capped at 50 entries in total: job_id, job_name, specifier, unit_name,
                     unsatisfied_prerequisite_ids
snapshot_paths[]     LOCAL paths where the state artifacts were mirrored at closure
verified_at          int, microsecond-precision epoch
```

Both lists are capped while the integer counts always cover the whole batch, so never infer a count from a list length.
A batch that succeeded on everything it dispatched still reports `complete: false` where it carried blocked jobs.

**Remote.**

```text
success              bool
batches[]            one entry per covered batch, in ledger order:
  batch_id           str
  submitted_at       int, microsecond-precision epoch
  pipelines[]        sorted list over the batch's submissions
  batch_directory    str, a path ON THE SERVER
  total_jobs         int
active               bool, true while any allocation is in a non-terminal state
summary
  total              int
  <scheduler state>  int, ONLY the states actually observed, sorted by key
breakdown            never elided on this branch:
  batch_id, pipeline, job_name, status, unit_path
outcomes[]           ALWAYS present, empty when nothing settled on this call. Same shape as above
jobs[]               ONLY when a filter is named or include_items=True. Projected:
  batch_id, job_id, slurm_job_id, pipeline, job_name, specifier, status, unit_name
  detailed adds:     cores, memory_mb, slurm_job_name, unit_path, output_log, error_log
<paging group>       same condition as jobs
```

There is no roll-up `status` label, no `canceled` flag, and no `blocked_jobs` count on this branch. A blocked
allocation appears as a job whose `status` reads `BLOCKED`. `output_log` and `error_log` are paths on the server and
are the route to a failed allocation's own diagnostics. Where the ledger holds nothing outstanding, the response is
`success` plus `active: false` plus a message, and carries no other key.

---

## cancel_processing_tool

```text
success              bool
canceled             true
dropped_jobs         int, the queued jobs cleared. LOCAL branch only
message              str. Locally it states that in-flight jobs will finish and queued jobs were dropped
canceled_jobs        int. REMOTE branch only. Counts every allocation the call NAMED, including
                     allocations that had already finished, so it is not a count of work killed
batch_ids[]          REMOTE branch only. The batches the cancellation covered
```

A remote cancel re-queries the scheduler and closes the settled batches before returning, so the covered batches leave
the ledger and their outcomes become the only remaining answer about them.

---

## reset_processing_jobs_tool

```text
success              bool
pipeline             str, the argument echoed
host                 str, the argument echoed
total_units          int, the length of unit_paths AS PASSED, before repeated paths are de-duplicated
jobs_reset           int when job_ids was truthy, otherwise null. It counts the identifiers NAMED, not the
                     records cleared, and is not multiplied by the unit count
message              str, one of exactly two strings, naming either every job each unit tracks or the named
                     identifiers each unit tracks with the ones it does not track skipped
```

---

## clean_processing_output_tool

```text
success              bool
pipeline             str, echoed
host                 str, echoed
removed[]            in removal order, unit by unit, tracker before owned directory within each unit:
  path               str
  removed_bytes      int
total_paths          int, the length of removed. PATHS, not units. A unit contributes 0, 1, or 2 entries
removed_bytes        int, the sum over every entry
```

A `checksum` unit contributes at most one entry, its tracker, because that pipeline owns no output directory and its
stored value stays in place. A `forging` unit's directory entry covers the whole assembled dataset hierarchy. Remote
figures are parsed from the server-side command's own output, so an unparseable line is dropped and the reported total
under-counts.

---

## list_prepared_batches_tool

```text
success              bool
batch_directory      str, the LOCAL registry path. There is no remote registry
total_batches        int, over ALL records, ignoring every filter
total_jobs           int, over ALL records, ignoring every filter
breakdown            over ALL records, ignoring every filter, never elided:
  pipeline, host
batches[]            ALWAYS present, projected, in ascending identifier order:
  batch_id           str
  pipeline           str
  host               str
  unit_count         int. ABSENT for a settled batch, whose document closure retired
  job_count          int. The prepared job count, or the outcome's total for a settled batch
  blocked_count      int. The prepared blocked count, or the outcome's blocked figure
  outcome_recorded   bool. Both values survive projection
  detailed adds, for PREPARED records only:
    options          dict, omitted when empty
    job_names        a job-name-to-count map
    unit_names[]     only the units whose entry carries a unit name
<paging group>
```

A settled record is one whose prepared document closure deleted, leaving the outcome. It carries no `unit_count` and
reports none of the detail fields even under `detailed=True`. The listing spans every batch the registry holds,
including batches prepared by earlier server sessions, which is what makes it the recovery path for a lost identifier.

---

## forget_prepared_batches_tool

```text
success              bool
batch_directory      str, the LOCAL registry path
forgotten[]          the identifiers this host actually held and removed, IN CALLER ORDER
total_forgotten      int
unknown[]            the named identifiers that held neither file, SORTED and DE-DUPLICATED
```

The two lists carry different ordering guarantees. A second call for the same identifier returns success with an empty
`forgotten` list and the identifier under `unknown`. A partial removal that then raises returns the error response
instead, losing the report of what was already removed, so re-list to establish what the registry still holds. The
closure snapshot directory a settled batch left behind is never removed by this call.
