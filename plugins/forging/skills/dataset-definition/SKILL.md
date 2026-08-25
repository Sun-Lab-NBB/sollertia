---
name: dataset-definition
description: >-
  Defines and grows forged dataset hierarchies and reports their forging job state via the sollertia-forgery MCP
  server (composition under the admission policy, the frozen-animal rule, job state snapshots, per-dataset job
  plans). Use when composing a dataset, adding sessions or animals to an existing one, asking what a dataset's
  forging jobs currently record, or sizing those jobs before a submission.
user-invocable: false
---

# Dataset definition

Defines the dataset hierarchy every forging job runs against and reports what those jobs currently record. Uses the
sollertia-forgery MCP server (`slf mcp`).

---

## Scope

**Covers:**
- Composing a dataset on disk and growing it (`define_forging_dataset_tool`)
- The admission policy deciding which sessions may join a dataset, and the frozen-animal rule that governs growth
- The `recreate_animals` versus `force_recreate` distinction and the resolution policy behind both
- Snapshotting and reading forging job state (`generate_dataset_state_tool`, `read_dataset_state_tool`)
- Listing a project's datasets and finding which of them hold a session or an animal (`list_project_datasets_tool`)
- Recording what a dataset's forging jobs will cost (`plan_dataset_jobs_tool`)
- The `host` parameter these five tools carry, and what a `remote` call does differently

**Does not cover:**
- The dataset container itself: the `dataset.yaml` marker, the dataset layout, the column-description companion,
  dataset discovery and structural inspection, and marker repair (see `assets:datasets`)
- Executing forging jobs, monitoring a run, and retrying failures (see `/dataset-forging`)
- Forged output formats and their interpretation (see `/dataset-forging-results`)
- The upstream artifacts a session must carry before it becomes admissible (see `/dataset-forging-input-format`)
- Session discovery and filtering (see `assets:session-discovery`)
- Remote compute server credentials and transfer settings (see `/server-configuration`)
- MCP server connectivity (see `/forging-mcp-environment-setup` for `slf`, `assets:assets-mcp-environment-setup`
  for `slsa`)
- Defining an acquisition system, its session types, or its processing pipelines (see
  `experiment:acquisition-system-design`)

**Handoff rules:** If the `slf` MCP tools are unavailable, invoke `/forging-mcp-environment-setup`. If the session list
is not yet confirmed, invoke `assets:session-discovery` first, which runs on the `slsa` server. Once the hierarchy is
defined, hand off to `/dataset-forging` to run its jobs. Send every question about the dataset marker, the dataset
layout, or the column descriptions to `assets:datasets`.

---

## Agent requirements

You MUST use `define_forging_dataset_tool` to compose or grow a dataset. It is the only tool that builds a dataset
hierarchy on disk. The assets-side `write_dataset_data_tool` rewrites an existing marker and applies none of the
admission policy below, so composing membership by writing a marker produces a dataset nothing verified.

You MUST define a dataset before any of its jobs are planned, prepared, or executed. Every forging job runs against a
hierarchy this tool established, and preparing a dataset it has not built reports an error.

You MUST confirm the session list with the user before defining or extending a dataset. Membership decides what is
forged, and an extension naming new sessions for an animal the dataset already holds is rejected rather than silently
applied.

You MUST run `generate_dataset_state_tool` before reading job state. `read_dataset_state_tool` reads the stored
snapshot rather than the tracker, so a read taken without a fresh snapshot reports the previous run.

---

## Server and response contract

Every tool documented here is served by `slf mcp`, the sollertia-forgery MCP server. The console script is `slf`, the
server subcommand is `mcp`, and its `-t/--transport` option defaults to `stdio`.

Every tool returns a `dict` and never raises. A successful call returns `{"success": true, ...payload}` with the
payload keys at the top level, and a failed call returns `{"success": false, "error": "<message>"}`. Branch on
`response["success"]` before reading any other key. Two tools also return `success: true` while individual datasets
inside them failed: `generate_dataset_state_tool` and `plan_dataset_jobs_tool` report a dataset they could not read in
that dataset's own `units` entry, so you MUST inspect every entry rather than the envelope alone.

### The host parameter

All five tools take `host: str = "local"`, accepting `"local"` for this machine and `"remote"` for the configured
compute server. Any other value returns `Unsupported host '<host>'. Available: local, remote.` In a `remote` call
every path argument is a path ON THE SERVER. A remote read mirrors the project's artifacts onto this machine and reads
the mirror, so it reports what the project currently records and regenerates nothing.

