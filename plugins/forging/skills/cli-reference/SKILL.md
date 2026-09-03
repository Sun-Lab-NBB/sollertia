---
name: cli-reference
description: >-
  Documents the human-facing slf command-line interface of the sollertia-forgery library. Covers the root group, the
  four subgroups, every leaf command and option with its short form, long form, type, default, and effect, the MCP tool
  each command maps to, and how the CLI path diverges from the MCP path. Use when a user asks what an slf command or
  option does, when a rendered remote job command line must be read, or when the MCP server is unavailable and the user
  must be told what to run by hand.
user-invocable: false
---

# CLI reference

> **The `slf` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Print the command, ask the user to run it, and
> ask them to paste the output back.

**The one exemption** is `--help`, which `/forging-mcp-environment-setup` owns. `slf --help` and `slf COMMAND --help`
may be run, and no other `slf` invocation is exempt.

---

## Scope

**Covers:**
- The complete `slf` command surface: every Click node, its purpose, and the MCP tool it maps to
- Every declared option: short form, long form, type, default, required, flag, or repeatable status, and effect
- Per-command failure modes, the exception each raises, and the exit code the user observes
- Which CLI commands have no MCP equivalent, which MCP tools have no CLI equivalent, and where paired surfaces differ
- The one-unit `slf` command lines the remote scheduler renders, and what to hand a user when MCP cannot be restored

**Does not cover:**
- The batch workflow, budgets, status reading, and batch remediation. Owned by `/batch-processing`.
- The planning tools behind `slf plan` and the resource model, owned by `/job-planning`, and the manifest and project
  jobs tools behind `slf manifest`, owned by `/project-state`.
- Dataset definition and state semantics behind `slf forge`. Owned by `/dataset-definition` and `/dataset-forging`.
- The server configuration payload, the scheduler reads, and what each resolved verdict and remediation means. Owned
  by `/server-configuration` and `/remote-execution`, with MCP recovery owned by `/forging-mcp-environment-setup`.

**Handoff rules:** If the user wants an operation performed rather than explained, use the MCP tools and invoke the
owning skill. If the tools are unavailable, invoke `/forging-mcp-environment-setup` first, and fall back to the handoff
table below only after the server cannot be restored.

---

## Agent requirements

You MUST answer CLI questions from this skill or from `slf COMMAND --help`, never from memory. When a user's report
disagrees with this reference, ask them to run that help and read the installed build's answer.

The `slf` CLI is not only the human path but also the remote execution substrate. `orchestration/dispatch.py` renders
every prepared job back into a one-unit `slf` command line, and `orchestration/remote.py::_submit_ordered_jobs` joins
that argv into the SBATCH script, so read a rendered job script as this reference documents it.

---

## Command surface

The CLI declares twenty-six Click nodes: the root group, four subgroups, and twenty-one leaf commands, entered through
`slf = "sollertia_forgery.interfaces.entry_points:slf_cli"`. The root group's docstring states the rule that shapes the
whole surface, that the acquisition system is inferred from the data, so no command takes a system selector.

