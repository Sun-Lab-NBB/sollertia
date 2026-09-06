---
name: remote-execution
description: >-
  Runs sollertia-forgery work on the configured SLURM compute server through the `slf mcp` server. Covers the host
  parameter, remote project discovery, the three-step remote run, the submission ledger, the resolved scheduler,
  tracker, and verdict vocabulary that decides what may be done with an outstanding allocation, the retirement that
  releases a stranded job, the generated job script, and scheduler queue and accounting reads. Use when preparing or
  submitting a batch with host='remote', when resolving a project's server-side unit paths, when a batch never leaves
  the outstanding listing, when a job's tracker still claims to be running, or when reading the scheduler's records.
user-invocable: false
---

# Remote execution

Drives the compute-server half of `sollertia-forgery`, where the data sits under the server's configured data root and
every job runs as its own SLURM allocation. This skill is the **exclusive** owner of `discover_remote_project_tool`,
`read_scheduler_jobs_tool`, the submission ledger, and the state vocabulary resolved against it. No other skill in the
marketplace may document or call those two tools.

---

## Scope

**Covers:**
- The `host` parameter, remote project discovery, and the server-side `unit_path` values every remote tool consumes
- What `host='remote'` does per tool, the remote run, the ledger, adoption, and the closure that retires a batch
- The resolved `scheduler_state`, `tracker_status`, `verdict`, `remediation`, and `progress` values, the state table
  they form, the refusal each remediation stands behind, and the recovery path from a stall to a re-runnable job
- The generated SLURM job script, scheduler queue and accounting reads, and the remote-versus-local divergences

**Does not cover:**
- Compute server credentials. Owned by `/server-configuration`.
- MCP connectivity and the shared response envelope. Owned by `/forging-mcp-environment-setup`.
- The `slf` surface a job script invokes. Owned by `/cli-reference`.
- Generic batch mechanics, per-tool parameter tables, status presentation, and batch error routing. Owned by
  `/batch-processing`.
- The resource model that sizes each allocation. Owned by `/job-planning`.
- The project manifest and job artifact a remote read mirrors. Owned by `/project-state`.
- Dataset hierarchies and state. Owned by `/dataset-definition`.
- This machine's session roots. Owned by `assets:session-discovery`.

**Handoff rules:** this skill owns everything the server does and everything the transport reports. Route what the
owners above claim, then return here for submission, adoption, resolution, and retirement.

---

## Agent requirements

You MUST drive every remote operation through the sollertia-forgery MCP tools: never import `sollertia_forgery`
functions, open an SSH session yourself, or hand-edit the submission ledger or a processing tracker. Where the tools are
unavailable, invoke `/forging-mcp-environment-setup`. You MUST confirm a server configuration exists before naming
`host='remote'`, since every remote path opens an SSH connection built from that record. You MUST take every server-side
path from `discover_remote_project_tool` or from a generate or plan response that ran with `host='remote'`. A remote
read reports the mirror under every artifact path key and echoes the caller's own path back only as `project_path`. You
MUST confirm the wall time and the unit selection with the user before submitting, since the scheduler owns the graph
once the last job is queued.

---

## The host parameter

`host` names where the data lives and where the work runs. `interfaces/host_resolution.py::HOST_LABELS` accepts
`local` and `remote` and every tool validates the label before it opens any connection.

| Tool group                                                                      | `host` parameter     | Why it takes that form                                           |
|---------------------------------------------------------------------------------|----------------------|------------------------------------------------------------------|
| The eighteen tools naming a project, a unit, a dataset, or a running batch      | `str = "local"`      | The caller names which machine holds the data                    |
| `list_prepared_batches_tool`                                                    | `str \| None = None` | A filter over this machine's registry, never a target            |
| `execute_jobs_tool`, `forget_prepared_batches_tool`, `read_resource_model_tool` | none                 | A batch runs where it was prepared; the other two are local      |
| `read_server_configuration_tool`, `write_server_configuration_tool`             | none, always local   | The configuration file lives in this machine's working directory |
| `discover_remote_project_tool`, `read_scheduler_jobs_tool`                      | none, always remote  | Nothing on this machine can answer either question               |
| `pull_remote_path_tool`                                                         | none, always remote  | It copies a server file or directory onto this machine           |
| `retire_remote_batches_tool`                                                    | none, always remote  | The submission ledger records remote allocations alone           |