No slsa dataset tool has a `host` parameter. Every tool `assets:datasets` documents is a local-filesystem tool, so a
dataset that lives only on the compute server is reachable through the five tools here and not through that skill.

---

## Ownership boundary

Both servers see the same directory tree and split it cleanly. sollertia-shared-assets owns the container, which is
the marker, the layout, and their self-consistency. sollertia-forgery owns composition, the admission policy, and
forging job state.

| Question                                                                           | Owner                      |
|------------------------------------------------------------------------------------|----------------------------|
| What the `dataset.yaml` marker holds, and how a corrupted one is repaired          | `assets:datasets`          |
| Which datasets exist across a data root, and whether each is structurally complete | `assets:datasets`          |
| Where forged artifacts sit inside the dataset                                      | `assets:datasets`          |
| What the forged columns mean and contain                                           | `/dataset-forging-results` |
| Which sessions may join a dataset, and how one grows                               | this skill                 |
| What a dataset's forging jobs record, and what they will cost                      | this skill                 |
| How those jobs are dispatched, monitored, and retried                              | `/dataset-forging`         |

---

## Available tools

| Tool                          | Purpose                                                                       |
|-------------------------------|-------------------------------------------------------------------------------|
| `define_forging_dataset_tool` | Creates or extends a hierarchy and materializes the per-animal configurations |
| `generate_dataset_state_tool` | Rewrites each named dataset's forging job state snapshot at the dataset root  |
| `read_dataset_state_tool`     | Reads one dataset's job state out of that snapshot, in three widening stages  |
| `list_project_datasets_tool`  | Lists a project's datasets, and which of them hold a given session or animal  |
| `plan_dataset_jobs_tool`      | Records what each named dataset's forging jobs will cost, caching the figures |

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

A `remote` definition returns `dataset_name`, `host`, `dataset_path`, and a `message` instead. The hierarchy sits on
the server, so nothing on this machine can load it to report its shape. Follow with `generate_dataset_state_tool` and
`read_dataset_state_tool` to see what it now holds.

Failures return `Unable to define the local dataset '<name>'. <reason>` or the matching remote form. The resolution
policy raises before anything is written, so a rejected request leaves the dataset exactly as it stood.

**What the call writes beyond the hierarchy.** For each animal it adds or rebuilds, it materializes that animal's
multi-recording configuration (`multi_recording_configuration.yaml`) in the animal's dataset directory. Whether a
configuration is needed at all follows from the dataset's own recorded session type, so a dataset whose sessions the
acquisition system tracks nothing across writes none and reads no source data. An animal already carrying its
configuration is left alone, and one whose sessions have moved off this machine is passed over, which is what lets a
large project be defined in passes. An animal the marker holds but the disk does not is materialized again, so a
definition that failed partway heals itself on the next identical call.

Each covered animal's `surgery_metadata.yaml` is also copied once into its dataset directory, taken from that animal's
most recent covered session. The metadata is optional, and an animal whose latest covered session carries none is
skipped with a warning.

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
directory tree and rebuilt from the sessions the provided list holds for it, while every other animal keeps its data,
and its tracked jobs return to the scheduled state. Each animal is named at most once, must already be in the dataset,
and must have at least one session in the provided list, and each of those three conditions is checked before the
hierarchy is touched.

`force_recreate` is the wider operation. It deletes the entire hierarchy and rebuilds it from the provided list, losing
every animal's tracked job state. Prefer `recreate_animals` whenever the change is confined to named animals, and
confirm either with the user first.

### generate_dataset_state_tool

| Parameter       | Type        | Default    | Description                                                           |
|-----------------|-------------|------------|-----------------------------------------------------------------------|
| `dataset_paths` | `list[str]` | (required) | Dataset root directories to snapshot, which are server paths remotely |
| `host`          | `str`       | `"local"`  | `"local"` or `"remote"`                                               |

Reads each dataset's forging tracker and rewrites `dataset_state.feather` at the dataset root, one row per tracked
job. It is cheap enough to run before deciding what to forge and again once a run finishes.

```text
host:         The host the snapshot was taken on
total_units:  The datasets reported
total_jobs:   The job rows summed across them
units[]:      dataset_path, dataset_name, job_count, and a summary counting jobs by status
              A local unit also carries state_path; a failed local unit carries dataset_path, error, job_count: 0
```

