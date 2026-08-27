---
name: remote-execution
description: >-
  Runs sollertia-forgery work on the configured SLURM compute server through the `slf mcp` server. Covers the host
  parameter, remote project discovery, the three-step remote run, the submission ledger, the generated job script, and
  scheduler queue and accounting reads. Use when preparing or submitting a batch with host='remote', when resolving a
  project's server-side unit paths, when reading the scheduler's records, or when a remote run behaves differently from
  a local one.
user-invocable: false
---

# Remote execution

Drives the compute-server half of `sollertia-forgery`, where the data sits under the server's configured data root and
every job runs as its own SLURM allocation. This skill is the **exclusive** owner of `discover_remote_project_tool` and
`read_scheduler_jobs_tool`. No other skill in the marketplace may document or call these two tools.

---

## Scope

**Covers:**
- The `host` parameter, which tools take it, and which tools are fixed to one host
- Remote project discovery and the absolute server-side `unit_path` values every remote tool consumes
- What `host='remote'` does per tool, namely regeneration on the server and mirroring for a read
- The three-step remote run, prepare on the server, submit one allocation per job, read while in flight
- The submission ledger, allocation adoption, and the closure that retires a batch
- The generated SLURM job script, its directives, its environment activation, and its wall time
- Scheduler queue and accounting reads
- The behaviors that diverge between a remote run and a local one

**Does not cover:**
- Compute server credentials, the configuration fields, and password masking. Owned by `/server-configuration`.
- Generic batch mechanics, per-tool parameter tables, status presentation, and batch error routing. Owned by
  `/batch-processing`.
- Job planning, per-unit estimates, and the resource model that sizes each allocation. Owned by `/job-planning`.
- The project manifest and the project job artifact a remote read mirrors. Owned by `/project-state`.
- Dataset hierarchy creation and dataset state. Owned by `/dataset-definition`.
- Unit-root discovery on this machine, with session roots owned by `assets:session-discovery` and dataset roots by
  `/dataset-definition`.
- The `slf` command surface a job script invokes. Owned by `/cli-reference`.
- MCP server connectivity. Owned by `/forging-mcp-environment-setup`.

**Handoff rules:** this skill owns everything the server does and everything the transport reports. Invoke
`/server-configuration` to author the access record, `/batch-processing` to prepare and dispatch, and `/job-planning`
for the figures that size each allocation, then return here for submission, adoption, and scheduler reads.

---

## Agent requirements

You MUST drive every remote operation through the sollertia-forgery MCP tools. Do not import `sollertia_forgery`
functions directly and do not open an SSH session yourself. Where the tools are unavailable, invoke
`/forging-mcp-environment-setup` to diagnose connectivity.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

You MUST confirm that a server configuration exists before naming `host='remote'` anywhere. Every remote path opens an
SSH connection built from that record, and an unconfigured host fails the tool rather than falling back to local work.
`/server-configuration` owns the read, the write, and the fields.

You MUST take every server-side path from `discover_remote_project_tool`, or from the response of a generate or plan
tool that ran with `host='remote'`. Never assemble a server path by hand, and never round-trip a path a `read_*`
response reported, because a remote read reports the local mirror rather than the server.

You MUST confirm the wall time and the unit selection with the user before submitting, since a submission requests one
allocation per job and the scheduler owns the graph the moment the last job is queued.

---

## The host parameter

`host` names where the data lives and where the work runs. The accepted labels are `local` and `remote`, resolved from
`interfaces/host_resolution.py::HOST_LABELS`, and a tool validates the label before it opens any connection.

```text
Unsupported host '{host}'. Available: local, remote.
```