| Click node                    | Kind    | Purpose                                                                | MCP equivalent                                |
|-------------------------------|---------|------------------------------------------------------------------------|-----------------------------------------------|
| `slf`                         | group   | Entry point. Dispatches to the subcommands and prints the listing      | None, dispatch only                           |
| `slf mcp`                     | command | Starts the MCP server over the named transport                         | None, this command IS the server              |
| `slf omp`                     | command | Reports and links the macOS OpenMP runtime the Numba layer loads       | None, and none is possible                    |
| `slf plan`                    | group   | Dispatches the job cost planning subcommands                           | None, dispatch only                           |
| `slf plan session`            | command | Records what every processing job of each named session will cost      | `plan_session_jobs_tool`                      |
| `slf plan dataset`            | command | Records what every forging job of each named dataset will cost         | `plan_dataset_jobs_tool`                      |
| `slf plan project`            | command | Projects every plan cache under a project into one table at its root   | `generate_project_plan_tool`                  |
| `slf process`                 | group   | Parses the shared session options its four subcommands consume         | None, dispatch only                           |
| `slf process video`           | command | Runs the frame timestamp, tracking, and motion energy stages           | The batch pair, `pipeline="video"`            |
| `slf process microcontroller` | command | Extracts and parses the microcontroller module log archives            | The batch pair, `pipeline="microcontroller"`  |
| `slf process runtime`         | command | Decodes and parses the acquisition runtime log archive                 | The batch pair, `pipeline="runtime"`          |
| `slf process two-photon`      | command | Runs the single-recording two-photon processing stages                 | The batch pair, `pipeline="two_photon"`       |
| `slf forge`                   | command | Defines a dataset hierarchy and runs its outstanding forging jobs      | `define_forging_dataset_tool` plus the pair   |
| `slf manifest`                | group   | Parses the shared project path its two subcommands consume             | None, dispatch only                           |
| `slf manifest create`         | command | Writes the project jobs table and then the project manifest snapshot   | `generate_project_manifest_tool`              |
| `slf manifest print`          | command | Renders the notes view, the summary view, or both, to the terminal     | `read_project_manifest_tool`                  |
| `slf checksum`                | command | Verifies or regenerates one session's raw data integrity checksum      | The batch pair, `pipeline="checksum"`         |
| `slf dataset-state`           | command | Snapshots each named dataset's forging job state to the dataset root   | `generate_dataset_state_tool`                 |
| `slf reset`                   | command | Returns tracked jobs of the named units to the scheduled state         | `reset_processing_jobs_tool`                  |
| `slf clean`                   | command | Removes a pipeline's output and processing tracker for the named units | `clean_processing_output_tool`                |
| `slf server`                  | group   | Dispatches the remote compute server subcommands                       | None, dispatch only                           |
| `slf server configure`        | command | Writes the remote server access configuration file                     | `write_server_configuration_tool`             |
| `slf server print`            | command | Prints the remote scheduler's accounting history, its queue, or both   | `read_scheduler_jobs_tool`                    |
| `slf server discover`         | command | Prints the sessions the named project holds on the remote server       | `discover_remote_project_tool`                |
| `slf server batches`          | command | Resolves the outstanding batches and each allocation's verdict         | `get_processing_status_tool`, `host='remote'` |
| `slf server retire-batch`     | command | Remediates named batches and drops them from the submission ledger     | `retire_remote_batches_tool`                  |

"The batch pair" means `prepare_batch_tool` and `execute_jobs_tool` taken together, under the named `pipeline` value,
and every one of those mappings carries the divergences documented below. **Note on `-h`:** the CLI leaves Click's
`help_option_names` at its `["--help"]` default, so `-h` is never a help alias, and on `slf server configure` it binds
to `--host` and consumes the next token as a host name.

---

## Option reference

Every option below is declared in `interfaces/entry_points.py`, `interfaces/plan.py`, `interfaces/process.py`,
`interfaces/forge.py`, `interfaces/manage.py`, or `interfaces/server.py`. "Required" means Click rejects the invocation
without it (exit code 2). Every node sets or inherits `max_content_width` 120, so every `--help` wraps at that width.

### `slf mcp`

| Short | Long          | Type     | Default   | Form     | Effect                                                     |
|-------|---------------|----------|-----------|----------|------------------------------------------------------------|
| `-t`  | `--transport` | `Choice` | `"stdio"` | optional | Transport. Choice of `stdio`, `sse`, and `streamable-http` |

The choice is case-insensitive. `stdio` disables the console, because one echoed status line inside the shared
JSON-RPC stream makes a message unparsable, while the other two keep the console on and serve the network.

### `slf omp`

| Short | Long       | Type   | Default | Form     | Effect                                                                         |
|-------|------------|--------|---------|----------|--------------------------------------------------------------------------------|
| `-s`  | `--source` | `Path` | `None`  | optional | The runtime to link. Omitted, the command searches for one. Must exist         |
| `-t`  | `--target` | `Path` | `None`  | optional | The path receiving the link. Omitted, it derives one from the loader's default |
| `-f`  | `--force`  | flag   | `False` | flag     | Links a runtime even on a host whose OpenMP runtime already loads              |
| `-y`  | `--yes`    | flag   | `False` | flag     | Creates the resolved link. Without it the command changes nothing              |

Omitting `-s` searches the macOS package manager directories, then the active conda environment, then the runtimes
vendored inside installed Python distributions, taking the first candidate that exists.

### `slf plan`

| Command              | Short | Long                | Type   | Default    | Form                 | Effect                                                           |
|----------------------|-------|---------------------|--------|------------|----------------------|------------------------------------------------------------------|
| `session`            | `-sp` | `--session-path`    | `Path` | (required) | required, repeatable | A session root to plan. Repeat per session                       |
| `dataset`            | `-dp` | `--dataset-path`    | `Path` | (required) | required, repeatable | A dataset root to plan. Repeat per dataset                       |
| `session`, `dataset` | `-rp` | `--regenerate-plan` | flag   | `False`    | flag                 | Re-estimates every job rather than only the ones the plan lacks  |
| `project`            | `-pp` | `--project-path`    | `Path` | (required) | required             | The project root whose plan caches are collected. Not repeatable |

The group itself declares no options, and `slf plan project` prints the written projection path as a bare line.

