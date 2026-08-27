---
name: dataset-definition
description: >-
  Composes and grows forged dataset hierarchies and reports their forging job state through the sollertia-forgery MCP
  server. Covers the admission policy, the frozen-animal rule, the artifacts a definition materializes, the dataset
  state schema, and the three forging job names with their scopes. Use when composing a dataset, adding sessions or
  animals to an existing one, asking what a dataset's forging jobs currently record, or listing which datasets already
  hold a session.
user-invocable: false
---

# Dataset definition

Defines the dataset hierarchy every forging job runs against and reports what those jobs currently record, through the
sollertia-forgery MCP server (`slf mcp`). This skill is the **exclusive** owner of `define_forging_dataset_tool`,
`generate_dataset_state_tool`, `read_dataset_state_tool`, and `list_project_datasets_tool`. No other skill in the
marketplace may document or call these four tools. `plan_dataset_jobs_tool` belongs to `/job-planning`, and only what
it reports differently for a dataset unit is recorded here.

---

## Scope

**Covers:**
- Composing a dataset on disk and growing it (`define_forging_dataset_tool`)
- The admission policy deciding which sessions may join a dataset, and the frozen-animal rule that governs growth
- The `recreate_animals` versus `force_recreate` distinction and the resolution policy behind both
- The artifacts a definition materializes, including the per-animal multi-recording configuration
- The three forging job names, their scopes, and the `dataset_state.feather` schema a state read serves from
- Snapshotting and reading forging job state (`generate_dataset_state_tool`, `read_dataset_state_tool`)
- Listing a project's datasets and finding which of them hold a session or an animal (`list_project_datasets_tool`)
- The `host` parameter these four tools carry, and what a `remote` call does differently

**Does not cover:**
- The dataset marker's own fields, the dataset layout, discovery across a whole data root, and marker repair. Owned by
  `assets:datasets`.
- Preparing, dispatching, monitoring, and retrying the forging jobs defined here. Owned by `/dataset-forging`.
- The batch mechanics every pipeline shares, including preparation, execution, and cleanup. Owned by
  `/batch-processing`.
- Job planning across pipelines and units, the resource model, and the project plan projection. Owned by
  `/job-planning`.
- Project-wide processing status and the manifest that reports it. Owned by `/project-state`.
- Forged output formats and the meaning of their columns. Owned by `/processing-results`.
- The upstream artifacts a session must carry before it becomes admissible. Owned by `/processing-input-format`.
- Session discovery and filtering. Owned by `assets:session-discovery`.
- Remote compute server credentials and transfer settings. Owned by `/server-configuration`.
- MCP server connectivity. Owned by `/forging-mcp-environment-setup` for `slf`, and by
  `assets:assets-mcp-environment-setup` for `slsa`.
- Declaring an acquisition system and its session types. Owned by `assets:library-extension`.
- Declaring a processing pipeline. Owned by `/library-extension`.

**Handoff rules:** If the `slf` MCP tools are unavailable, invoke `/forging-mcp-environment-setup`. If the session list
is not yet confirmed, invoke `assets:session-discovery` first, which runs on the `slsa` server. Once the hierarchy is
defined, hand off to `/dataset-forging` to run its jobs. Send every question about the dataset marker, the dataset
layout, or the column-description companion to `assets:datasets`.

---

## Agent requirements

You MUST use `define_forging_dataset_tool` to compose or grow a dataset. It is the only tool that builds a dataset
hierarchy on disk. Composing membership by rewriting a dataset marker through the assets server applies none of the
admission policy below, so it produces a dataset nothing verified.

You MUST define a dataset before any of its jobs are planned, prepared, or executed. Every forging job runs against a
hierarchy this tool established, and preparing a dataset it has not built reports an error.

You MUST confirm the session list with the user before defining or extending a dataset. `assets:session-discovery` is
the exclusive producer of the session lists this skill consumes. Membership decides what is forged, and an extension
naming new sessions for an animal the dataset already holds is rejected rather than silently applied.

You MUST run `generate_dataset_state_tool` before reading job state. `read_dataset_state_tool` reads the stored
snapshot rather than the tracker, so a read taken without a fresh snapshot reports the previous run.

---

## Response and host contract

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