`batches.py::resolve_batch_host` reads the host off the recorded document, so mixed-host identifiers are refused.

---

## `discover_remote_project_tool`

Enumerates the sessions and forged datasets a project holds on the server, in two widening stages. It is the canonical
source of the server-side paths every other remote tool takes as its argument.

| Parameter                           | Type                 | Default         | Meaning                                                                         |
|-------------------------------------|----------------------|-----------------|---------------------------------------------------------------------------------|
| `project`                           | `str`                | (required)      | Only the final path component names the project, resolved under the data root   |
| `unit_kind`                         | `str \| None`        | `None`          | `"session"` or `"dataset"`                                                      |
| `animals`, `sessions`, `datasets`   | `list[str] \| None`  | `None`          | Restrict the listing to these animals, session names, or forged datasets        |
| `limit`, `start_row`                | `int \| None`, `int` | `None`, `0`     | Units to list and the match index to begin at, as the response contract defines |
| `include_items`, `include_sessions` | `bool`, keyword      | `False`, `True` | List units when no filter is named, and cover sessions alongside datasets       |

The tool takes `Path(project).name` and joins it onto the configured data root, so a full server path works while a
value whose final component is empty is rejected. A bare call reports `project`, `project_path`, `total_sessions`,
`total_datasets`, and a `breakdown` over `unit_kind`, `animal`, and `dataset`. Naming any of the four selectors, or
`include_items=True`, adds the paged `units` list, whose entries carry `unit_kind`, `unit_path`, and either `animal` and
`session` or `dataset`, datasets first. **The animals and datasets filters are mutually exclusive**, since a dataset
entry carries no `animal` key and a session entry none for `dataset`. **The tool rejects no filter**, so confirm every
name against the bare call's `breakdown` before trusting an empty page. The whole tree is read in one server-side
search: `server/discovery.py::discover_project_markers` matches a dataset marker at depth two and a session's at depth
four, then drops any session whose first component names a dataset directory. Pass `include_sessions=False` to hold the
search to dataset depth, which still succeeds where another account owns the session directories.

## `read_scheduler_jobs_tool`

Reads the compute server's own record of its allocations, in three widening stages. It asks the scheduler rather than
this machine's submission ledger, so it answers for an allocation another machine submitted and for one already settled.

| Parameter                   | Type                 | Default        | Meaning                                                           |
|-----------------------------|----------------------|----------------|-------------------------------------------------------------------|
| `view`                      | `str`                | `"accounting"` | `"accounting"` or `"queue"`                                       |
| `user`                      | `str \| None`        | `None`         | The account. `None` takes the configured one, `"all"` covers all  |
| `job_ids`                   | `list[str] \| None`  | `None`         | Allocation identifiers, pushed into the scheduler command itself  |
| `job_names`, `states`       | `list[str] \| None`  | `None`         | Scheduler job names and states, both applied in memory            |
| `start_time`, `end_time`    | `str \| None`        | `None`         | `YYYY-MM-DD[ HH:MM:SS]` bounds. Accounting view only              |
| `limit`, `start_row`        | `int \| None`, `int` | `None`, `0`    | Allocations to list and the match index to begin at               |
| `include_items`, `detailed` | `bool`, keyword      | `False`        | List when no filter is named, and add each allocation's resources |

**The two views cover different populations.** `accounting` covers every allocation for which the scheduler still holds
a record, finished ones included, while `queue` covers the allocations it currently holds queued or running. The
breakdown axes follow the view, `state` and `job_name` for accounting and those plus `partition` and `user` for the
queue, and every response echoes the `view` and the `user` it covered.

| View         | Listed fields                                               | `detailed=True` appends                                                                                                    |
|--------------|-------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| `accounting` | `job_id`, `job_name`, `state`, `elapsed`, `cores`           | `peak_step_cpu_time`, `requested_memory`, `maximum_resident_memory`, `peak_step_resident_memory`, `maximum_virtual_memory` |
| `queue`      | `job_id`, `partition`, `job_name`, `user`, `state`, `cores` | `nodes`, `requested_memory`, `elapsed`, `time_limit`, `time_left`                                                          |