### `slf process` group options

| Short | Long             | Type   | Default | Form     | Effect                                                                 |
|-------|------------------|--------|---------|----------|------------------------------------------------------------------------|
| `-sp` | `--session-path` | `Path` | `None`  | optional | The session root to process. Enforced by the subcommand, not by Click  |
| `-id` | `--job-id`       | `str`  | `None`  | optional | Runs ONLY the job whose hexadecimal identifier matches (remote mode)   |
| `-w`  | `--workers`      | `int`  | `-1`    | optional | The per-job core allotment. `-1` resolves automatically, `1` is serial |
| `-np` | `--no-progress`  | flag   | `False` | flag     | Suppresses the progress bar. The bar is displayed by default           |

**All four are parsed on the group**, so they must be supplied **before** the subcommand name, as in
`slf process -sp <session> video`. Omitting `-sp` raises a Click usage error at exit code 2, and placing it after the
subcommand name aborts on the unknown option.

### `slf process` subcommands

| Command      | Short | Long              | Type  | Default | Form     | Effect                                                                               |
|--------------|-------|-------------------|-------|---------|----------|--------------------------------------------------------------------------------------|
| `video`      | `-ts` | `--timestamp`     | flag  | `False` | flag     | Runs the frame timestamp stage over each camera's archive                            |
| `video`      | `-tr` | `--track`         | flag  | `False` | flag     | Post-processes externally produced pose predictions                                  |
| `video`      | `-en` | `--energy`        | flag  | `False` | flag     | Measures each recording into a per-frame movement signal                             |
| `video`      | `-tc` | `--target-camera` | `int` | `-1`    | optional | The camera source ID the timestamp and energy stages cover. `-1` covers every camera |
| `two-photon` | `-b`  | `--binarize`      | flag  | `False` | flag     | Runs the binarization stage                                                          |
| `two-photon` | `-r`  | `--register`      | flag  | `False` | flag     | Runs the per-plane registration stage                                                |
| `two-photon` | `-p`  | `--process`       | flag  | `False` | flag     | Runs the per-plane processing stage                                                  |
| `two-photon` | `-cb` | `--combine`       | flag  | `False` | flag     | Runs the multi-plane combination stage                                               |
| `two-photon` | `-tp` | `--target-plane`  | `int` | `-1`    | optional | The imaging plane the per-plane stages cover. `-1` covers every plane                |

`slf process microcontroller` and `slf process runtime` declare no options of their own. **Note on the stage flags:**
naming none of them runs every stage, all three on `video` and all four in sequence on `two-photon`. Every stage flag,
`--target-camera`, and `--target-plane` is ignored when `-id` is supplied, since remote mode selects by identifier.

### `slf forge`

| Short | Long                | Type   | Default    | Form       | Effect                                                             |
|-------|---------------------|--------|------------|------------|--------------------------------------------------------------------|
| `-dn` | `--dataset-name`    | `str`  | (required) | required   | The name of the dataset to create or use                           |
| `-pp` | `--project-path`    | `Path` | (required) | required   | The project root holding the animal and session data directories   |
| `-s`  | `--session`         | `str`  | `()`       | repeatable | A session the dataset must contain. One the dataset lacks is added |
| `-id` | `--job-id`          | `str`  | `None`     | optional   | Runs ONLY the job whose hexadecimal identifier matches             |
| `-w`  | `--workers`         | `int`  | `-1`       | optional   | Workers for the multi-day and assembly stages. `-1` is automatic   |
| `-f`  | `--force-recreate`  | flag   | `False`    | flag       | Deletes the whole dataset hierarchy and rebuilds it                |
| `-ra` | `--recreate-animal` | `str`  | `()`       | repeatable | Rebuilds one animal already in the dataset, leaving the rest alone |
| `-np` | `--no-progress`     | flag   | `False`    | flag       | Suppresses the progress bars. They are displayed by default        |

**Note on the two-phase body:** an invocation naming `-s`, `-f`, or `-ra` defines the hierarchy first through
`forging/pipeline.py::define_forging_dataset` and then runs the outstanding jobs, while an invocation naming none of the
three skips definition and runs jobs alone, as the scheduler-rendered command deliberately does. **Note on the freeze
policy:** an animal already in the dataset is frozen, because widening its session set invalidates the outputs already
forged for it. Name that animal with `-ra` to rebuild it from the provided sessions.

### `slf manifest`