| Tool group                                                                 | `host` parameter     | Why it takes that form                                           |
|----------------------------------------------------------------------------|----------------------|------------------------------------------------------------------|
| The eighteen tools naming a project, a unit, a dataset, or a running batch | `str = "local"`      | The caller names which machine holds the data                    |
| `list_prepared_batches_tool`                                               | `str \| None = None` | A filter over this machine's registry, never a target            |
| `execute_jobs_tool`                                                        | none                 | A batch runs where it was prepared                               |
| `forget_prepared_batches_tool`, `read_resource_model_tool`                 | none                 | The batch registry and the resource model describe this machine  |
| `read_server_configuration_tool`, `write_server_configuration_tool`        | none, always local   | The configuration file lives in this machine's working directory |
| `discover_remote_project_tool`, `read_scheduler_jobs_tool`                 | none, always remote  | Nothing on this machine can answer either question               |

`execute_jobs_tool` reads the host off the recorded batch document, so a set of batch identifiers spanning both hosts
is refused rather than split. `orchestration/batches.py::resolve_batch_host` is where that refusal is raised.

---

## Available tools

### `discover_remote_project_tool`

Enumerates the sessions and forged datasets a project holds on the server, in two widening stages, and is the canonical
source of the absolute server-side paths every other remote tool takes as its argument.

```python
discover_remote_project_tool(
    project: str,
    unit_kind: str | None = None,
    animals: list[str] | None = None,
    sessions: list[str] | None = None,
    datasets: list[str] | None = None,
    limit: int | None = None,
    start_row: int = 0,
    *,
    include_items: bool = False,
    include_sessions: bool = True,
) -> dict[str, Any]
```

| Parameter          | Type                | Default    | Meaning                                                                       |
|--------------------|---------------------|------------|-------------------------------------------------------------------------------|
| `project`          | `str`               | (required) | Only the final path component names the project, resolved under the data root |
| `unit_kind`        | `str \| None`       | `None`     | `"session"` or `"dataset"`                                                    |
| `animals`          | `list[str] \| None` | `None`     | Restricts the listing to these animals' sessions                              |
| `sessions`         | `list[str] \| None` | `None`     | Restricts the listing to these session names                                  |
| `datasets`         | `list[str] \| None` | `None`     | Restricts the listing to these forged datasets                                |
| `limit`            | `int \| None`       | `None`     | Units to list, paged as the response contract describes                       |
| `start_row`        | `int`               | `0`        | Match index at which the listing begins                                       |
| `include_items`    | `bool`              | `False`    | Lists units when no filter is named                                           |
| `include_sessions` | `bool`              | `True`     | Covers acquired sessions alongside datasets                                   |

**The project argument uses its final component alone.** Passing a full server path works because the tool takes
`Path(project).name` and joins it onto the configured data root. A value whose final component is empty is rejected,
since it would resolve to the data root itself.

A bare call reports `project`, `project_path`, `total_sessions`, `total_datasets`, and a `breakdown` over `unit_kind`,
`animal`, and `dataset`. Naming any of the four selectors, or `include_items=True`, adds the paged `units` list, whose
entries carry `unit_kind`, `unit_path`, and either `animal` and `session` or `dataset`. Datasets are listed first.

**The animals and datasets filters are mutually exclusive by construction.** A dataset entry carries no `animal` key
and a session entry carries no `dataset` key, so filtering on one kind discards every entry of the other, and naming
both together matches nothing. Filter on `unit_kind` first when you want one kind, and run the tool twice when you
want both.

**The tool rejects no filter.** There is no unknown-value check on `animals`, `sessions`, or `datasets`, so a mistyped
name yields a silent empty page with `matched_rows: 0` rather than an error. Confirm every name against the bare
call's `breakdown` before trusting an empty result.

The whole tree is read in one server-side search, so the cost is one round trip rather than one query per directory.
`server/discovery.py::discover_project_markers` matches a dataset marker at depth two and an acquired session's marker
at depth four. It then drops any session whose first component names a dataset directory, since a directory carrying a
dataset marker is a forged dataset rather than an animal. Pass `include_sessions=False` to hold the search to the
dataset depth, which reads fewer directories and therefore still succeeds on a project whose session directories
another account owns.