A local snapshot reports a dataset it cannot read in that dataset's own entry and leaves the others alone. A remote
snapshot fails the whole call instead, because one failure anywhere in the server-side sequence aborts it.

### read_dataset_state_tool

| Parameter       | Type                | Default    | Description                                                        |
|-----------------|---------------------|------------|--------------------------------------------------------------------|
| `dataset_path`  | `str`               | (required) | Dataset root directory, whose parent directory names the project   |
| `host`          | `str`               | `"local"`  | `"local"` or `"remote"`                                            |
| `scope`         | `str \| None`       | `None`     | Restricts the listing to `animal` or `session` scoped jobs         |
| `animal`        | `str \| None`       | `None`     | Restricts the listing to one animal's jobs                         |
| `session`       | `str \| None`       | `None`     | Restricts the listing to one session's jobs                        |
| `job_names`     | `list[str] \| None` | `None`     | Restricts the listing to these forging job type names              |
| `status`        | `str \| None`       | `None`     | Restricts the listing to one tracker status, such as `FAILED`      |
| `limit`         | `int \| None`       | `None`     | Jobs to list. Defaults to 200, or 50 with detail. `<= 0` lists all |
| `start_row`     | `int`               | `0`        | The match index to begin at. Follow `next_start_row`               |
| `include_items` | `bool`              | `False`    | Keyword-only. Lists jobs even when no filter is named              |
| `detailed`      | `bool`              | `False`    | Keyword-only. Adds the executor, timestamps, and error text        |

The read widens in three stages. A bare call reports `dataset_path`, `state_path`, a `summary` counting every job by
status, and a `breakdown` naming every scope, animal, job name, and status the snapshot holds, which is how you find
what needs attention without listing anything. Naming any filter, or passing `include_items=True`, adds a `jobs` page
alongside `rows`, `matched_rows`, `start_row`, and `next_start_row`. Each listed job carries `animal`, `session`,
`scope`, `job_name`, `specifier`, `status`, and `job_id`, and `detailed=True` adds `executor_id`, `error_message`,
`started_at`, and `completed_at`. A field carrying nothing is omitted from a job entry rather than reported empty.

The summary and the breakdown span every job regardless of the filters, so narrowing the listing never distorts the
totals. A filter naming a value the snapshot does not hold returns an error listing the available values, which makes
the breakdown the right place to pick filter values from.

Job scope decides what a row's `specifier` names, so resolve the subject from `scope` rather than assuming a session:

| Scope     | What the specifier names | Jobs                                                         |
|-----------|--------------------------|--------------------------------------------------------------|
| `animal`  | The animal identifier    | The cross-recording discovery job, one per tracked animal    |
| `session` | The session name         | The cross-recording extraction and per-session assembly jobs |

Reading the stored table rather than the tracker is what lets a snapshot pulled off a remote host answer without any
access to the data it describes. When no snapshot exists the call returns
`No dataset state snapshot exists at '<path>'. Run generate_dataset_state_tool before reading it.`

### list_project_datasets_tool

| Parameter      | Type          | Default    | Description                                                  |
|----------------|---------------|------------|--------------------------------------------------------------|
| `project_path` | `str`         | (required) | The project's root data directory                            |
| `host`         | `str`         | `"local"`  | `"local"` or `"remote"`                                      |
| `session`      | `str \| None` | `None`     | Narrows the listing to the datasets holding this session     |
| `animal`       | `str \| None` | `None`     | Narrows the listing to the datasets holding this animal      |
| `limit`        | `int \| None` | `None`     | Datasets to list. Defaults to 200, or 50 with detail         |
| `start_row`    | `int`         | `0`        | The match index to begin at. Follow `next_start_row`         |
| `detailed`     | `bool`        | `False`    | Keyword-only. Adds each dataset's animals and its job counts |

Returns `project_path`, `total_datasets`, `total_memberships` summed across datasets, and a `breakdown` per session
type and acquisition system, alongside the `datasets` page and its paging keys. Each entry carries `name`,
`dataset_path`, `session_type`, `acquisition_system`, `session_count`, and `animal_count`. A detailed entry adds
`state_exists`, the `jobs` counts by status when a snapshot exists, and the `animals` the dataset covers.