**Naming `job_ids` produces a listing and bypasses the account and the date restrictions**, unlike every other read tool
on this server, which stays at its bare stage until a filter is named. They scope the scheduler command itself, so the
account flag and both date bounds are left off, which is the sanctioned way to read an allocation another account
submitted, and the queue view ignores both bounds in every case. The accounting view merges each allocation's step rows
into one record, taking the largest figure any step reported for a measured field and the first non-empty value for
every other. `total_jobs` counts those merged rows before the in-memory filters run, while `matched_rows` counts what
they kept.

---

## Remote execution architecture

Two mechanisms cover every tool accepting `host='remote'`, and which one runs depends on whether the tool writes. **A
write regenerates on the server.** `orchestration/hosts.py::RemoteHost` renders the argument vector that calls the same
function the local host calls in process, an `slf` command line for every write but the dataset definition. It chains
the vectors with `&&` and ships them through `hosts.py::_environment_commands`, which activates the configured conda
environment first. A non-zero exit raises, naming the rendered invocation and the server's stderr.

| Tool with `host='remote'`        | Server-side command                                                 |
|----------------------------------|---------------------------------------------------------------------|
| `plan_session_jobs_tool`         | `slf plan session -sp <unit> …` then `slf plan project -pp <root>`  |
| `plan_dataset_jobs_tool`         | `slf plan dataset -dp <unit> …` then `slf plan project -pp <root>`  |
| `generate_project_plan_tool`     | `slf plan project -pp <root>` alone                                 |
| `generate_project_manifest_tool` | `slf manifest -pp <root> create`                                    |
| `generate_dataset_state_tool`    | `slf dataset-state -dp <dataset>`, one invocation per named dataset |
| `define_forging_dataset_tool`    | `python -c` calling the dataset definition entry point              |
| `reset_processing_jobs_tool`     | `slf reset -p <pipeline> -up <unit> [-id <job_id> …]`, one per unit |
| `clean_processing_output_tool`   | `slf clean -p <pipeline> -up <unit> …`                              |

**A read mirrors, then reads the copies.** `interfaces/host_resolution.py::resolve_readable_project` pulls the project's
artifacts into `<working directory>/remote_state/<project>` and hands the read tool that local directory.
`orchestration/remote.py::sync_project_state` pulls six artifact kinds and skips any the server does not hold. Those
kinds are the project manifest, the manifest tracker, the project job artifact, the plan projection, and each dataset's
marker and state table, the last two because the read tools resolve a dataset from its marker. **A read never
regenerates**, so generate with `host='remote'` first whenever the state must be fresh. Every reader echoes the caller's
own server path back as its `project_path`, through `host_resolution.py::resolve_reported_project_path`, since a write
tool takes a server path and handing back the mirror would name a machine the caller never named. The artifact keys
beside it, such as `manifest_path`, `jobs_path`, and `plan_path`, report the mirror the read actually opened.

---

## The remote workflow

| Step               | Call                                                            | Confirm or read back                                                    |
|--------------------|-----------------------------------------------------------------|-------------------------------------------------------------------------|
| Confirm the access | `/server-configuration`                                         | The data root and the environment name, with the user                   |
| Discover the units | `discover_remote_project_tool`                                  | The bare `breakdown`, then the `unit_path` values a filter lists        |
| Prepare            | `prepare_batch_tool`, `host='remote'`                           | The `batch_id`, since `execute_jobs_tool` takes identifiers and no path |
| Confirm the size   | `include_job_descriptors=True`, or `inspect_job_resources_tool` | Per-job `cores` and `resident_mb`, and the wall time, with the user     |
| Submit             | `execute_jobs_tool` with the confirmed `walltime_minutes`       | `batch_directory`, `submissions`, and `adopted_jobs`                    |
| Poll               | `get_processing_status_tool`, `host='remote'`                   | `stalled_batch_ids` every time, and the two log paths under `detailed`  |
| Read the result    | Regenerate on the server, then `/project-state`                 | Or the closure outcome the status tool reports for a settled batch      |