| Node     | Short | Long             | Type   | Default | Form     | Effect                                                     |
|----------|-------|------------------|--------|---------|----------|------------------------------------------------------------|
| group    | `-pp` | `--project-path` | `Path` | `None`  | optional | The project root. Enforced by the subcommand, not by Click |
| `create` | `-np` | `--no-progress`  | flag   | `False` | flag     | Suppresses the preamble and completion messages            |
| `print`  | `-a`  | `--animal`       | `str`  | `None`  | optional | One animal to print. Omitted, every participating animal   |
| `print`  | `-n`  | `--notes`        | flag   | `False` | flag     | Prints the experimenter notes view of the manifest         |
| `print`  | `-s`  | `--summary`      | flag   | `False` | flag     | Prints the data processing view of the manifest            |

**`-pp` is parsed on the group**, so it must precede the subcommand name, as in `slf manifest -pp <project> create`,
and `slf manifest create` overwrites any existing manifest with a fresh snapshot.

### `slf checksum`

| Short | Long                    | Type   | Default    | Form     | Effect                                                         |
|-------|-------------------------|--------|------------|----------|----------------------------------------------------------------|
| `-sp` | `--session-path`        | `Path` | (required) | required | The processed session root. Single valued, not repeatable      |
| `-rc` | `--regenerate-checksum` | flag   | `False`    | flag     | Recalculates the stored checksum rather than verifying it      |
| `-w`  | `--workers`             | `int`  | `-1`       | optional | Hashing workers. Below 1 requests every core minus the reserve |
| `-np` | `--no-progress`         | flag   | `False`    | flag     | Suppresses the preamble message and progress bar               |

`slf checksum` is a top-level command rather than a `process` subcommand, so it shares none of that group's options,
declares no job identifier, and writes into the session's raw data directory alone.

### `slf dataset-state`

| Short | Long             | Type   | Default    | Form                 | Effect                                                                                                       |
|-------|------------------|--------|------------|----------------------|--------------------------------------------------------------------------------------------------------------|
| `-dp` | `--dataset-path` | `Path` | (required) | required, repeatable | A dataset root to snapshot. Repeat once per dataset. The written artifact path prints for each, one per line |

### `slf reset` and `slf clean`

| Command | Short | Long          | Type     | Default    | Form                 | Effect                                           |
|---------|-------|---------------|----------|------------|----------------------|--------------------------------------------------|
| both    | `-p`  | `--pipeline`  | `Choice` | (required) | required             | The pipeline to act on. Six values, listed below |
| both    | `-up` | `--unit-path` | `Path`   | (required) | required, repeatable | A processing unit. Repeat once per unit          |
| `reset` | `-id` | `--job-id`    | `str`    | `()`       | repeatable           | A tracked job to reset. Omit to reset every one  |

A unit is a session root for a session pipeline and a dataset root for `forging`. The choice list is computed at import
from `orchestration/dispatch.py::BATCH_PIPELINES` and accepts exactly `checksum`, `forging`, `microcontroller`,
`runtime`, `two_photon`, and `video`, case-insensitively, rejecting `manifest`, which is a `ProcessingPipelines` member
but not a batch pipeline. The pipeline value spells two-photon with an underscore while the `slf process` subcommand
spells it with a hyphen. **Note on `-id`:** a job identifier derives from the job name and the specifier alone, so two
units of one project share the identifier of the same stage, and every named identifier is applied to every named unit.
A caller holding per-unit identifiers passes one unit per invocation rather than a flat set.

### `slf server`

| Command        | Short | Long                     | Type  | Default    | Form                 | Effect                                                                                         |
|----------------|-------|--------------------------|-------|------------|----------------------|------------------------------------------------------------------------------------------------|
| `configure`    | `-u`  | `--username`             | `str` | (required) | required             | The username used for server authentication                                                    |
| `configure`    | `-p`  | `--password`             | `str` | (prompted) | optional             | The password. Prompted with hidden double entry if absent                                      |
| `configure`    | `-h`  | `--host`                 | `str` | (required) | required             | The host name or IP address of the server                                                      |
| `configure`    | `-r`  | `--root`                 | `str` | (required) | required             | The absolute remote path to the root Sollertia data directory                                  |
| `configure`    | `-e`  | `--environment`          | `str` | (required) | required             | The shared remote conda environment holding the library                                        |
| `print`        | `-j`  | `--job-data`             | flag  | `False`    | flag                 | Displays the accounting history through `sacct`                                                |
| `print`        | `-q`  | `--queue`                | flag  | `False`    | flag                 | Displays the queue status through `squeue`                                                     |
| `print`        | `-u`  | `--user`                 | `str` | `None`     | optional             | Filters to one user. `all` covers every user                                                   |
| `print`        | `-id` | `--job-id`               | `str` | `None`     | optional             | One job to report, in whichever of the two views was requested. Bypasses user and date filters |
| `print`        | `-st` | `--start-time`           | `str` | `None`     | optional             | Keeps jobs started on or after this date. `-j` view only                                       |
| `print`        | `-et` | `--end-time`             | `str` | `None`     | optional             | Keeps jobs ended on or before this date. `-j` view only                                        |
| `discover`     | `-p`  | `--project`              | `str` | (required) | required             | The project whose sessions to discover under the data root                                     |
| `batches`      | `-b`  | `--batch-id`             | `str` | `()`       | repeatable           | One outstanding batch to report. Omit to report every one                                      |
| `batches`      | `-a`  | `--allocations`          | flag  | `False`    | flag                 | Adds one row per resolved allocation beneath the batch table                                   |
| `retire-batch` | `-b`  | `--batch-id`             | `str` | (required) | required, repeatable | One outstanding batch to remediate and drop from the ledger                                    |
| `retire-batch` | `-f`  | `--force`                | flag  | `False`    | flag                 | Remediates batches holding an allocation that resolves as running, cancelling each one first   |
| `retire-batch` | `-do` | `--drop-without-outcome` | flag  | `False`    | flag                 | Drops the entries when what their jobs recorded cannot be snapshotted                          |