All four tools take `host: str = "local"`, accepting `"local"` for this machine and `"remote"` for the configured
compute server. Any other value returns `Unsupported host '<host>'. Available: local, remote.` In a `remote` call every
path argument is a path on the server. A remote read mirrors the project's artifacts onto this machine and reads the
mirror, so it reports what the project currently records and regenerates nothing.

No slsa dataset tool carries a `host` parameter. Every tool `assets:datasets` documents is a local-filesystem tool, so
a dataset that lives only on the compute server is reachable through the four tools here and not through that skill.

---

## Ownership boundary

Both servers see the same directory tree and split it cleanly. sollertia-shared-assets owns the container, which is the
marker, the layout, and their self-consistency. sollertia-forgery owns composition, admission, and forging job state.

| Question                                                                           | Owner             |
|------------------------------------------------------------------------------------|-------------------|
| What the `dataset.yaml` marker holds, and how a corrupted one is repaired          | `assets:datasets` |
| Which datasets exist across a data root, and whether each is structurally complete | `assets:datasets` |
| Where forged artifacts sit inside the dataset                                      | `assets:datasets` |
| Which sessions may join a dataset, and how one grows                               | this skill        |
| What a dataset's forging jobs record                                               | this skill        |

---

## The dataset artifacts

A dataset root is a sibling of the project's animal directories, and every artifact below sits at that root apart from
the per-animal configuration.

| Artifact                                      | Written by                                    | Contents                                                                                               |
|-----------------------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| `dataset.yaml`                                | `DatasetData.create`, last                    | The marker naming the dataset, its project, its session type, its acquisition system, and its sessions |
| `data_descriptions.feather`                   | `DatasetData._write_column_descriptions`      | Two `String` columns, `column` and `description`, written once at definition time                      |
| `forging_tracker.yaml`                        | the forging pipeline                          | Every forging job of the dataset, and what the last run recorded for each                              |
| `dataset_state.feather`                       | `forging.state.generate_dataset_state`        | One row per tracked forging job, the table `read_dataset_state_tool` serves from                       |
| `job_plan.yaml`                               | `orchestration.planning`                      | The dataset's plan cache, one entry per planned job                                                    |
| `<animal>/multi_recording_configuration.yaml` | `forging.pipeline._materialize_multiday_plan` | That animal's recording set, its qualified dataset name, and the progress flag                         |

The description companion is written before the marker, so a discoverable dataset always carries its descriptions. It
holds the column descriptions the acquisition system donates through `registries.resolve_forging_column_descriptions`,
baked in at creation time, which is why the acquisition system is fixed for the dataset's life. An empty donation is
valid and writes an empty companion. The Mesoscope-VR donations that fill this seam are documented by
`mesoscope:mesoscope-vr-processing-schema`.

A per-animal configuration is written only for an animal whose acquisition system resolves a multi-recording
configuration for the dataset's recorded session type, through `registries.resolve_multi_recording_session_types` and
`registries.resolve_multi_recording_configuration_resolver`. A dataset whose session type resolves none writes no
configuration and reads no source data. An animal is tracked across recordings exactly when that file is on disk.

### The dataset state schema

`forging.state._DATASET_STATE_SCHEMA` declares twelve columns, one row per tracked forging job:

| Column          | Polars type | Meaning                                                                                         |
|-----------------|-------------|-------------------------------------------------------------------------------------------------|
| `dataset`       | `String`    | The dataset that owns the job, constant across every row of one snapshot                        |
| `animal`        | `String`    | The specifier itself for an animal-scoped row, or the session's animal for a session-scoped one |
| `session`       | `String`    | The specifier for a session-scoped row, and null for an animal-scoped one                       |
| `scope`         | `String`    | `animal` or `session`                                                                           |
| `job_id`        | `String`    | The tracker registry key, which joins directly against the project plan artifact                |
| `job_name`      | `String`    | One of the three forging job names                                                              |
| `specifier`     | `String`    | The raw tracker specifier, an animal identifier or a session name                               |
| `status`        | `String`    | `SCHEDULED`, `RUNNING`, `SUCCEEDED`, or `FAILED`                                                |
| `executor_id`   | `String`    | The executor that ran the job, and null when none was recorded                                  |
| `error_message` | `String`    | The failure text, and null for a job that recorded no failure                                   |
| `started_at`    | `UInt64`    | Microsecond epoch at which the job started, and null when it never started                      |
| `completed_at`  | `UInt64`    | Microsecond epoch at which the job finished, and null otherwise                                 |