Preparation runs the planning and state steps over SSH and resolves the batch graph here, while submission requests
one allocation per job. Say which jobs were adopted rather than resubmitted whenever `adopted_jobs` is non-empty.

---

## The submission ledger

`<working directory>/remote_state/submission_ledger.yaml` is this machine's durable record of what it submitted. Every
write takes a `FileLock` on the sibling `submission_ledger.yaml.lock` with a twenty-second timeout, while reads take no
lock. It records one `orchestration/ledger.py::SubmissionBatch` per submission, carrying the batch identifier, the
prepared identifiers it covered, the server-side batch directory, the submission timestamp, the wall time, and one
`RemoteSubmission` per accepted allocation. **The write sits in a `finally` block**, so a submission the scheduler
rejects partway through still records every allocation it accepted, and a re-submitted job replaces its entry rather
than duplicating it.

**A batch leaves the ledger through closure or through explicit retirement.** Four calls snapshot a batch. A status read
closes what its own resolution clears, and a remote cancellation closes the batches it canceled. A submission closes
every batch the same rule clears while the new one was being prepared, and `retire_remote_batches_tool` closes each
batch it is about to drop. The first three go through `orchestration/closure.py::close_settled_batches`. There is no
`batch_is_settled` predicate, in `ledger.py` or anywhere: the rule is that function, and it takes the caller's own
`AllocationResolution` list from `orchestration/remote.py` rather than a second scheduler read, so the closure and the
published verdicts cannot diverge. **It closes a batch exactly when every entry it holds for that batch prescribes the
plain `drop` remediation**, which alone leaves every tracker as it stands. `none`, `reset_and_drop`, and
`cancel_reset_and_drop` each hold the batch open for the explicit retirement, while a batch holding no allocation holds
no entry and closes. A failed closure is logged as a warning and leaves its batch outstanding for the next query, so an
identifier retires only where its own closure raised nothing. Retirement runs the same `close_covered_batches` those
three do, then drops the entry whatever its allocations hold.

Closure is what makes a finished run answerable. It regenerates the state artifacts on the server, pulls each into
`remote_state/prepared_batches/<batch_id>/<that artifact's own parent directory name>/`, writes the outcome to
`remote_state/prepared_batches/<batch_id>.outcome.yaml`, and only then retires the prepared document. That outcome file
is **a sibling of that snapshot directory rather than a file inside it**, since
`orchestration/batches.py::_outcome_path` joins the name onto the registry directory itself. A batch absent from the
outstanding listing has finished, was retired, or was never submitted, and its answer is that outcome file or a
`host='remote'` read of the project's job artifact.

**A live allocation is adopted rather than resubmitted.** Before anything is submitted,
`orchestration/reconcile.py::reconcile_remote_jobs` collects each job's claimed allocation from two sources and asks
the scheduler for their states. A job whose verdict resolves as `running` because either allocation reads `held` is
adopted onto that allocation, which seeds the dependency map so a dependent waits on the work already running.

| Claim source                    | Covers                                                    | Blind spot                                      |
|---------------------------------|-----------------------------------------------------------|-------------------------------------------------|
| The submission ledger           | Allocations this machine submitted, queued ones included  | A batch submitted from another machine          |
| The job's own recorded executor | Every submitter, because the record travels with the data | Appears only once the allocation starts running |

The executor wins where both name an allocation, because it describes a later moment than the submission record, and
`reconcile.py::_resolve_job_claims` builds one claim per job from the descriptors preparation filled, whatever status it
recorded. Only the `slurm` scheme names an allocation, so a job whose tracker reads `RUNNING` under another scheme, such
as the `pid:` a local run writes, resolves `running` with nothing to adopt. This run withholds it and everything
downstream as `withheld_jobs`, rather than running a second copy over a tracker to which its own executor may still be
writing.

---

## Resolving an outstanding allocation