### `read_scheduler_jobs_tool`

Reads the compute server's own record of its allocations, in three widening stages. It asks the scheduler rather than
this machine's submission ledger, so it answers for an allocation another machine submitted and for one that has
already settled.

```python
read_scheduler_jobs_tool(
    view: str = "accounting",
    user: str | None = None,
    job_ids: list[str] | None = None,
    job_names: list[str] | None = None,
    states: list[str] | None = None,
    start_time: str | None = None,
    end_time: str | None = None,
    limit: int | None = None,
    start_row: int = 0,
    *,
    include_items: bool = False,
    detailed: bool = False,
) -> dict[str, Any]
```

| Parameter       | Type                | Default        | Meaning                                                                       |
|-----------------|---------------------|----------------|-------------------------------------------------------------------------------|
| `view`          | `str`               | `"accounting"` | `"accounting"` or `"queue"`                                                   |
| `user`          | `str \| None`       | `None`         | The account to cover. `None` takes the configured account, `"all"` covers all |
| `job_ids`       | `list[str] \| None` | `None`         | Allocation identifiers, pushed into the scheduler command itself              |
| `job_names`     | `list[str] \| None` | `None`         | Scheduler job names, applied in memory                                        |
| `states`        | `list[str] \| None` | `None`         | Scheduler states, applied in memory                                           |
| `start_time`    | `str \| None`       | `None`         | `YYYY-MM-DD` or `YYYY-MM-DD HH:MM:SS`. Accounting view only                   |
| `end_time`      | `str \| None`       | `None`         | Same formats. Accounting view only                                            |
| `limit`         | `int \| None`       | `None`         | Allocations to list, paged as the response contract describes                 |
| `start_row`     | `int`               | `0`            | Match index at which the listing begins                                       |
| `include_items` | `bool`              | `False`        | Lists allocations when no filter is named                                     |
| `detailed`      | `bool`              | `False`        | Adds each allocation's resource figures                                       |

**The two views cover different populations.** `accounting` covers every allocation for which the scheduler still holds
a record, finished ones included, and its detail carries what each one truly occupied. `queue` covers the allocations
the scheduler currently holds queued or running, and its detail carries their shape and their remaining wall time. The
breakdown axes follow the view, `state` and `job_name` for accounting, and `state`, `partition`, `user`, and `job_name`
for the queue. Every response echoes the `view` it read and the `user` it covered, which is the account the server
configuration names when the caller passes none.

| View         | Listed fields                                               | `detailed=True` appends                                                                                                    |
|--------------|-------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| `accounting` | `job_id`, `job_name`, `state`, `elapsed`, `cores`           | `peak_step_cpu_time`, `requested_memory`, `maximum_resident_memory`, `peak_step_resident_memory`, `maximum_virtual_memory` |
| `queue`      | `job_id`, `partition`, `job_name`, `user`, `state`, `cores` | `nodes`, `requested_memory`, `elapsed`, `time_limit`, `time_left`                                                          |

**Naming `job_ids` alone produces a listing.** Every other read tool on this server stays at its bare stage until a
filter is named or items are requested. This one treats the identifiers as an explicit request for items, even though
they never act as an in-memory filter.

**Naming `job_ids` also bypasses the account and the date restrictions.** The identifiers scope the scheduler command
itself, so the account flag and both date bounds are left off entirely. That is the sanctioned way to read an
allocation another account submitted. The queue view ignores `start_time` and `end_time` in every case.

The accounting view merges each allocation's step rows into one record, taking the largest figure any step reported for
a measured field and the first non-empty value for every other field. The record therefore names the widest step rather
than a figure folded across them, and it keeps that step's own text so every figure stays in the units the scheduler
chose. `total_jobs` counts the merged rows before the in-memory filters run, while `matched_rows` counts what the
filters kept, so the two describe different populations.

---