Rows are naturally sorted by `animal`, `session`, and `job_name` with nulls last, so animal 2 precedes animal 10 and an
animal-scoped row sorts last within its animal. `animal` is a `String` in every project artifact, so this table joins
against the project manifest, jobs, and plan tables on animal and session without casting.

---

## The forging jobs

`forging.state._DATASET_JOB_SCOPES` maps each forging job name to the unit its specifier names, `multiday_discovery` to
`animal`, and `multiday_extraction` and `session_data_assembly` to `session`. `/dataset-forging` owns what each job type
does, when it exists, and the order the three run in.

These three are the only values `read_dataset_state_tool` accepts in `job_names`, and `animal` and `session` are the
only values it accepts in `scope`. Resolve a row's subject from `scope` rather than assuming the specifier names a
session. A dataset whose animals are tracked across no recordings carries assembly jobs alone, and an animal-scoped row
carries no `session`, which is omitted from its listing entry rather than reported empty. A session-scoped row resolves
its animal through the dataset's own session list, so a dataset that dropped a session during a rebuild reports that
session's outstanding rows with no animal until the next pipeline run realigns the tracker.

---

## Admission policy

A session joins a dataset only once its acquisition system reports a success for every pipeline the session's type
requires. The requirement set is resolved through `registries.resolve_forging_admission_pipelines`, so it is a property
of the system that recorded the session rather than of the dataset. Every pipeline resolves its own job universe from
the manifests written at acquisition, so a tracker reporting every job as succeeded already accounts for every source
the session recorded. A pipeline whose tracker is missing or holds no jobs counts as not run, because a pipeline that
resolved jobs records them before dispatching any.

The gate runs on every session, at creation and at extension alike, and before any part of the hierarchy is built:

- A session whose type joins no dataset for its acquisition system is rejected, and the error lists the session types
  that system does admit.
- A session with an outstanding pipeline is rejected, and the error names every outstanding pipeline alongside its
  state. A pipeline that recorded nothing reads `not_started`, and a partly finished one reads the partial-state shape
  `<unfinished> of <total> job(s) not succeeded`, which counts scheduled, running, and failed jobs alike.
- Every session must share the first session's session type and acquisition system, and an extension must match the
  ones the dataset already records. Split a mixed batch by session type or by acquisition system.

`/processing-input-format` documents what each admission-gating pipeline consumes, and `/project-state` reports which
sessions have cleared their pipelines across a whole project.

---

## MCP tool surface

| Tool                          | Purpose                                                                       |
|-------------------------------|-------------------------------------------------------------------------------|
| `define_forging_dataset_tool` | Creates or extends a hierarchy and materializes the per-animal configurations |
| `generate_dataset_state_tool` | Rewrites each named dataset's forging job state snapshot at the dataset root  |
| `read_dataset_state_tool`     | Reads one dataset's job state out of that snapshot, in three widening stages  |
| `list_project_datasets_tool`  | Lists a project's datasets, and which of them hold a given session or animal  |

### define_forging_dataset_tool

| Parameter          | Type                | Default    | Description                                                      |
|--------------------|---------------------|------------|------------------------------------------------------------------|
| `project_path`     | `str`               | (required) | Project root holding the animal and session directories          |
| `dataset_name`     | `str`               | (required) | Unique dataset name, which becomes the directory under the root  |
| `session_names`    | `list[str]`         | (required) | The session names the dataset must contain                       |
| `recreate_animals` | `list[str] \| None` | `None`     | Animals already in the dataset to rebuild from the provided list |
| `host`             | `str`               | `"local"`  | `"local"` or `"remote"`                                          |
| `force_recreate`   | `bool`              | `False`    | Keyword-only. Deletes the whole hierarchy and rebuilds it        |

A local definition returns:

```text
dataset_name:   The resolved dataset name
dataset_path:   Absolute path to {project_path}/{dataset_name}/
tracker_path:   Absolute path to the dataset's forging_tracker.yaml, beside its marker
session_count:  The sessions the dataset now holds
animal_count:   The animals the dataset now covers
animals:        The animal identifiers the dataset covers
```