Every value below is resolved from three records and from no elapsed period. `Server.get_job_statuses` issues one
`sacct` call and answers `UNRESOLVED` for any allocation for which it returned no row. `Server.get_queued_job_ids`
issues `squeue -h -u <user> -o "%i"` and answers the set the controller holds right now, empty for an account holding
none. `remote.py::resolve_tracker_claims` regenerates each covered project's state artifacts on the host before reading
them, once per project and unit kind, because nothing else refreshes them while a batch runs. `read_scheduler_records`
reads accounting first and raises on a failure, since a failed query writes nothing and reading that as "no row for
anything" would resolve every allocation as gone at once. The queue is read second, and a failure there is carried into
`SchedulerReading.unreadable_reason`, because everything then resolves `held`.

### The three scheduler states

`SchedulerReading.resolve_state` returns exactly one `scheduler_state`, testing in this order.

| Order | Condition                                                     | Resolves to |
|-------|---------------------------------------------------------------|-------------|
| 1     | The record names no allocation at all                         | `gone`      |
| 2     | This call canceled it, or accounting answers `BLOCKED`       | `settled`   |
| 3     | A scheduler record could not be read, or the queue carries it | `held`      |
| 4     | Accounting answers `UNRESOLVED`, which is no row              | `gone`      |
| 5     | Accounting answers one of the ten other terminal states       | `settled`   |
| 6     | Anything else, so `PENDING`, `RUNNING`, or `UNKNOWN`          | `held`      |

The eleven `server/server.py::TERMINAL_JOB_STATUSES` are `COMPLETED`, `FAILED`, `CANCELLED`, `TIMEOUT`, `NODE_FAIL`,
`OUT_OF_MEMORY`, `BOOT_FAIL`, `DEADLINE`, `PREEMPTED`, `REVOKED`, and `BLOCKED`; neither `UNKNOWN` nor `UNRESOLVED` is
one. **A settled allocation still has an accounting row**, since `COMPLETED` resolves `settled` precisely because
accounting answers for it while the queue does not, so the predicate is what the two records say and never whether a
row exists. Only `gone` needs both to disclaim the allocation, which is why the queue read exists, and a cancellation
is recorded through `SchedulerReading.canceling` rather than re-queried, since `scancel` is asynchronous.

### The tracker claim

Every job yields a `TrackerClaim` of three keys. `tracker_status` is the `ProcessingStatus` member the job's tracker
holds, one of `SCHEDULED`, `RUNNING`, `SUCCEEDED`, and `FAILED`, or `""` when the state artifact holds no row for it.
`tracker_executor_id` is the scheme-tagged executor the same row named, and `claimed_allocation` is the `slurm:`
allocation it claims, or `""` for every other scheme. `resolve_queried_allocations` queries that allocation alongside
the ledger's own record and reports its state as `claim_state`, since the two cover different submitters.

### The state table

`_resolve_verdict` and `_VERDICT_REMEDIATIONS` are the table itself, so shown and applied `remediation` cannot drift.

| `scheduler_state`, `claim_state`, and the executor | `tracker_status`       | `verdict`   | `remediation`    | What the remediation does                                       |
|----------------------------------------------------|------------------------|-------------|------------------|-----------------------------------------------------------------|
| either state reads `held`                          | anything               | `running`   | `none`           | Nothing, and the whole remediation is refused without `force`   |
| neither, executor names no `slurm:` allocation     | `RUNNING`              | `running`   | `none`           | Nothing. No record answers for that executor, so it is refused  |
| neither state reads `held`                         | `SUCCEEDED`            | `finished`  | `drop`           | Snapshot, drop the ledger entry, leave the tracker alone        |
| neither state reads `held`                         | `FAILED`               | `failed`    | `drop`           | Snapshot, drop, leave the tracker alone. A failure is a verdict |
| neither state reads `held`                         | `SCHEDULED`, or no row | `abandoned` | `drop`           | Snapshot, drop. Nothing claims the job, so it is runnable       |
| neither, `slurm:` or no executor                   | `RUNNING`              | `stranded`  | `reset_and_drop` | Reset that job to `SCHEDULED` on its tracker, snapshot, drop    |

`held` on either allocation outranks every tracker status, because the tracker of a job the scheduler still carries may
be written at any moment and a verdict read off it would describe the past. So does an executor naming no scheduler
allocation, since `TrackerClaim.unqueryable_executor` marks one no record can answer for. **The tracker is written for
`stranded` and nothing else**, so `reset_processing_jobs_tool` and `slf reset` stay the way to clear a failure. Under
`force` the verdicts are resolved again after the cancellation, and the `remediation` the response then reports is
composed from what actually ran, reading `cancel_reset_and_drop` only where a canceled allocation's job was reset.

