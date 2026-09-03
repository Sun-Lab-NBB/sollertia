# Batch tool responses

The complete return-key tree of the nine tools `/batch-processing` owns, with every conditional key and the condition
that produces it. Read this alongside the parameter tables in that skill.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

Two reading rules govern every tree below. A per-item listing is projected, so a field whose value is `None`, `""`,
`[]`, or `{}` is dropped from that row entirely while `0` and `False` survive. Every summary key is computed over the
whole covered set and ignores every per-job filter, so only `jobs`, `rows`, and `matched_rows` respond to one.
`batch_ids` on the remote status branch is the exception, since it narrows the covered batches themselves and therefore
narrows `active`, `summary`, `breakdown`, and `outcomes` along with `batches`. Where a tree names the paging group, it
stands for `rows`, `matched_rows`, `start_row`, and `next_start_row`, merged at the top level of the response rather
than nested under the listing.

---

## Rendering a live batch status

The template the `## Status formatting` section of `/batch-processing` renders, filled from a
`get_processing_status_tool` call with `include_items=True` and `detailed=True`.

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
withheld_jobs[]      ALWAYS present, empty on a run that withheld nothing. One entry per job this run neither
                     submitted nor adopted: unit_path, job_id, and the executor_id its tracker claims. A job
                     resolves here when its verdict is running but neither allocation is held, which is a tracker
                     recorded RUNNING under a non-slurm executor such as pid:, plus every job downstream of one.
                     Its tracker is NOT cleared, so it is withheld again on every later batch until
                     reset_processing_jobs_tool returns it to SCHEDULED
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
active               bool, true while the manager thread is alive. False here is TERMINAL: the thread ended
                     and closure recorded no outcome, so the state was never released
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

A run reaches this shape with `active: false` when every closure raised or its batches had already been retired, so the
state stayed installed and no `outcomes` branch was ever taken. Treat it as a finished run whose closure recorded
nothing: its `summary` carries `total`, `scheduled`, `running`, `succeeded`, and `failed` but no `blocked` or
`outstanding`, and the batches are confirmed through `list_prepared_batches_tool` and its `outcome_recorded` flag.

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

**Remote.** This branch resolves before it reports. It reads the ledger, connects, regenerates and reads each covered
project's state artifacts, reads scheduler accounting and then the scheduler queue, resolves every covered allocation
ONCE, closes each batch whose entries ALL prescribe the plain `drop` remediation, and re-reads the ledger to learn what
that closure left, so a batch that closed on this call is reported as closed rather than as outstanding. The closure is
a derivation from the verdicts below rather than a second reading, so nothing closes that this response reports as
still held.

```text
success              bool
batches[]            one entry per STILL-OUTSTANDING batch, in ledger order. NOT projected, so every key
                     below is present even when empty:
  batch_id           str
  submitted_at       int, microsecond-precision epoch
  outstanding_seconds  float rounded to three decimals, or null when submitted_at is not positive
  pipelines[]        sorted list over the batch's submissions
  batch_directory    str, a path ON THE SERVER
  total_jobs         int
  progress           str, the batch verdict: progressing, stalled, or awaiting_closure
  verdicts{}         one count per allocation verdict this batch holds, keyed by the verdict
  live_allocations   int, the allocations resolving to the running verdict
  running_allocations[]           those allocations' ids, capped at 50
  stranded_allocation_count       int, over the WHOLE batch
  stranded_allocations[]          those allocations' ids, capped at 50
  unresolvable_allocation_count   int, the allocations whose scheduler_state is gone, WHOLE batch
  unresolvable_allocations[]      those allocations' ids, capped at 50
  remedy             str, the instruction for this verdict. It names a tool call on every branch and
                     quotes 'slf server retire-batch' on the stalled branch ALONE
stalled_batch_ids[]  ALWAYS present, the batch_id of every entry whose progress reads stalled
uncovered_batch_ids[]  ALWAYS present, the batch_id of every batch another process recorded WHILE this
                     read ran. Named rather than resolved, since the records gathered predate them
active               bool, any(verdict == running) over every covered allocation, so it is set by a held
                     TRACKER CLAIM, or by a tracker running under a NON-slurm executor, even when every
                     recorded allocation reads settled or gone
summary
  total              int
  <scheduler state>  int, ONLY the accounting states actually observed, sorted by key
breakdown            never elided on this branch:
  batch_id, pipeline, job_name, status, scheduler_state, tracker_status, verdict, unit_path
outcomes[]           ALWAYS present, empty when nothing settled on this call. Same shape as above
scheduler_read_error str, ALWAYS present. Empty unless a scheduler record could not be read, in which
                     case every allocation resolves held
message              str, present when no covered batch remains outstanding, which this call's own
                     closure normally caused, and when uncovered_batch_ids is non-empty
jobs[]               ONLY when a filter is named or include_items=True. Projected:
  batch_id, job_id, slurm_job_id, pipeline, job_name, specifier, status, scheduler_state,
  tracker_status, verdict, remediation, unit_name
  detailed adds:     queued, tracker_executor_id, claimed_allocation, claim_state, cores, memory_mb,
                     slurm_job_name, unit_path, output_log, error_log
<paging group>       same condition as jobs
```