## Remote execution architecture

Two mechanisms cover every tool that accepts `host='remote'`, and which one runs depends on whether the tool writes or
reads.

**A write regenerates on the server.** `orchestration/hosts.py::RemoteHost` renders the same `slf` argument vector the
local host would call in process, chains the vectors with `&&`, and ships them through
`orchestration/hosts.py::environment_commands`, which activates the configured conda environment first. Both hosts
therefore write identical artifacts from identical code.

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

A non-zero exit raises, and the wrapped message names the exact rendered invocation and the server's own stderr.

**A read mirrors, then reads the copies.** `interfaces/host_resolution.py::resolve_readable_project` takes the final
component of the project path, pulls the project's artifacts into `<working directory>/remote_state/<project>`, and
hands the read tool that local directory. The mirror keeps the project's name, so the artifact filenames a remote
writer derived are the filenames a local reader resolves, which is why one reader serves both hosts.

`orchestration/remote.py::sync_project_state` pulls six artifact kinds and skips any the server does not hold, the
project manifest, the manifest tracker, the project job artifact, the project plan projection, and each dataset's
marker and state table. The markers and the tracker travel alongside the tables because the read tools resolve a
dataset from its marker and read manifest progress from its tracker.

**A read never regenerates.** The mirroring call runs with regeneration disabled, so a remote read reports what the
server already wrote. Run `generate_project_manifest_tool` or `generate_dataset_state_tool` with `host='remote'` first
whenever the state must be fresh, then read.

**A remote read reports the local mirror path.** `read_project_manifest_tool`, `read_project_jobs_tool`,
`read_project_plan_tool`, and `get_manifest_status_tool` all report the mirror directory as their `project_path`, and
`read_dataset_state_tool` and `list_project_datasets_tool` echo the caller's own argument while reporting mirror paths
inside their entries.

---

## The remote workflow

### Execution model

1. **Prepare on the server.** `prepare_batch_tool` with `host='remote'` runs the planning and state steps over SSH,
   pulls the resulting tables here, and resolves the whole batch graph on this machine. The prepared document is
   durable under `remote_state/prepared_batches/` and outlives the MCP server process.

2. **Submit one allocation per job.** `execute_jobs_tool` names the recorded identifiers, sizes each allocation from
   that job's own estimate, and sequences the graph through `afterok` dependencies. Nothing has to stay running here
   once the last job is queued.

3. **Read while it is in flight.** `get_processing_status_tool` with `host='remote'` queries the scheduler for every
   allocation the ledger holds. It is the only progress source a remote run has, because no tracker on the server is
   ever opened from this machine.

### Workflow steps

1. **Confirm the configuration.** Read the server configuration through `/server-configuration` and confirm the data
   root and the environment name with the user, since every path and every job activation depends on them.

2. **Discover the units.** Call `discover_remote_project_tool` on the project, read the bare `breakdown` back to the
   user, then name a filter or `include_items=True` to list the `unit_path` values the batch will consume.

3. **Prepare the batch.** Call `prepare_batch_tool` with the pipeline, the discovered unit paths, and `host='remote'`.
   Record the returned `batch_id`, since `execute_jobs_tool` takes identifiers and no path.

4. **Confirm the wall time.** Present the `walltime_minutes` default the job script section names, alongside the
   per-job `cores` and `memory_mb` from a `prepare_batch_tool` call with `include_job_descriptors=True`, or from
   `inspect_job_resources_tool` in `/job-planning`, and ask whether the user wants an override. One figure covers
   every allocation of the submission, because the bound exists to stop a run that has stopped progressing rather
   than to describe a stage.

5. **Submit.** Call `execute_jobs_tool` with the recorded identifiers and the confirmed `walltime_minutes`. Read back
   `batch_directory`, `submissions`, and `adopted_jobs`, and tell the user which jobs were adopted rather than
   resubmitted.