The group declares no options. Supplying `-p` skips the prompt and leaks the password into shell history, so hand the
user `slf server configure` without it. The command guards no existing file, so a repeat call replaces the stored
configuration outright, and `-u` on `slf server print` defaults to the configured username. `retire-batch` requires
`-b`, so Click rejects a bare invocation with exit 2 rather than letting the tool's own no-identifier error be reached.

### Short forms that collide

The same short form means different things in different commands, so quote the long form whenever one of these is
handed to a user. `-p` expands to `--password` on `server configure`, `--pipeline` on `reset` and `clean`, `--process`
on `process two-photon`, and `--project` on `server discover`. `-s` expands to `--source` on `omp`, `--session` on
`forge`, and `--summary` on `manifest print`. `-t` expands to `--transport` on `mcp` and `--target` on `omp`. `-f`
expands to `--force` on `omp` and on `server retire-batch`, but `--force-recreate` on `forge`. `-b` expands to
`--binarize` on `process two-photon` and `--batch-id` on `server batches` and `server retire-batch`. `-a` expands to
`--animal` on `manifest print` and `--allocations` on `server batches`. `-u` expands to `--username` on
`server configure` and `--user` on `server print`, and `-r` to `--root` there and `--register` on `process two-photon`.

---

## Command behavior and failure modes

### How a failure reaches the user

No command body is wrapped in a catching decorator, so every failure the body raises leaves a Python traceback and a
non-zero exit. The message text identifies the fault, so ask the user to paste the traceback rather than the status.

| Path                                                                       | Mechanism                                 | Observable outcome                         |
|----------------------------------------------------------------------------|-------------------------------------------|--------------------------------------------|
| `slf omp` finding no runtime to link                                       | `raise SystemExit(1)`                     | Exit 1, the only deliberate non-zero exit  |
| A missing required option, a rejected choice, or a path that must exist    | Click's own parameter validation          | Usage message, exit 2                      |
| `-sp` omitted before a `process` subcommand, `-pp` before a `manifest` one | `click.UsageError` from the shared object | Usage message naming the omission, exit 2  |
| Any guard or pipeline error on every other command                         | The exception propagates uncaught         | Traceback, non-zero exit                   |
| `slf plan session` or `slf plan dataset` meeting an unplannable unit       | Caught per path inside the loop           | `<name>: planned nothing. <error>`, exit 0 |
| `slf reset` or `slf clean` meeting an unloadable unit                      | Warning logged, the unit is skipped       | Exit 0                                     |

`slf dataset-state` catches nothing, so one bad dataset path aborts the whole loop partway through. There is no
version, verbosity, or global host option anywhere: `slf` addresses this machine, and the remote host is reachable only
through `slf server`.

### The macOS OpenMP precondition

`shared_assets/openmp.py::verify_openmp_runtime` runs at the head of every pipeline except `manifest`, so on macOS
`slf process`, `slf forge`, and `slf checksum` raise `RuntimeError` naming `slf omp` as the remedy before doing any work
when `libomp.dylib` does not load. The MCP server reports healthy on such a host while every parallel job fails, which
makes this a first-order environment diagnostic. A bare `slf omp` is always a dry run, and writing the link usually runs
through sudo. The `## OpenMP runtime prerequisite` section of `/forging-mcp-environment-setup` owns its four outcomes.

### The job identifier contract on `slf process` and `slf forge`

| Case                  | `-id`    | Behavior                                                                      |
|-----------------------|----------|-------------------------------------------------------------------------------|
| Local mode            | omitted  | Discovers and runs every available job for the unit, honoring the stage flags |
| Remote mode           | supplied | Runs only the matching job, chosen entirely by the identifier                 |
| `slf process runtime` | either   | Always runs its single job. The identifier is never forwarded                 |