`status` is the accounting state, uppercase, while `scheduler_state`, `verdict`, and `remediation` are the resolved
lowercase values `/remote-execution` defines. `tracker_status` is a ProcessingStatus MEMBER NAME and is dropped from a
row when the state artifact holds no row for the job, as `specifier`, `claimed_allocation`, and `claim_state` are when
empty, while `queued: false` survives projection. There is no roll-up `status` label, no `canceled` flag, and no
`blocked_jobs` count here; a blocked allocation appears as a job whose `status` reads `BLOCKED`. Where the ledger holds
nothing outstanding at all, the response is `success` plus `active: false` plus a message and carries no other key.

`progress` rests on the resolved verdicts and never on `outstanding_seconds`, which is reported beside it. `active` is
one flag over every allocation the call covered, and it counts the `running` VERDICT rather than the `held`
`scheduler_state`, so a job whose tracker claims a held allocation, or one recorded as running under an executor that
names no scheduler allocation, sets it even where the allocation this ledger recorded reads `settled` or `gone`. A
stalled batch holds no `running` verdict and so never sets it, while another covered batch that does sets it whatever
the stalled batch reads. Branch on `stalled_batch_ids` rather than on `active`, and clear each named batch with
`retire_remote_batches_tool`.

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

A remote cancel issues the cancellation first, then resolves every named batch exactly as the remote status branch
does and closes each one whose entries all prescribe the plain `drop` remediation, so a canceled run leaves the same
durable record a completed one leaves. The scheduler applies a cancellation asynchronously, so an allocation it still
carries leaves its batch outstanding, and so does a job whose own tracker still claims to be running; both are released
by `retire_remote_batches_tool`. Each step names its own cause on failure: the cancellation itself reports that nothing
was cancelled, while a tracker read, an accounting read, or a closure that fails behind an accepted cancellation
reports that the cancellation still stands and that retrying is safe.

---

## retire_remote_batches_tool

Resolves every allocation exactly as the remote status branch does, then applies the resolved remediation: cancel, but
only under `force` and then over EVERY allocation those entries leave `held`; reset each `stranded` job to `SCHEDULED`;
snapshot each named batch; drop the ledger entries.

**The cancelled set is not held to this ledger.** `resolve_live_allocations` unions the recorded allocation of every
resolution whose `scheduler_state` reads `held` with the `claimed_allocation` of every resolution whose `claim_state`
reads `held`, and that second one may be an allocation another machine submitted. `force` therefore issues `scancel`
against work this host never recorded, deliberately, so a tracker is never reset underneath the allocation writing to
it. Confirm with the user before passing it.