### The batch verdict

`classify_batch` derives one `progress` per batch from those same resolutions, so a batch verdict and its allocations'
verdicts cannot drift, and `_diagnose_batch` reports it beside `outstanding_seconds`, which is never an input to it.

| `progress`         | Exactly when                                       | Act on it by                                                                                  |
|--------------------|----------------------------------------------------|-----------------------------------------------------------------------------------------------|
| `progressing`      | At least one allocation resolves `running`         | Waiting. A held allocation is live work however long it has queued                            |
| `stalled`          | None resolves `running` and at least one is `gone` | Retiring it. It outlived this read's own closure, so a job is stranded or that closure failed |
| `awaiting_closure` | None resolves `running` and none is `gone`         | Reading the status again, which closes it unless one of those two holds it open               |

The third row covers a batch every allocation of which settled and one holding none. Each entry also carries `verdicts`,
a count per verdict, plus `live_allocations`, `stranded_allocation_count`, and `unresolvable_allocation_count`, each
beside a list of up to 50 of them while the count covers the batch, and a `remedy`.

**`remedy` leads with the retirement wherever closure cannot release the batch.** In `_batch_remedy`, `stalled`
quotes `retire_remote_batches_tool` and `slf server retire-batch -b <id>`, and so does `awaiting_closure` while any of
its jobs is stranded, since closure never releases a tracker still claiming an allocation. An `awaiting_closure` batch
holding none names the status read first, and `progressing` names `cancel_processing_tool`.

**The response-level `active` flag counts verdicts, not scheduler states.** `remote_batch_status` computes
`any(resolution.verdict == running)` over every covered allocation, so a tracker claim alone sets it. `active` is one
flag over all covered batches, so monitor on each batch's own `progress` and on `stalled_batch_ids`. The response also
carries `uncovered_batch_ids`, naming any batch another process recorded while the call ran.

---

## Retiring a batch

`retire_remote_batches_tool(batch_ids=[...], force=False, drop_without_outcome=False)`, and the equivalent `slf server
retire-batch -b <id>`, is the one tool that acts on those verdicts. It resolves exactly as the status read resolves,
then runs four steps in this order. It cancels, but only under `force`, resets every `stranded` job to `SCHEDULED`
through the host's own reset, snapshots each named batch through the same `close_covered_batches` through which a
settled batch goes, and drops the ledger entries. `batch_ids` is required and takes no wildcard, since retirement is
irreversible.

**That cancellation is not held to this ledger.** `remote.py::resolve_live_allocations` unions the recorded allocation
of every resolution whose `scheduler_state` reads `held` with `resolution.tracker.allocation` of every one whose
`claim_state` reads `held`. `force` can therefore `scancel` an allocation another machine submitted and this ledger
never recorded. That is deliberate, since resetting a tracker under the allocation still writing to it is the
destruction this resolution refuses, but it reaches work this host did not start: confirm with the user.

Two guarantees stand in front of that, each waived by its own flag alone, and each refusal names only that flag.

| Guarantee                                          | Waived by                     | Refused when                                                                                                      |
|----------------------------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------|
| No allocation that resolves `running` is disturbed | `force`, `-f`                 | The scheduler holds one, its tracker claims a held one or an unqueryable executor, or a record could not be read  |
| Every dropped entry was snapshotted first          | `drop_without_outcome`, `-do` | Closure could not read what a named batch's jobs recorded                                                         |

An unreachable server hides both scheduler records and the trackers at once, so every recorded allocation resolves
`held`, the refusal names `force`, and its message quotes the cause verbatim. A failed queue read reaches the same
refusal by the same route. Under `force` the held allocations are canceled first, so a live allocation cannot write
into a tracker reset underneath it. The verdicts are then resolved again against that recorded cancellation, and only a
job that then reads `stranded` is reset, another machine's claimant included, since its claim now reads `settled`.