A `remote` definition returns `dataset_name`, `host`, `dataset_path`, and a `message` instead. The hierarchy sits on the
server, so nothing on this machine can load it to report its shape. Follow with `generate_dataset_state_tool` and
`read_dataset_state_tool` to see what it now holds.

Failures return `Unable to define the local dataset '<name>'. <reason>` or the matching remote form. The resolution
policy raises before anything is written, so a rejected request leaves the dataset exactly as it stood.

**What the call writes beyond the hierarchy.** For each animal it covers, it materializes that animal's multi-recording
configuration when the dataset's session type resolves one. The covered set is the animals this call added, plus the
animals named in `recreate_animals`, plus every animal that still has a source directory under the project root and
holds no configuration on disk. An animal already carrying its configuration is left alone. An animal whose sessions
have moved off this machine is passed over, so a dataset keeps growing while part of its source data lives elsewhere,
and a large project is forged in passes. Membership in the marker is not evidence that a configuration was written,
because the marker commits first, so a definition that failed partway heals itself on the next identical call.

The `surgery_metadata.yaml` of each animal whose sessions this call added or rebuilt is also copied once into its
dataset directory, taken from that animal's most recent session among them. The metadata is optional, and an animal
whose latest such session carries none is skipped with a warning.

#### Resolution policy

| Hierarchy | `session_names` | `recreate_animals` | `force_recreate` | Outcome                                          |
|-----------|-----------------|--------------------|------------------|--------------------------------------------------|
| Absent    | non-empty       | omitted            | `False`          | Creates the dataset from the provided sessions   |
| Absent    | empty           | omitted            | any              | Error: nothing to create the dataset from        |
| Absent    | any             | named              | `False`          | Error: rebuilding requires an existing dataset   |
| Present   | empty           | omitted            | `False`          | Loads the dataset unchanged                      |
| Present   | empty           | named              | `False`          | Error: the session list must define the new set  |
| Present   | non-empty       | omitted            | `False`          | Appends the sessions of animals it does not hold |
| Present   | non-empty       | named              | `False`          | Rebuilds the named animals, keeps the others     |
| Present   | non-empty       | omitted            | `True`           | Deletes the hierarchy and recreates it           |
| Present   | empty           | omitted            | `True`           | Error: recreation requires a session list        |
| any       | any             | named              | `True`           | Error: the two arguments are mutually exclusive  |

`session_names` names the sessions the dataset must contain, not the sessions to append one by one. A session the
dataset already holds is left alone, and a name repeated in the list resolves once. A name matching no session
directory under the project root, or matching more than one, is an error naming that session.

#### The frozen-animal rule

An animal already in the dataset is frozen. Providing a session it does not hold is rejected, because widening an
animal's session set invalidates the outputs already forged for the sessions it keeps. Adding sessions for an animal
the dataset does not hold stays safe and needs nothing extra.

Name the animal in `recreate_animals` to opt it out of the freeze. The animal is dropped from the dataset with its
directory tree and rebuilt from the sessions the provided list holds for it, and its tracked jobs return to the
scheduled state. Every other animal keeps its data. Each animal is named at most once, must already be in the dataset,
and must have at least one session in the provided list, all three checked before the hierarchy is touched.

`force_recreate` is the destructive alternative. It deletes the entire hierarchy and rebuilds it from the provided
list, losing every animal's tracked job state, and it cannot be combined with `recreate_animals`. Prefer
`recreate_animals` whenever the change is confined to named animals, and confirm either with the user first.

### generate_dataset_state_tool

| Parameter       | Type        | Default    | Description                                                           |
|-----------------|-------------|------------|-----------------------------------------------------------------------|
| `dataset_paths` | `list[str]` | (required) | Dataset root directories to snapshot, which are server paths remotely |
| `host`          | `str`       | `"local"`  | `"local"` or `"remote"`                                               |

Reads each dataset's forging tracker and rewrites `dataset_state.feather` at the dataset root, one row per tracked job.
It is cheap enough to run before deciding what to forge and again once a run finishes.