```text
success              bool
retired              true, literally, on every success
batch_ids[]          the identifiers the ledger actually HELD and dropped, IN LEDGER ORDER rather than
                     caller order. A named identifier the ledger did not hold errors before this point
total_allocations    int, the allocations the dropped batches held between them
batches[]            one per dropped batch, in ledger order. NOT projected:
  batch_id           str
  covered_batch_ids[]   every prepared batch this one submission dispatched, which is [batch_id] for a
                     record written before that field existed
  allocations[]      every scheduler identifier the batch held, UNCAPPED
  outstanding_seconds   float, or null when the record carries no submission time
allocations[]        one per RESOLVED allocation of the named batches, UNCAPPED and not projected:
  batch_id, slurm_job_id, job_id, pipeline, job_name, specifier, unit_name, unit_path
  scheduler_state    str, held, settled, or gone
  tracker_status     str, the ProcessingStatus member name, or "" when no row covers the job
  verdict            str, running, finished, failed, abandoned, or stranded
  remediation        str, COMPOSED from what actually ran rather than copied from the verdict: none
                     when the entry was not dropped, cancel_reset_and_drop when the job was reset AND
                     either of its allocations was cancelled, reset_and_drop when it was reset alone,
                     and drop otherwise. A cancelled allocation whose job recorded an outcome therefore
                     reads drop, and the cancelled flag beside it is what names the cancellation
  cancelled          bool, whether this call issued scancel for it
  tracker_reset      bool, whether its job was returned to SCHEDULED. True for stranded jobs alone
  snapshot_recorded  bool, whether closure recorded an outcome for its batch
  entry_dropped      bool, whether the ledger held and dropped its batch
cancelled_allocations[]   sorted, de-duplicated ids of every allocation the cancellation named
reset_jobs           int, the jobs returned to SCHEDULED, counted by unit path and job identifier
outcomes[]           what closure recorded for each covered batch, same shape as the outcome above.
                     EMPTY when the batches' prepared documents were already forgotten, since closure
                     records nothing for a document this host no longer holds
outcome_directory    str, the LOCAL registry path holding those outcome files and their snapshots
snapshot_error       str, ALWAYS present. Empty when every snapshot succeeded, otherwise naming the
                     batches that failed. Non-empty here means the caller passed drop_without_outcome
message              str, stating how many batches were remediated and that their jobs are unclaimed
```

The drop is all or nothing across the named batches, while the snapshot is per batch, so a partial snapshot failure
still reports every outcome it did record. There is no `host` key, since only a remote batch has a ledger entry, and no
`unknown` list, since an unheld identifier is an error rather than a skipped entry. The error branch covers an
unreadable or unwritable ledger, an empty ledger, an empty `batch_ids`, an unheld identifier, a failed tracker read, a
failed accounting read, an allocation that resolves `running` without `force`, a failed cancellation, a failed tracker
reset, and a failed snapshot without `drop_without_outcome`. An unreachable server resolves every recorded allocation
as `held`, so it refuses for `force` first and for `drop_without_outcome` after, while a failed tracker read, accounting
read, cancellation, or reset is reported rather than waived, because each leaves the world unchanged.

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
removed[]            in removal order, unit by unit. Within one unit: the tracker, then the directory that
                     unit's pipeline owns, then one entry per external directory that pipeline declares:
  path               str
  removed_bytes      int
total_paths          int, the length of removed. PATHS, not units. A unit contributes 0, 1, or 2 entries plus
                     one for each external directory it holds, so its entry count is unbounded
removed_bytes        int, the sum over every entry
```

A `checksum` unit contributes at most one entry, its tracker, because that pipeline owns no output directory and its
stored value stays in place. A `forging` unit's directory entry covers the whole assembled dataset hierarchy, and it is
the one pipeline declaring external directories: one further entry per source session that still resolves, holding the
cross-recording output that dataset owns inside that session. Every external path is resolved before the unit's first
removal, so a resolver that fails leaves the unit untouched, and a source session that no longer loads is warned about
and passed over rather than removed. Remote figures are parsed from the server-side command's own output, so an
unparseable line is dropped and the reported total under-counts.

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