**Five failures waive nothing**, because each leaves the world unchanged and retrying costs nothing: a failed tracker
read, a failed accounting read, a failed cancellation, a failed tracker reset, and a ledger lock timeout on the drop.
Each error names its own cause, `force` carries a caller past none, and the snapshot is per batch, the drop atomic.

### The recovery path, end to end

1. **Notice the stall.** Poll `get_processing_status_tool` with `host='remote'`, or `slf server batches`, and branch on
   `stalled_batch_ids`, never on `active`, for the reason the batch verdict section gives.
2. **Read the allocations.** Call the same tool with `batch_ids=[...]`, `include_items=True`, and `detailed=True`. Every
   row carries `scheduler_state`, `tracker_status`, `verdict`, and `remediation`; only detail adds `claimed_allocation`
   and `claim_state`, the evidence behind a `running` verdict, which `slf server batches -b <id> -a` cannot print.
3. **Clear anything running.** Where a row reads `running`, either wait, or call `cancel_processing_tool` with
   `host='remote'` and those `batch_ids` and read again. Prefer that over `force`, which abandons live work. One
   `running` row neither waiting nor canceling ever clears: the one whose `claimed_allocation` is empty while
   `tracker_executor_id` names a non-`slurm` scheme. No cancellation names it and no elapsed period settles it, so its
   batch stays `progressing` rather than joining `stalled_batch_ids`. Confirm with the user that nothing is still
   writing to that unit, then reset that job with `reset_processing_jobs_tool`. Its verdict then reads `abandoned`,
   whose remediation is the plain `drop`, so the next status read closes the batch on its own.
4. **Retire the batch.** Call `retire_remote_batches_tool(batch_ids=[...])`, or run `slf server retire-batch -b <id>`,
   adding `force=True` only after step 3 and `drop_without_outcome=True` only after a snapshot has actually failed.
5. **Confirm what it did.** Read `reset_jobs`, each `allocations[*]` entry's `canceled`, `tracker_reset`,
   `snapshot_recorded`, and `entry_dropped`, then `snapshot_error`, `outcomes`, and `outcome_directory`.
6. **Re-run the released work.** Prepare and execute the same pipeline and units again through `/batch-processing`.
   Preparation omits succeeded jobs alone, so a job reset in step 4 is prepared again, and reconciliation no longer
   adopts it, since the reset cleared both the running status and the executor identifier on which its claim lived.

A job whose verdict read `failed` keeps that record deliberately: step 6 prepares and reruns it as outstanding work.
`reset_processing_jobs_tool` clears the recorded failure first where a clean slate is wanted, while one that read
`finished` is omitted by preparation. Nothing is left to the machine that submitted a claimed allocation: without
`force` that held claim refuses the whole batch, and with `force` the allocation is canceled and the job reset here. A
job running under a non-`slurm` executor is the one case `force` does not release, since it cancels and resets nothing.
The ledger entry goes while the tracker keeps claiming the run, so every later batch withholds that job and its
dependents again. Reset it through step 3 first.

---

## The remote job script

`server/job.py` composes one shell script per allocation and `Server.submit_job` uploads it, marks it executable, and
submits it. See [remote-job-script.md](references/remote-job-script.md) for the script template, its six SBATCH
directives, the job name format, and where each resource request comes from.

---

## Remote versus local divergences

| Concern                           | `host='local'`                                                              | `host='remote'`                                                                                 |
|-----------------------------------|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Where a run's state lives         | A process-global that dies with the server                                  | The ledger plus the scheduler, both surviving a restart                                         |
| Concurrent batches                | One run at a time, over any number of batches, a second dispatch is refused | Unbounded, the ledger tracks many at once                                                       |
| Arguments on execute              | `core_budget_override` and `memory_budget_mb` honored, wall time ignored    | Both budgets ignored, `walltime_minutes` honored, 480 by default                                |
| Concurrency ceilings              | Enforced by the admission engine                                            | Never expressed to the scheduler                                                                |
| A job already recorded as running | Rerun, since the record describes a dead pool                               | Adopted onto the held allocation, or withheld with its dependents when neither record names one |
| Progress source and vocabulary    | The processing trackers, in lowercase tracker labels                        | Both records and the trackers, resolved into the verdicts above                                 |
| Tracker paths on job descriptors  | Real paths                                                                  | Empty, by design                                                                                |
| Cancellation                      | Cooperative, in-flight jobs finish                                          | Queued and running killed alike, dependents cascade                                             |
| Cleaning guard and failures       | Refused while a batch runs here, and an unloadable unit is warned and kept  | Never refused, and a non-zero server exit raises                                                |