```text
host:         The host the snapshot was taken on
total_units:  The datasets reported, counting failed entries
total_jobs:   The job rows summed across them
units[]:      dataset_path, dataset_name, job_count, and a summary counting jobs by status
              A local unit also carries state_path, and its summary opens with a total key
              A failed local unit carries dataset_path, error, and job_count: 0
```

A local snapshot reports an unreadable dataset in that dataset's own entry, leaves the others alone, and still
returns `success: true`, so inspect every entry rather than the envelope alone. A remote snapshot fails the whole call
instead, because one failure anywhere in the server-side sequence aborts it. A dataset with no tracker file, or a
tracker holding no jobs, produces an empty snapshot rather than an error.

### read_dataset_state_tool

| Parameter       | Type                | Default    | Description                                                      |
|-----------------|---------------------|------------|------------------------------------------------------------------|
| `dataset_path`  | `str`               | (required) | Dataset root directory, whose parent directory names the project |
| `host`          | `str`               | `"local"`  | `"local"` or `"remote"`                                          |
| `scope`         | `str \| None`       | `None`     | Restricts the listing to `animal` or `session` scoped jobs       |
| `animal`        | `str \| None`       | `None`     | Restricts the listing to one animal's jobs                       |
| `session`       | `str \| None`       | `None`     | Restricts the listing to one session's jobs                      |
| `job_names`     | `list[str] \| None` | `None`     | Restricts the listing to these forging job names                 |
| `status`        | `str \| None`       | `None`     | Restricts the listing to one status, such as `FAILED`            |
| `limit`         | `int \| None`       | `None`     | Jobs to list. See the response contract                          |
| `start_row`     | `int`               | `0`        | The match index to begin at. Follow `next_start_row`             |
| `include_items` | `bool`              | `False`    | Keyword-only. Lists jobs even when no filter is named            |
| `detailed`      | `bool`              | `False`    | Keyword-only. Adds the executor, timestamps, and error text      |

A bare call reports `dataset_path`, `state_path`, a `summary` counting every job by status, and a `breakdown` counting
`scope`, `animal`, `job_name`, and `status`, which is how you find what needs attention without listing anything.
Naming any filter, or passing `include_items=True`, adds a `jobs` page alongside `rows`, `matched_rows`, `start_row`,
and `next_start_row`. Each listed job carries `animal`, `session`, `scope`, `job_name`, `specifier`, `status`, and
`job_id`, and `detailed=True` adds `executor_id`, `error_message`, `started_at`, and `completed_at`.

The summary and the breakdown are computed from the whole snapshot before any filter applies, so narrowing the listing
never distorts the totals. Filters combine conjunctively, and each is validated against the whole snapshot, so a filter
naming a value the snapshot does not hold returns an error listing the available values. That makes the breakdown the
right place to pick filter values from.

Reading the stored table rather than the tracker is what lets a snapshot pulled off a remote host answer without any
access to the data it describes. When no snapshot exists the call returns `No dataset state snapshot exists at
'<path>'. Run generate_dataset_state_tool before reading it.`

### list_project_datasets_tool

| Parameter      | Type          | Default    | Description                                                  |
|----------------|---------------|------------|--------------------------------------------------------------|
| `project_path` | `str`         | (required) | The project's root data directory                            |
| `host`         | `str`         | `"local"`  | `"local"` or `"remote"`                                      |
| `session`      | `str \| None` | `None`     | Narrows the listing to the datasets holding this session     |
| `animal`       | `str \| None` | `None`     | Narrows the listing to the datasets holding this animal      |
| `limit`        | `int \| None` | `None`     | Datasets to list. See the response contract                  |
| `start_row`    | `int`         | `0`        | The match index to begin at. Follow `next_start_row`         |
| `detailed`     | `bool`        | `False`    | Keyword-only. Adds each dataset's animals and its job counts |

Returns `project_path`, `total_datasets`, `total_memberships` summed across datasets, and a `breakdown` per session
type and acquisition system, alongside the `datasets` page and its paging keys. Each entry carries `name`,
`dataset_path`, `session_type`, `acquisition_system`, `session_count`, and `animal_count`. A detailed entry adds
`state_exists`, the `jobs` counts by status when a snapshot exists, and the `animals` the dataset covers.