6. **Poll the run.** Call `get_processing_status_tool` with `host='remote'` on an interval. Its `active` flag reads
   true while any allocation is outside a terminal state, and its `outcomes` list fills as batches settle and close.

7. **Read a failure at its source.** Pass `detailed=True` on the status read to obtain each allocation's `output_log`
   and `error_log`, which are paths on the server holding that allocation's own diagnostics.

8. **Read the finished state.** Once nothing is outstanding, regenerate on the server and then read through
   `/project-state` or `/dataset-definition` with `host='remote'`, or read the recorded closure outcome the status
   tool reports.

---

## The submission ledger

`<working directory>/remote_state/submission_ledger.yaml` is this machine's durable record of what it submitted, and
every write to it takes a `FileLock` on the sibling `submission_ledger.yaml.lock` with a twenty-second timeout. Reads
take no lock. The file records one `orchestration/ledger.py::SubmissionBatch` per submission, carrying the batch
identifier, the prepared identifiers it covered, the server-side batch directory, the submission timestamp, the wall
time, and one `RemoteSubmission` per accepted allocation.

**The ledger write sits in a `finally` block.** A submission the scheduler rejects partway through still records every
allocation it accepted, and a re-submitted job replaces its earlier entry rather than duplicating it.

**A batch leaves the ledger only through closure.** `orchestration/closure.py::close_settled_batches` retires a batch
when every one of its allocations reached a terminal state, the status query actually observed every one of them, and
`close_batch` succeeded for every prepared identifier the submission covered. An allocation the status map does not
cover counts as unfinished, so a partial query never retires a batch it did not fully observe. A closure that raises
is logged as a warning and the batch stays outstanding, so the next query can try again.

Closure is what makes a finished run answerable. It regenerates the state artifacts on the server, pulls them into
`remote_state/prepared_batches/<batch_id>/`, writes `<batch_id>.outcome.yaml`, and only then retires the prepared
document. A batch absent from the outstanding listing has therefore either finished or was never submitted, and its
answer lives in that outcome file or in a `host='remote'` read of the project's job artifact.

**A live allocation is adopted rather than resubmitted.** Before anything is submitted,
`orchestration/reconcile.py::reconcile_remote_jobs` collects the allocation claimed for each job from two sources and
asks the scheduler for their states. A job whose claimed allocation is not terminal is adopted, and its identifier
seeds the dependency map so a dependent waits on the allocation already running the work. Every other job is
dispatched with its recorded state cleared first.

| Claim source                    | Covers                                                    | Blind spot                                      |
|---------------------------------|-----------------------------------------------------------|-------------------------------------------------|
| The submission ledger           | Allocations this machine submitted, queued ones included  | A batch submitted from another machine          |
| The job's own recorded executor | Every submitter, because the record travels with the data | Appears only once the allocation starts running |

The executor wins where both name an allocation, because it describes a later moment than the record of submitting.
Only the `slurm` executor scheme is honored, and a record naming any other scheme is dispatched with its state cleared.

---

## The remote job script

`server/job.py` composes one shell script per allocation and `Server.submit_job` uploads it, marks it executable, and
submits it. The script carries exactly six unconditional SBATCH directives, in this order.

```text
#!/bin/bash
#SBATCH --cpus-per-task=<cores>
#SBATCH --job-name=<index>-<sanitized name>
#SBATCH --output=<batch directory>/<name>.out
#SBATCH --error=<batch directory>/<name>.err
#SBATCH --mem=<gigabytes>G
#SBATCH --time=<[D-]HH:MM:SS>
[#SBATCH --dependency=afterok:<id>[:<id>…]]
[#SBATCH --kill-on-invalid-dep=yes]

trap 'rm -f <script path>' EXIT
eval $(conda shell.bash hook)
conda init bash
source activate <environment>

set -eo pipefail
<one slf command>
```