**The cleaning divergence is the dangerous one.** The running-batch guard covers this process's pool alone, so a remote
clean is accepted while allocations are in flight and will remove the output and the tracker of an allocation still
running. A `forging` clean removes more than the unit named: the dispatch entry declares an external-output hook that
`orchestration/maintenance.py::clean_pipeline_output` resolves and removes, so each source session's cross-recording
directory for that dataset goes with the dataset tree. Confirm through a `host='remote'` status read that nothing is
outstanding before cleaning, and prefer a reset, which touches records alone.

---

## Error routing

| Message                                                                                                 | Raised by                      | Remedy                                                                       |
|---------------------------------------------------------------------------------------------------------|--------------------------------|------------------------------------------------------------------------------|
| `Unknown unit kind '{unit_kind}'. Available: ['dataset', 'session'].`                                   | `discover_remote_project_tool` | Name one of the two kinds, or omit the parameter                             |
| `Unable to discover the '{project}' project on the compute server. …`                                   | `discover_remote_project_tool` | Pass a value whose final component names a project the data root holds       |
| `Unable to search {path} on the remote compute server. The search reached only part of the tree …`      | `discover_remote_project_tool` | Retry with `include_sessions=False`, which holds the search to dataset depth |
| `Unknown scheduler view '{view}'. Available: ['accounting', 'queue'].`                                  | `read_scheduler_jobs_tool`     | Name `accounting` or `queue`                                                 |
| `Unable to reach the compute server's scheduler. …`, `Unable to read the scheduler's {view} records. …` | `read_scheduler_jobs_tool`     | Read the wrapped exception or the quoted command, both safe to show the user |
| `No scheduler record has '{field}' in {unknown}. Available: {sorted}.`                                  | `read_scheduler_jobs_tool`     | Correct the `job_names` or `states` value against the listed set             |
| `Unable to read the allocations the remote compute server's queue holds with '{command}'. …`            | `Server.get_queued_job_ids`    | Reaches a caller as the read failure that resolves every allocation as held  |

`/batch-processing` owns the status and remediation messages themselves. An authentication failure is fatal at once,
while any other connection failure retries thirty times at two second intervals before the transport reports the server
unreachable, accepting an unrecognized host key rather than prompting for it.

---

## Related skills

| Skill                                                       | Relationship                                                                   |
|-------------------------------------------------------------|--------------------------------------------------------------------------------|
| `/server-configuration`, `/forging-mcp-environment-setup`   | Prerequisites: the access record, MCP connectivity, the response contract      |
| `/batch-processing`, `/job-planning`                        | Peer and upstream: the batch surface, and the estimates sizing each allocation |
| `/project-state`, `/dataset-definition`, `/dataset-forging` | Downstream: the mirrored manifest and job artifact, dataset state, and forging |
| `/cli-reference`, `/pipeline`                               | Reference: the `slf` surface a job script invokes, and where this sits         |
| `assets:working-directory`, `assets:session-discovery`      | Upstream: the working directory holding the ledger, and session roots          |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Submission and monitoring:
- [ ] The server configuration was read through /server-configuration before any host='remote' call
- [ ] Every server-side unit path came from discover_remote_project_tool or a generate or plan response, and no path
      a read_* response reported was fed back into a tool with host='remote'
- [ ] The batch was prepared with host='remote', its batch_id and wall time confirmed, adopted jobs reported
      separately, stalled_batch_ids read on every poll, and nothing cleaned while a batch was outstanding

Retirement:
- [ ] Every verdict and remediation was read from the tool's own response, never inferred from elapsed time
- [ ] Any running allocation was waited out or canceled before force was considered, the user confirmed that force
      cancels a claimed allocation this ledger never recorded, and each waiver followed its own plain refusal
- [ ] tracker_reset and reset_jobs were read back to confirm each stranded job was released, and no ledger file and
      no processing tracker was edited by hand
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