`interfaces/process.py::runtime_command` does not pass the shared job identifier to
`runtime/pipeline.py::run_runtime_processing_pipeline`, which declares no such parameter, so `-id` is silently ignored
there. Local mode on `slf forge` skips every succeeded stage, while remote mode runs the named job regardless.

### `slf manifest print`

Three guards fire in order, and each raises uncaught. Giving neither `-n` nor `-s` raises `ValueError`. No manifest on
disk raises `FileNotFoundError` naming the generation command, because printing reads an existing snapshot and never
regenerates one. An animal that did not participate raises `ValueError`. Both views print together, notes first.

### `slf clean` holds no running-batch guard

`clean_processing_output_tool` refuses while a batch is running locally, and `slf clean` performs no such check, so it
will remove output from under a running batch. Confirm no batch is executing before handing the command over. Each
removed path is reported as the bytes it held, a single space, then the path. Cleaning `checksum` removes its tracker
alone, since that pipeline owns no output directory, while cleaning `forging` removes the whole dataset hierarchy.

### The four `slf server` reporting commands

`slf server print` requires at least one of `-j` and `-q`, raising `ValueError` when neither is given, and prints both
views in one call when both are given, accounting first. Both views read `-id`, and only `-j` reads `-st` and `-et`,
which `-q` accepts and ignores. A non-zero `sacct` or `squeue` return raises `RuntimeError` quoting the scheduler's own
error stream, and an empty result logs a warning and returns. `slf server discover` prints one line per discovered
session, reporting no forged dataset and no absolute path, so it cannot supply the paths a `host="remote"` tool takes.

`slf server batches` prints a `batch_id | progress | outstanding_h | jobs | running | stranded | gone | pipelines`
table, where `outstanding_h` reads in hours or `unknown`, `running`, `stranded`, and `gone` count the allocations
carrying each resolved verdict, and `progress` is one of `progressing`, `stalled`, and `awaiting_closure`. A
`scheduler_read_error` prints first as a WARNING, `-a` adds a
`batch_id | allocation | job_name | specifier | scheduler | tracker | verdict | remediation` table beneath it, and each
stalled batch then prints one WARNING line naming the allocations both scheduler records disclaim and the remedy, which
names `slf server retire-batch`. A read that closes every batch it covered prints that message as a warning and stops.

`slf server retire-batch` prints the snapshot error if any, one SUCCESS line per retired batch with the allocations it
held, then an `allocation | job_name | specifier | verdict | cancelled | reset | snapshot | dropped` table, one row per
allocation with the four booleans naming which steps ran, and finally the message and the local outcome directory.
Where their tool answers with an error rather than a report, both commands raise `RuntimeError` through `console.error`
and it propagates uncaught, which is how a `running` allocation without `--force`, and a failed snapshot without
`--drop-without-outcome`, reach the user.

---

## How the CLI diverges from the MCP path

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

### CLI commands with no MCP equivalent

- `slf mcp` starts the server, so it is CLI-only by definition.
- `slf omp` has no tool and cannot usefully have one, since writing the link normally needs sudo. An agent hitting
  that error hands the operator a shell command.
- `slf manifest print` renders two fixed human views no tool renders, though `read_project_manifest_tool` with
  `detailed=True` returns the underlying `notes` field per session.

### MCP tools with no CLI equivalent

- **Most of the batch machinery.** `execute_jobs_tool`, `cancel_processing_tool`, `inspect_job_resources_tool`,
  `list_prepared_batches_tool`, `forget_prepared_batches_tool`, and `read_resource_model_tool` have no CLI surface, and
  `get_processing_status_tool` has one only for `host='remote'`. The CLI runs one unit synchronously in the calling
  process, so it has no local batch identity to dispatch, poll, or forget, and a running command is stopped with
  Ctrl-C. Resolving and remediating a remote batch are the exception, because the ledger outlives every process.
- **Every structured read.** `read_project_plan_tool`, `read_project_jobs_tool`, `get_manifest_status_tool`,
  `read_dataset_state_tool`, `list_project_datasets_tool`, and `read_server_configuration_tool` have no CLI surface.
  The CLI writes each state artifact and prints its path, and reads one back only through `slf manifest print`.

### Where a paired command and tool differ