The two conditional directives appear together whenever the job names a prerequisite. Every upstream allocation
identifier joins one `afterok:` prefix separated by colons. The kill directive turns a dependency that can never be
satisfied into a terminal state a status query can report, rather than a queue entry that waits forever. No other
directive is ever emitted, so a partition, an account, a QOS, a node count, and a GPU request are all outside what this
library expresses.

**Resource requests come from the job's own estimate.** `--cpus-per-task` takes the job's core weight and `--mem` takes
its memory estimate rounded up to whole gigabytes and floored at one. `/job-planning` owns the model that produces both
figures. `--time` takes one figure for the whole submission, `orchestration/remote.py::REMOTE_JOB_WALLTIME_MINUTES`,
which is 480 minutes when the caller names none.

**The environment activation sits outside error checking.** A conda hook exports shell state whose exit status does not
describe whether the environment is usable, so `set -eo pipefail` is emitted after the preamble and the work fails
there instead. The script removes itself through an exit trap rather than a trailing command, so its exit status stays
the status of the work it ran and the scheduler sequences dependents on the truth.

**Each script runs the same `slf` command a local run would.** The command renderer is the dispatch table both backends
share, so a job runs the same stage at the same width whichever way it is executed. That is why a pipeline is proven on
one session locally before a project-wide remote batch. A defect in the stage reproduces on this machine in one job,
where the tracker, the error message, and the output are all directly readable. The alternative is a wave of failed
allocations whose only diagnostics are log files on the server.

---

## Remote versus local divergences

| Concern                           | `host='local'`                                        | `host='remote'`                                                   |
|-----------------------------------|-------------------------------------------------------|-------------------------------------------------------------------|
| Where a run's state lives         | A process-global that dies with the server            | The ledger plus the scheduler, both surviving a restart           |
| Concurrent batches                | Exactly one, a second dispatch is refused             | Unbounded, the ledger tracks many at once                         |
| Budget arguments on execute       | `core_budget_override` and `memory_budget_mb` honored | Both ignored, each job requests its own allocation                |
| Wall time on execute              | Ignored                                               | `walltime_minutes`, at the job script's own default               |
| Concurrency ceilings              | Enforced by the admission engine                      | Never expressed to the scheduler                                  |
| A job already recorded as running | Rerun, since the record describes a dead pool         | Adopted, with dependents wired to the live allocation             |
| Progress source                   | The processing trackers                               | The scheduler alone, no tracker is opened from this machine       |
| Tracker paths on job descriptors  | Real paths                                            | Empty, by design                                                  |
| Status vocabulary                 | Lowercase tracker labels                              | Uppercase scheduler states                                        |
| Per-job unit key                  | `session_path`                                        | `unit_path`                                                       |
| Cancellation                      | Cooperative, in-flight jobs finish                    | Queued and running killed alike, dependents cascade               |
| Cleaning guard                    | Refused while a batch runs on this machine            | Never refused, even with allocations in flight                    |
| Cleaning figures                  | Measured directly                                     | Parsed from the server command's output, unparsable lines dropped |
| Reset failures                    | Warn per unit and continue                            | A non-zero server exit raises                                     |

**The cleaning divergence is the dangerous one.** The running-batch guard covers this machine's pool alone, so a remote
clean is accepted while allocations are in flight and will remove the output and the tracker of an allocation still
running, with no warning from the tool. Read `get_processing_status_tool` with `host='remote'` and confirm that nothing
is outstanding before cleaning anything on the server.

---

## Error routing