A project holds a handful of datasets rather than thousands, so this tool carries no `include_items` and its listing
appears in every response. Detail is the expensive half, because it opens one stored state table per listed dataset. A
dataset whose snapshot was never generated reports `state_exists: false` rather than falling back to its tracker, so
generate the snapshot first when the job counts are what you need. A `session` or `animal` value nothing matches yields
an empty page with `matched_rows: 0` rather than an error.

Use this tool to answer "which datasets already hold this session" before composing a new one, which is the cheapest
guard against forging the same sessions twice under two names. It is project-scoped and reports membership and job
counts, so pair it with `assets:datasets` when the question is instead which datasets exist across a whole data root or
whether one is structurally complete. Of the two listings, only this one reaches a remote host.

---

## Sizing a dataset's jobs

`plan_dataset_jobs_tool(dataset_paths=[...], host=..., regenerate_plan=...)` records what a dataset's forging jobs will
cost and caches the figures in `job_plan.yaml` at each dataset root. `/job-planning` owns the tool, the resource model,
and the project plan projection every plan call rewrites. Three facts are specific to a dataset unit. A dataset is
plannable as soon as its hierarchy is defined, because its figures follow from the single-day outputs its jobs consume
and admission already requires each session to carry those outputs. Every named dataset must belong to the same
project, since the plan and state artifacts a batch is resolved from are written per project. A local unit entry
carries `unsized_jobs` whenever the plan recorded a sizing refusal, mapping each refused job to the reason its sizing
pass gave, and a refused job is left out of the plan rather than failing it.

`unsized_jobs` is local-only. The remote summariser reads its figures back out of the project plan projection and
cannot see the plan cache's refusal map. A remote unit therefore reports `unit_path`, `unit_name`, `job_count`, and
`summed_memory_mb` alone, or an error saying the projection holds no job for the unit.

---

## Composition workflow

### Pre-composition checklist

```text
- [ ] slf MCP tools available (else /forging-mcp-environment-setup)
- [ ] Project root and the exact session list confirmed with the user
- [ ] Every named session has completed the pipelines its acquisition system requires
- [ ] list_project_datasets_tool checked for an existing dataset covering these sessions
- [ ] For a growth request, the affected animals identified and the freeze decision made with the user
```

**STOP**: If any checkbox is incomplete, do not proceed. Composing a dataset writes a hierarchy and materializes
per-animal configurations, and undoing that means deleting and rebuilding it.

### Workflow steps

1. **Confirm inputs.** Take the project root and the session names from the user. Candidate sessions are found on the
   **slsa** server through `assets:session-discovery`, which needs `assets:assets-mcp-environment-setup` rather than
   the forging setup skill. Do not infer paths from within this skill.

2. **Check for an existing dataset.** Call `list_project_datasets_tool(project_path=...)`, optionally with `session` or
   `animal`, to see whether a dataset already covers the sessions in question. Prefer extending one over creating a
   near-duplicate.

3. **Decide create, extend, or rebuild.** Read the resolution policy table above with the user. A pure addition of new
   animals needs neither flag. A change to an animal already in the dataset needs `recreate_animals`. A change to the
   dataset's whole definition needs `force_recreate`, which discards every animal's job state.

4. **Define the dataset.** Call `define_forging_dataset_tool` with the confirmed arguments. Inspect the response:
   - `success: true` with `session_count` and `animal_count` matching the intent means the hierarchy is ready.
   - An error naming an outstanding pipeline means the session is not yet admissible. See the error routing below.
   - An error naming frozen animals means the request would widen an existing animal. Return to step 3.

5. **Snapshot the job state.** Call `generate_dataset_state_tool(dataset_paths=[dataset_path])` to write the dataset's
   state artifact, then `read_dataset_state_tool(dataset_path=...)` to confirm the job universe matches the session and
   animal counts the definition reported, at the scopes `_DATASET_JOB_SCOPES` records.

6. **Size the jobs.** Call `plan_dataset_jobs_tool(dataset_paths=[dataset_path])` when the run needs sizing, which any
   remote submission does, and read `/job-planning` for what the figures mean.

7. **Hand off to execution.** Invoke `/dataset-forging` to prepare and dispatch the jobs. Return here only when the
   session set changes or the job state has to be read again.

---

## Error routing