| CLI command                            | Nearest MCP tool                                  | Divergence                                                                                                                                                                                                                          |
|----------------------------------------|---------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `slf plan session`, `slf plan dataset` | `plan_session_jobs_tool`                          | Same batch shape. The CLI echoes one summary line per unit and one line per skipped pipeline or refused sizing, where the tool returns totals and a structured `unsized_jobs` mapping. No host on the CLI                           |
| `slf plan project`                     | `generate_project_plan_tool`                      | The CLI echoes the projected job count with the planned and unplanned unit counts, then prints the path as its one machine-readable line, where the tool returns the resource totals. No host on the CLI                            |
| `slf process <pipeline>`               | The batch pair                                    | The CLI runs one session synchronously. The stage flags, `-tc`, and `-tp` have no tool counterpart, and the budgets and replan controls have no CLI counterpart                                                                     |
| `slf forge`                            | `define_forging_dataset_tool` plus the batch pair | The CLI fuses definition and dispatch into one command, and names the dataset by name plus project path where the batch pair names it by dataset root                                                                               |
| `slf checksum`                         | The batch pair, `pipeline="checksum"`             | One session per call against a list per call. `-rc` is the only pipeline option that survives into the tool `options` dictionary                                                                                                    |
| `slf manifest create`                  | `generate_project_manifest_tool`                  | Same operation. The CLI takes no host and returns no snapshot summary                                                                                                                                                               |
| `slf manifest print`                   | `read_project_manifest_tool`                      | Fixed terminal views against a filtered, paginated payload. The CLI has no pagination and no filters beyond `-a`                                                                                                                    |
| `slf dataset-state`                    | `generate_dataset_state_tool`                     | Same batch shape. The CLI prints the written path per dataset and takes no host                                                                                                                                                     |
| `slf reset`                            | `reset_processing_jobs_tool`                      | The closest pairing on the surface. The CLI reports the true count of reset identifiers where the tool reports an upper bound                                                                                                       |
| `slf clean`                            | `clean_processing_output_tool`                    | The tool refuses while a local batch runs and the CLI does not. The CLI names units where the tool names the same argument as session paths                                                                                         |
| `slf server configure`                 | `write_server_configuration_tool`                 | Five discrete options with an interactive prompt against one payload dictionary. The CLI always replaces where the tool demands an overwrite flag                                                                                   |
| `slf server print`                     | `read_scheduler_jobs_tool`                        | The CLI selects with two independent flags and may print both views, while the tool takes one view string and answers one. The CLI's job identifier scopes whichever views were requested, as the tool's list does for its one view |
| `slf server discover`                  | `discover_remote_project_tool`                    | The CLI prints sessions only, unfiltered and unpaginated. The tool also covers forged datasets and returns the absolute server-side paths                                                                                           |
| `slf server batches`                   | `get_processing_status_tool`, `host='remote'`     | The same call, with `limit=0`. Without `-a` the CLI prints the batch entries alone, so the per-allocation page a named `-b` produces is computed and never shown                                                                    |
| `slf server retire-batch`              | `retire_remote_batches_tool`                      | The same call with the same two waivers. The CLI requires `-b` at the parser, and prints the per-allocation remediation table and the outcome directory rather than returning the recorded outcomes                                 |

### The three rules behind the table

1. **Unit cardinality.** Every pipeline command runs one unit per invocation while every MCP processing tool takes a
   list. The management commands are the exception, since `plan session`, `plan dataset`, `dataset-state`, `reset`,
   and `clean` all repeat their unit option, as do the two `-b` options on `slf server`.
2. **Execution model.** The CLI executes synchronously with no batch identity, no budgets, and no cancellation, while
   the MCP surface is prepare, then execute, then read, mediated by a durable batch identifier.
3. **Host reach.** No command outside `slf server` takes a host, and almost every tool does.

### The rendered command lines

`orchestration/dispatch.py` renders each prepared job into the argv below and `orchestration/remote.py` joins it into
the SBATCH script, with the session commands sharing the preamble `slf process -sp <unit> -w <cores> -np`.

| Pipeline          | Rendered command                                                              |
|-------------------|-------------------------------------------------------------------------------|
| `checksum`        | `slf checksum -sp <unit> -w <cores> -np`, plus `-rc` when the job requests it |
| `runtime`         | `<preamble> runtime`, the only session command rendering no `-id`             |
| `microcontroller` | `<preamble> -id <job_id> microcontroller`                                     |
| `video`           | `<preamble> -id <job_id> video`                                               |
| `two_photon`      | `<preamble> -id <job_id> two-photon`                                          |
| `forging`         | `slf forge -dn <name> -pp <parent> -id <job_id> -w <cores> -np`               |

Note that `-id` is rendered after `-np` and before the subcommand name, the group parsing rule holding remotely too,
and the forging command names no session and requests no rebuild, so it runs the tracked job alone.