| Message                                                                                             | Raised by                      | Remedy                                                                       |
|-----------------------------------------------------------------------------------------------------|--------------------------------|------------------------------------------------------------------------------|
| `Unknown unit kind '{unit_kind}'. Available: ['dataset', 'session'].`                               | `discover_remote_project_tool` | Name one of the two kinds, or omit the parameter                             |
| `Unable to discover the '{project}' project on the compute server. Only the final component …`      | `discover_remote_project_tool` | Pass a value whose final path component names the project                    |
| `Unable to discover the '{project}' project on the compute server. The server holds no directory …` | `discover_remote_project_tool` | Confirm the data root through `/server-configuration`, then the name         |
| `Unable to search {path} on the remote compute server. The search reached only part of the tree …`  | `discover_remote_project_tool` | Retry with `include_sessions=False`, which holds the search to dataset depth |
| `Unknown scheduler view '{view}'. Available: ['accounting', 'queue'].`                              | `read_scheduler_jobs_tool`     | Name `accounting` or `queue`                                                 |
| `Unable to reach the compute server's scheduler. {exception}`                                       | `read_scheduler_jobs_tool`     | Read the wrapped exception, which names the connection or the command        |
| `Unable to read the scheduler's {view} records. The command '{command}' exited with the status …`   | `read_scheduler_jobs_tool`     | The message carries the exact command, which is safe to show the user        |
| `No scheduler record has '{field}' in {unknown}. Available: {sorted}.`                              | `read_scheduler_jobs_tool`     | Correct the `job_names` or `states` value against the listed set             |

An authentication failure is fatal on the first attempt, while any other connection failure retries thirty times at two
second intervals before the transport reports the server unreachable. Both surface through whichever tool opened the
connection.

---

## Related skills

The `video:` and `communication:` entries below resolve through the ataraxis marketplace. Every other entry resolves
inside the sollertia marketplace.

| Skill                            | Relationship                                                                     |
|----------------------------------|----------------------------------------------------------------------------------|
| `/server-configuration`          | Prerequisite: the access record every remote connection is built from            |
| `/batch-processing`              | Peer: prepares, dispatches, monitors, and cleans, on either host                 |
| `/job-planning`                  | Upstream: the estimates that size every allocation this skill submits            |
| `/project-state`                 | Downstream: the manifest and job artifact a remote read mirrors                  |
| `/dataset-definition`            | Downstream: dataset state, generated on the server and mirrored back             |
| `/dataset-forging`               | Consumer: the forging pipeline's own prerequisites, on either host               |
| `/cli-reference`                 | Reference: the `slf` command surface each generated job script invokes           |
| `/pipeline`                      | Context: where a remote run sits in the end-to-end workflow                      |
| `/forging-mcp-environment-setup` | Prerequisite: MCP connectivity and the response contract                         |
| `assets:working-directory`       | Upstream: the working directory holding the ledger, the mirror, and the registry |
| `assets:session-discovery`       | Upstream: session roots on this machine, where a local batch takes its paths     |
| `video:log-processing`           | Reference: the upstream archive format a video job reads                         |
| `communication:log-processing`   | Reference: the upstream archive format a microcontroller job reads               |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Remote prerequisites:
- [ ] sollertia-forgery MCP server is connected
- [ ] The server configuration was read through /server-configuration before any host='remote' call
- [ ] The data root and the environment name were confirmed with the user

Path provenance:
- [ ] Every server-side unit path came from discover_remote_project_tool or from a generate or plan response
- [ ] No path reported by a read_* response was fed back into a tool with host='remote'
- [ ] A project argument was passed as its final component, or as a path whose final component names the project
- [ ] An empty discovery page was checked against the bare call's breakdown before being reported as absent

Submission and monitoring:
- [ ] The batch was prepared with host='remote' and its batch_id was recorded
- [ ] The wall time was confirmed with the user before execute_jobs_tool was called
- [ ] Adopted jobs were reported to the user separately from newly submitted allocations
- [ ] Progress was read with get_processing_status_tool at host='remote', never inferred from a tracker
- [ ] A failed allocation was investigated through its output_log and error_log on the server
- [ ] Nothing was cleaned on the server until a remote status read reported no outstanding allocation

Boundaries:
- [ ] Credential authoring was routed to /server-configuration rather than described here
- [ ] Generic batch mechanics were routed to /batch-processing rather than restated
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