A project holds a handful of datasets rather than thousands, so the totals and the breakdown appear in every response.
Detail is the expensive half, because it opens one stored state table per listed dataset. A dataset whose snapshot was
never generated reports `state_exists: false` rather than falling back to its tracker, so generate the snapshot first
when the job counts are what you need.

Use this tool to answer "which datasets already hold this session" before composing a new one, which is the cheapest
guard against forging the same sessions twice under two names. It is project-scoped and reports membership and job
counts, so pair it with `assets:datasets` when the question is instead which datasets exist across a whole data root
or whether one is structurally complete. Of the two listings, only this one reaches a remote host.

### plan_dataset_jobs_tool

| Parameter         | Type        | Default    | Description                                                       |
|-------------------|-------------|------------|-------------------------------------------------------------------|
| `dataset_paths`   | `list[str]` | (required) | Dataset root directories to plan, which are server paths remotely |
| `host`            | `str`       | `"local"`  | `"local"` or `"remote"`                                           |
| `regenerate_plan` | `bool`      | `False`    | Keyword-only. Re-estimates the figures the cache already holds    |

Records what every forging job of the named datasets will cost and caches the figures in `job_plan.yaml` at each
dataset root. A dataset's figures follow from the single-day outputs its jobs consume, and admission already requires
each session to carry those outputs, so a dataset is plannable as soon as its hierarchy is defined.

```text
host:             The host the datasets were planned on
total_units:      The datasets reported
total_jobs:       The planned jobs summed across them
elapsed_seconds:  How long planning took
units[]:          unit_path, unit_name, job_count, and summed_memory_mb, or unit_path and the error that stopped it
```

Naming no dataset returns `No processing unit was named.` Every named dataset must belong to the same project, since
the plan and state artifacts a batch is resolved from are written per project. Leave `regenerate_plan` at `False`
unless a deliberate retune should be adopted, because a submission may already have been sized against the recorded
figures.

---

## Admission policy

A session joins a dataset only once every pipeline its acquisition system requires for its session type reports every
tracked job as succeeded. The requirement set is resolved per acquisition system and per session type, so it is a
property of the system that recorded the session rather than of the dataset. A pipeline whose tracker is missing or
holds no jobs counts as not run, because a pipeline that resolved jobs records them before dispatching any.

The gate runs on every session, at creation and at extension alike, and before any part of the hierarchy is built:

- A session whose type joins no dataset for its acquisition system is rejected, and the error lists the session types
  that system does admit.
- A session with an outstanding pipeline is rejected, and the error names each outstanding pipeline alongside its
  state, which reads `<n> of <m> job(s) not succeeded` for a partially finished one.
- Every session must share the first session's session type and acquisition system, and an extension must match the
  ones the dataset already records. Split a mixed batch by session type or by acquisition system.

The dataset's column descriptions are resolved from its acquisition system at creation time and baked into the
dataset, which is why the acquisition system is fixed for the dataset's life. See `assets:datasets` for the
description companion itself, and `/dataset-forging-input-format` for what each required pipeline produces.

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

1. **Confirm inputs.** Take the project root and the session names from the user. Session names come from
   `assets:session-discovery`. Do not infer paths from within this skill.

2. **Check for an existing dataset.** Call `list_project_datasets_tool(project_path=...)`, optionally with `session`
   or `animal`, to see whether a dataset already covers the sessions in question. Prefer extending one over creating a
   near-duplicate.

3. **Decide create, extend, or rebuild.** Read the resolution policy table above with the user. A pure addition of new
   animals needs neither flag. A change to an animal already in the dataset needs `recreate_animals`. A change to the
   dataset's whole definition needs `force_recreate`, which discards every animal's job state.

4. **Define the dataset.** Call `define_forging_dataset_tool` with the confirmed arguments. Inspect the response:
   - `success: true` with `session_count` and `animal_count` matching the intent means the hierarchy is ready.
   - An error naming an outstanding pipeline means the session is not yet admissible. See the error routing below.
   - An error naming frozen animals means the request would widen an existing animal. Return to step 3.

5. **Snapshot the job state.** Call `generate_dataset_state_tool(dataset_paths=[dataset_path])` to write the
   dataset's state artifact, then `read_dataset_state_tool(dataset_path=...)` to confirm the job universe matches the
   session and animal counts the definition reported.

6. **Plan the jobs.** Call `plan_dataset_jobs_tool(dataset_paths=[dataset_path])` when the run needs sizing, which
   any remote submission does.