The same rendering carries every remote operation but the dataset definition, which goes out as a `python -c` call.
`/remote-execution` carries the tool-to-command mapping. Its `_parse_removals` reads the byte and path lines
`slf clean` printed, so a remote cleanup returns figures parsed straight out of the CLI's own output.

---

## Fallback: what to tell a user when MCP is unavailable

Confirm the server is genuinely unrecoverable through `/forging-mcp-environment-setup` first.

| Blocked MCP tool                      | Tell the user to run                                                         |
|---------------------------------------|------------------------------------------------------------------------------|
| `plan_session_jobs_tool`              | `slf plan session -sp <session>`                                             |
| `plan_dataset_jobs_tool`              | `slf plan dataset -dp <dataset>`                                             |
| `generate_project_plan_tool`          | `slf plan project -pp <project>`                                             |
| The batch pair, any session pipeline  | `slf process -sp <session> <video, microcontroller, runtime, or two-photon>` |
| The batch pair, `pipeline="checksum"` | `slf checksum -sp <session>`                                                 |
| The batch pair, `pipeline="forging"`  | `slf forge -dn <dataset> -pp <project>`                                      |
| `define_forging_dataset_tool`         | `slf forge -dn <dataset> -pp <project> -s <session>`                         |
| `generate_project_manifest_tool`      | `slf manifest -pp <project> create`                                          |
| `read_project_manifest_tool`          | `slf manifest -pp <project> print -s`                                        |
| `generate_dataset_state_tool`         | `slf dataset-state -dp <dataset>`                                            |
| `reset_processing_jobs_tool`          | `slf reset -p <pipeline> -up <unit>`                                         |
| `clean_processing_output_tool`        | `slf clean -p <pipeline> -up <unit>`                                         |
| `write_server_configuration_tool`     | `slf server configure -u <user> -h <host> -r <root> -e <env>`                |
| `read_scheduler_jobs_tool`            | `slf server print -j` or `slf server print -q`                               |
| `discover_remote_project_tool`        | `slf server discover -p <project>`                                           |
| `get_processing_status_tool`, remote  | `slf server batches`, or `slf server batches -b <batch> -a` for each verdict |
| `retire_remote_batches_tool`          | `slf server retire-batch -b <batch>`                                         |

Three caveats. Every substitute runs one unit on this machine, so a batch spanning many sessions becomes one command
per session, and a remote batch can be read and remediated but never dispatched from the CLI. `slf clean` carries none
of the running-batch guard its tool holds, and only `slf plan project`, `slf dataset-state`, and `slf clean` print
machine-readable output, so ask for it. Everything else genuinely blocks until the server is back: dispatching,
cancelling, and polling a local batch, listing and forgetting prepared batches, reading the resource model, inspecting
what outstanding jobs would cost, and reading the project plan, the project jobs table, the dataset state, the dataset
listing, the manifest generation status, and the stored server configuration. Say so plainly rather than improvising.

---

## Related skills

| Skill                                     | Relationship                                                                             |
|-------------------------------------------|------------------------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`          | Owns the `--help` exemption, the `slf mcp` transports, and MCP recovery                  |
| `/batch-processing`                       | Owns the batch pair every pipeline command stands in for, and the reset and clean tools  |
| `/job-planning`                           | Owns the planning tools behind `slf plan` and the resource model with no CLI surface     |
| `/project-state`                          | Owns the manifest and project jobs tools behind `slf manifest`                           |
| `/dataset-definition`, `/dataset-forging` | Own the tools behind `slf forge` and `slf dataset-state`, and its stages                 |
| `/server-configuration`                   | Owns the configuration payload `slf server configure` writes                             |
| `/remote-execution`                       | Owns the ledger, the verdicts, and the remediation the two `server` batch commands drive |
| `assets:working-directory`                | Owns the working directory holding the server configuration and the batch records        |
| `assets:session-discovery`, `/pipeline`   | Upstream session paths for `-sp`, and where each command sits in the pipeline            |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill

Answering a CLI question, reader-judged:
- [ ] Answered from this skill or from slf COMMAND --help, never from memory
- [ ] Quoted the long option form, and never presented -h as a help alias
- [ ] Named the MCP tool the command maps to, or said plainly that none exists, and invoked no slf command but --help

Handing a user a CLI command, reader-judged:
- [ ] Confirmed the MCP server is genuinely unrecoverable through forging-mcp-environment-setup first
- [ ] Printed the command for the user instead of running it
- [ ] Placed every shared option before the subcommand name on slf process and slf manifest, and named one unit per
      invocation, since no pipeline command takes a list
- [ ] Confirmed no batch is executing before recommending slf clean, and named slf omp as the remedy when a macOS
      host reports the OpenMP runtime error
```