| Error pattern                                                                   | Resolution                                                                      |
|---------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| `Unsupported host '<host>'. Available: local, remote.`                          | Pass `local` or `remote`                                                        |
| `Unable to resolve the directory for session '<s>' under '<root>'.`             | Name is absent or ambiguous, recheck the session list                           |
| `... joins no dataset for the '<system>' acquisition system ...`                | Remove the session, its type is not forgeable there                             |
| `... the following are outstanding: {...}`                                      | Process the session through the named pipelines                                 |
| `All sessions in a dataset must share the same session type ...`                | Split the batch by session type                                                 |
| `All sessions in a dataset must be acquired by the same acquisition system ...` | Split the batch by acquisition system                                           |
| `Sessions absent from the dataset were provided for the animal(s) ...`          | Name those animals in `recreate_animals`                                        |
| `... force_recreate and recreate_animals ... are mutually exclusive ...`        | Choose one, prefer `recreate_animals`                                           |
| `Rebuilding an animal requires an existing dataset ...`                         | Drop `recreate_animals` for a first definition                                  |
| `Every animal named for rebuilding must already be part of the dataset ...`     | Correct the identifier against the dataset's animals                            |
| `Each animal is named for rebuilding at most once ...`                          | Remove the repeated identifier                                                  |
| `Rebuilding an animal replaces its session set ...`                             | Provide at least one session for each named animal                              |
| `Unable to load the <host> dataset at '<path>'.`                                | Recheck the dataset root, and the host it sits on                               |
| `No dataset state snapshot exists at '<path>'.`                                 | Run `generate_dataset_state_tool` first                                         |
| `No job has '<column>' in [...]. Available: [...]`                              | Pick a filter value from the response `breakdown`                               |
| `... records job name(s) [...], which declare no scope.`                        | A job name reached the tracker with no scope entry, invoke `/library-extension` |
| MCP tools unavailable                                                           | Invoke `/forging-mcp-environment-setup`                                         |

---

## Related skills

| Skill                                      | Relationship                                                                       |
|--------------------------------------------|------------------------------------------------------------------------------------|
| `assets:datasets`                          | Owns the dataset container, the marker, the layout, discovery, and inspection      |
| `assets:session-discovery`                 | Upstream: the exclusive producer of the candidate session list, on the slsa server |
| `assets:assets-mcp-environment-setup`      | Prerequisite: the slsa server that supplies the candidate sessions                 |
| `/forging-mcp-environment-setup`           | Prerequisite: slf MCP server connectivity, and owner of the response contract      |
| `/dataset-forging`                         | Downstream: dispatches and monitors the jobs defined here                          |
| `/batch-processing`                        | Owns the batch mechanics every forging run goes through                            |
| `/job-planning`                            | Owns `plan_dataset_jobs_tool`, the resource model, and the project plan projection |
| `/project-state`                           | Sibling: reports project-wide processing status and which sessions are admissible  |
| `/processing-input-format`                 | Reference: what each admission-gating pipeline consumes                            |
| `/processing-results`                      | Downstream reference: the forged outputs and what their columns contain            |
| `/server-configuration`                    | Prerequisite: every `host="remote"` call resolves through it                       |
| `/library-extension`                       | Recipe for a new pipeline, job name, admission mapping, or description donation    |
| `assets:library-extension`                 | Reference: declares the acquisition system and the session types admitted here     |
| `mesoscope:mesoscope-vr-processing-schema` | Owns the Mesoscope-VR donations that fill the column-description seam              |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Dataset definition:
- [ ] sollertia-forgery MCP server is connected (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Session list confirmed with the user, sourced from assets:session-discovery
- [ ] list_project_datasets_tool checked for an existing dataset covering these sessions
- [ ] Create, extend, or rebuild decided against the resolution policy table
- [ ] recreate_animals versus force_recreate confirmed with the user when either was used
- [ ] define_forging_dataset_tool returned success with the expected session and animal counts
- [ ] generate_dataset_state_tool run after the definition and after every forging run
- [ ] read_dataset_state_tool confirmed the job universe, with each row's subject read from its scope
- [ ] scope and job_names filter values taken from the three declared forging job names
- [ ] Container questions routed to assets:datasets, execution routed to /dataset-forging
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