7. **Hand off to execution.** Invoke `/dataset-forging` to prepare and dispatch the jobs. Return here only when the
   session set changes or the job state has to be read again.

---

## Membership lookup

Candidate sessions are found on the **slsa** server, not this one. `get_data_root_overview_tool` enumerates the data
root and `filter_sessions_tool` narrows it, both documented by `assets:session-discovery`, and both require the slsa
server to be connected (see `assets:assets-mcp-environment-setup`, not the forging setup skill).

`filter_sessions_tool` handles date range narrowing and animal inclusion and exclusion server-side. Project and
session-type narrowing remain caller-side, so apply those to the returned list yourself before presenting a membership
proposal. Confirm the final list with the user, then bring it here as `session_names`.

---

## Error routing

| Error pattern                                                                   | Resolution                                            |
|---------------------------------------------------------------------------------|-------------------------------------------------------|
| `Unsupported host '<host>'. Available: local, remote.`                          | Pass `local` or `remote`                              |
| `Unable to resolve the directory for session '<s>' under '<root>'.`             | Name is absent or ambiguous, recheck the session list |
| `... joins no dataset for the '<system>' acquisition system ...`                | Remove the session, its type is not forgeable there   |
| `... the following are outstanding: {...}`                                      | Process the session through the named pipelines       |
| `All sessions in a dataset must share the same session type ...`                | Split the batch by session type                       |
| `All sessions in a dataset must be acquired by the same acquisition system ...` | Split the batch by acquisition system                 |
| `Sessions absent from the dataset were provided for the animal(s) ...`          | Name those animals in `recreate_animals`              |
| `... force_recreate and recreate_animals ... are mutually exclusive ...`        | Choose one, prefer `recreate_animals`                 |
| `Rebuilding an animal requires an existing dataset ...`                         | Drop `recreate_animals` for a first definition        |
| `Every animal named for rebuilding must already be part of the dataset ...`     | Correct the identifier against the dataset's animals  |
| `No dataset state snapshot exists at '<path>'.`                                 | Run `generate_dataset_state_tool` first               |
| `No <subject> has '<column>' in [...]. Available: [...]`                        | Pick a filter value from the response `breakdown`     |
| `No processing unit was named.`                                                 | Pass at least one dataset path                        |
| `Unable to resolve a batch spanning the projects [...]`                         | Split the call, one project per call                  |
| MCP tools unavailable                                                           | Invoke `/forging-mcp-environment-setup`               |

---

## Related skills

| Skill                                  | Relationship                                                                         |
|----------------------------------------|--------------------------------------------------------------------------------------|
| `assets:datasets`                      | Owns the dataset container: the marker, the layout, discovery, and inspection        |
| `assets:session-discovery`             | Upstream: supplies the candidate session list, on the slsa server                    |
| `assets:assets-mcp-environment-setup`  | Prerequisite for the slsa half of membership lookup                                  |
| `/forging-mcp-environment-setup`       | Prerequisite: slf MCP server connectivity                                            |
| `/dataset-forging`                     | Downstream: prepares, dispatches, and monitors the jobs defined here                 |
| `/dataset-forging-input-format`        | Reference: what each admission-gating pipeline must have produced                    |
| `/dataset-forging-results`             | Downstream reference: the forged outputs and what their columns contain              |
| `/project-manifest`                    | Sibling: project-wide processing status, which shows whether a session is admissible |
| `/server-configuration`                | Prerequisite for any `host="remote"` call                                            |
| `experiment:acquisition-system-design` | Reference: how a system declares its session types and processing pipelines          |

---

## Verification checklist

```text
Dataset definition:
- [ ] Verified slf MCP connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Session list confirmed with the user, sourced from assets:session-discovery
- [ ] list_project_datasets_tool checked for an existing dataset covering these sessions
- [ ] Create, extend, or rebuild decided against the resolution policy table
- [ ] recreate_animals versus force_recreate confirmed with the user when either was used
- [ ] define_forging_dataset_tool returned success with the expected session and animal counts
- [ ] generate_dataset_state_tool run after the definition and after every forging run
- [ ] read_dataset_state_tool confirmed the job universe against those counts
- [ ] plan_dataset_jobs_tool run when the jobs had to be sized
- [ ] Container questions routed to assets:datasets, execution routed to /dataset-forging
```
