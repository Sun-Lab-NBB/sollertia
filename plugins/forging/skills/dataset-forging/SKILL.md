---
name: dataset-forging
description: >-
  Documents what the forging batch pipeline does differently from the five per-session pipelines. Covers the dataset
  processing unit, the three job types its tracker records, the cross-recording stages it dispatches in process, and
  the assembly concurrency ceiling. Use when forging a dataset, when preparing or executing a forging batch, or when a
  forging job reports blocked or fails to assemble.
user-invocable: false
---

# Dataset forging

Runs the `forging` batch pipeline, whose processing unit is a dataset rather than a session and whose tracker records
three job types rather than one. This skill owns no MCP tools of its own. Preparation, execution, monitoring,
cancellation, reset, and cleaning are owned by `/batch-processing`, which drives `forging` through the same generic
tools the other five pipelines use, so this skill states only what `forging` does differently.

---

## Scope

**Covers:**
- The dataset processing unit and the tool parameters that carry it
- The three forging job types, their job-name constants, their scopes, and their specifiers
- How the job universe is built from the animals and sessions the dataset hierarchy holds
- The cross-recording stages the pipeline dispatches in process and records under its own job names
- The prerequisite chain that orders the three job types, and what a blocked forging job means
- The concurrency ceiling the assembly job type carries
- The per-session assembly output and the two gates that guard it
- What cleaning the `forging` pipeline destroys

**Does not cover:**
- Batch preparation, execution, monitoring, cancellation, reset, and the cleaning mechanic. Owned by
  `/batch-processing`.
- Dataset composition, animal and session membership, and the dataset state artifact. Owned by `/dataset-definition`.
- Core and memory estimates, the resource model, and per-unit planning. Owned by `/job-planning`.
- The project manifest and the project job artifact. Owned by `/project-state`.
- Remote submission, scheduler reporting, and remote project discovery. Owned by `/remote-execution`.
- What a session must already carry before it joins a dataset. Owned by `/processing-input-format`.
- Output discovery, the dataset hierarchy on disk, and result interpretation. Owned by `/processing-results`.
- The meaning of each assembled column. Owned by `mesoscope:mesoscope-vr-processing-schema` for Mesoscope-VR.

**Handoff rules:** A request to build, extend, rebuild, or inspect the membership of a dataset goes to
`/dataset-definition`, which owns every mutation of the hierarchy this pipeline reads. A request that names any
prepare, execute, monitor, cancel, reset, or clean mechanic goes to `/batch-processing` with `pipeline="forging"`, and
this skill supplies only the forging-specific arguments and readings that call needs.

---

## Agent requirements

You MUST drive this pipeline through the sollertia-forgery MCP tools `/batch-processing` owns, passing
`pipeline="forging"`. Do not import `sollertia_forgery.forging` functions and do not run `slf` commands. The `slf` CLI
is the human path and `/cli-reference` owns it. Where the MCP tools are unavailable, invoke
`/forging-mcp-environment-setup`.

You MUST confirm the dataset has been defined before preparing a forging batch. Preparation resolves jobs from the
hierarchy `define_forging_dataset_tool` built and never builds it, so a dataset that does not exist yet is an error
rather than an empty batch.

You MUST pass dataset roots wherever a forging batch names units, never session paths. A dataset root is the one
`define_forging_dataset_tool` reported or one `list_project_datasets_tool` lists, both owned by `/dataset-definition`.
`assets:session-discovery` is the exclusive producer of the session lists that compose a dataset, and the session paths
it returns are the units the five session pipelines take, never the unit a forging batch takes.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## The dataset processing unit

`forging` is the only batch pipeline whose processing unit is a dataset. `orchestration/preparation.py::prepare_batch`
selects `DATASET_UNIT` for `ProcessingPipelines.FORGING` and `SESSION_UNIT` for every other member of
`BATCH_PIPELINES`. `orchestration/preparation.py::_UNIT_DEPTHS` records that a dataset root sits one level under its
project root, where a session root sits two. Every unit named in one batch must still belong to a single project,
because the plan and state artifacts that resolve a batch are written per project.

The parameter that carries the unit list changes name from tool to tool, and a forging batch fills every one of them
with dataset roots. Execution is the exception, because it names prepared batch identifiers rather than any path.

| Tool                           | Parameter       | What a forging batch passes                                |
|--------------------------------|-----------------|------------------------------------------------------------|
| `prepare_batch_tool`           | `session_paths` | Dataset root directories                                   |
| `inspect_job_resources_tool`   | `session_paths` | Dataset root directories                                   |
| `reset_processing_jobs_tool`   | `unit_paths`    | Dataset root directories                                   |
| `clean_processing_output_tool` | `session_paths` | Dataset root directories                                   |
| `get_processing_status_tool`   | `session_paths` | Dataset root directories, matched against each job's unit  |
| `execute_jobs_tool`            | `batch_ids`     | Identifiers a forging preparation returned, never any path |

A local status read reports the dataset root under each job's `session_path` key, because that key carries the job's
unit path whatever the unit kind is.

The tracker is one per dataset rather than one per session. `forging/pipeline.py::forging_tracker_path` places
`ProcessingTrackers.FORGING`, the file `forging_tracker.yaml`, beside the dataset's own marker at the dataset root.
`shared_assets/pipelines.py::resolve_session_tracker_path` refuses the forging pipeline outright, since
`SESSION_PIPELINES` lists only the pipelines that record a per-session tracker.

---

## The three job types

All three job names are declared in `forging/pipeline.py` and re-exported by `forging/__init__.py`. Their scopes are
the ones `forging/state.py::_DATASET_JOB_SCOPES` records, and a dataset state read reports each job under that scope.

| Job name                | Constant                       | Scope     | Specifier             | Exists for                                                                    |
|-------------------------|--------------------------------|-----------|-----------------------|-------------------------------------------------------------------------------|
| `multiday_discovery`    | `MULTIDAY_DISCOVERY_JOB_NAME`  | `animal`  | The animal identifier | Each animal whose acquisition system resolves a multi-recording configuration |
| `multiday_extraction`   | `MULTIDAY_EXTRACTION_JOB_NAME` | `session` | The session name      | Each session of such an animal                                                |
| `session_data_assembly` | `FORGING_JOB_NAME`             | `session` | The session name      | Every session the dataset holds                                               |

`forging/pipeline.py::_build_forging_universe` composes the universe in that order, one discovery job per tracked
animal, one extraction job per that animal's session, then one assembly job per session in the dataset. An animal is
tracked across recordings exactly when `MULTI_RECORDING_CONFIGURATION_FILENAME` sits in its dataset directory, and
`define_forging_dataset` is the only writer of that file, so a dataset whose sessions carry no cross-recording imaging
carries assembly jobs alone.

Discovery reads nothing outside the dataset hierarchy. `discover_forging_jobs` loads the dataset marker and each
animal's configuration, never a session marker, so a dataset whose source sessions have moved off the machine still
resolves its jobs. Its possible subset equals its universe, because a session is admitted into the hierarchy only once
it carries the single-day outputs the forging stages consume.

An assembly job produces exactly one `data.feather` for its session and re-exports the source session's shared raw
assets beside it. What each of its columns means is recorded once per dataset, in the `data_descriptions.feather`
companion at the dataset root, because the column universe is donated per acquisition system. `/processing-results`
owns where those files sit on disk.

---

## Where the cross-recording stages run

Both cross-recording stages are jobs of the forging pipeline itself, dispatched in process by
`forging/pipeline.py::_run_multiday_job`, which calls the cindra imaging library's `prime_dataset` and
`execute_multi_recording_job` against the animal's materialized configuration. That library records the job's state
directly on the forging tracker under the identifier the forging universe issued, so one tracker holds all three job
types for the dataset.

The opposite assumption costs a whole batch. The two stages are owned by an upstream library that also exposes them
under its own pipeline. An agent reading only the stage names therefore concludes that a separate run has to finish
first, prepares a batch naming the assembly jobs alone, and then cannot explain why every one of them reports blocked.

The vocabularies differ on purpose. `forging/pipeline.py::_MULTIDAY_JOB_NAMES` maps cindra's own two multi-recording
job names onto this library's `multiday_discovery` and `multiday_extraction`, and its docstring calls that table the
only place the two vocabularies meet. Every tracker entry, every status reading, and every `job_names` filter therefore
uses this library's names, and cindra's own two names appear nowhere on a forging tracker.

Priming is carried by the first stage of each animal alone. `_resolve_multiday_stages` sets a stage's prime flag from
whether cindra reports it as having no prerequisite, so the animal's discovery job writes the shared bootstrap before
its own tracked work starts. A bootstrap that cannot be written therefore leaves the job unstarted rather than recorded
as failed, and a later stage still runs correctly on a run that skips the stage which primed it.

---

## Prerequisite ordering

`forging/pipeline.py::forging_job_prerequisites` returns one chain per session of a tracked animal.

```text
multiday_discovery(animal) -> multiday_extraction(session) -> session_data_assembly(session)
```

An assembly job whose session has no tracked extraction job carries an empty prerequisite tuple, so the assembly jobs
of a dataset with no cross-recording tracking are unordered among themselves. A job whose animal is no longer in the
dataset also keeps an empty tuple, so every job in the universe carries an entry.

The chain is three deep and blocking propagates through it. An animal whose discovery job is neither queued in the
batch nor already succeeded therefore blocks that animal's extraction jobs, and through them the assembly jobs of the
same sessions. Preparing the whole dataset in one batch queues every stage of the chain, which is why a forging batch
is prepared per dataset rather than per session. `/batch-processing` owns the blocked-job reporting and the general
rule that produces it.

---

## Processing workflow

1. **Confirm the dataset exists.** Ask `/dataset-definition` for the project's datasets before preparing anything. A
   name that no hierarchy backs fails preparation with the dataset marker's own `FileNotFoundError` wrapped in the
   preparation error, and composing the dataset is that skill's work rather than an argument to this pipeline.

2. **Read the recorded job state first.** `/dataset-definition` owns the dataset state artifact, which reports every
   tracked job with its scope, animal, session, and status. Read it whenever the dataset may already have been forged,
   so a resumed run knows which animals and which sessions are outstanding before anything is prepared.

3. **Prepare with dataset roots.** Call the preparation tool with `pipeline="forging"` and the dataset roots as its
   unit list. Pass no `options`, since `forging` reads none. The one key `options` carries anywhere in this library
   belongs to a different pipeline.

4. **Account for the blocked list before executing.** Read the preparation's blocked entries and confirm each one names
   a prerequisite the same batch queues. A blocked assembly job whose extraction prerequisite is absent from the batch
   means the batch named a subset of the dataset rather than the dataset.

5. **Execute the prepared batch.** `/batch-processing` owns the dispatch, the budgets, and the identifier recovery path
   for a lost batch identifier.

6. **Monitor by job name.** Filter a status read on `job_names` to separate the three types, and report the
   cross-recording jobs by animal and the assembly jobs by session, matching the scope each job type declares. A
   dataset of many sessions produces far more assembly jobs than cross-recording jobs, so an unfiltered listing buries
   the stages that gate everything else.

7. **Verify through the state reads.** There is no output-verification tool and no feather-query tool on this server.
   Confirm a forged dataset from the dataset state read and the job breakdowns `/batch-processing` and `/project-state`
   expose, then hand off to `/processing-results` for the outputs themselves.

8. **Route a failure by its stage.** A failed cross-recording job and a failed assembly job have different causes and
   different remedies, listed in the next section. Reset and re-execute rather than clean, since cleaning this pipeline
   discards the whole dataset.

---

## Resource management

`session_data_assembly` is the one forging job type that declares a concurrency ceiling.
`forging/pipeline.py::FORGING_JOB_CONCURRENCY_LIMITS` sets it to `4`, and `orchestration/dispatch.py` folds that
mapping into `_JOB_CONCURRENCY_LIMITS`, so at most four sessions assemble at once no matter how wide the host is. An
assembly job holds one core, so the ceiling rather than the core budget sets the width of the pool the stage opens.

The two cross-recording job types declare no ceiling. Each takes a wide core allocation of its own, read at import from
the installed cindra distribution, so the core budget already bounds how many run at once. Never quote those widths
from memory. Read the live figures through `read_resource_model_tool`, which `/job-planning` owns.

---

## Assembly failure modes

Every row below is specific to this pipeline. `/batch-processing` owns the generic dispatch and tracker failures.

| Condition                                                         | Where it surfaces                                                      | Remedy                                                                                                 |
|-------------------------------------------------------------------|------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The named dataset has no marker under the project root            | Preparation, as a wrapped `FileNotFoundError`                          | Define the dataset through `/dataset-definition`                                                       |
| The dataset carries no `data_descriptions.feather` companion      | Pipeline initialization, before any job runs                           | Recreate the dataset, which writes the companion at creation                                           |
| The assembler wrote a column the dataset does not describe        | One assembly job, as `ValueError` naming every undescribed column      | The acquisition system's description donation is incomplete, so extend it through `/library-extension` |
| A source session lacks a shared asset it is required to re-export | One assembly job, as `FileNotFoundError` naming the asset and its path | Restore the source session, since the check runs before any expensive work                             |
| A dispatched job identifier matches no job of this dataset        | The job, as `ValueError` listing every valid identifier                | Re-read the identifiers from a freshly prepared batch                                                  |

The column gate is the one failure worth understanding rather than merely recognizing.
`forging/pipeline.py::_forge_session` reads the assembled file's schema after the donated assembler writes it and
refuses any column the description companion does not describe. That companion is filled at dataset creation from
`registries.py::resolve_forging_column_descriptions` while the assembler comes from
`registries.py::resolve_forging_assembly_worker`, so a column that one donation emits and the other omits marks a gap in
the acquisition system's registration rather than a data fault.

---

## Cleaning a forged dataset

Cleaning this pipeline is the most destructive operation on the batch surface. The forging pipeline's output path is
the dataset root itself, so a clean removes the entire dataset hierarchy, every assembled per-session feather included,
alongside the tracker. The dataset then has to be defined again before anything can be forged into it.

Reset the tracked jobs instead whenever the failure cause was external and the hierarchy is still sound. Reset takes
dataset roots in `unit_paths` and returns each named dataset's jobs to the scheduled state without touching a single
assembled file, and `/batch-processing` owns its arguments and its return tree.

---

## Related skills

The `cindra:` entry below resolves through the cindra marketplace. Every other entry resolves inside the sollertia
marketplace.

| Skill                                      | Relationship                                                              |
|--------------------------------------------|---------------------------------------------------------------------------|
| `/batch-processing`                        | Owner: every prepare, execute, monitor, cancel, reset, and clean mechanic |
| `/dataset-definition`                      | Prerequisite: builds the hierarchy this pipeline resolves its jobs from   |
| `/job-planning`                            | Upstream: per-dataset planning and the live resource model                |
| `/processing-input-format`                 | Reference: what a session carries before it joins a dataset               |
| `/processing-results`                      | Downstream: the assembled outputs and how to read them                    |
| `/project-state`                           | Reference: project-wide job and manifest artifacts                        |
| `/remote-execution`                        | Variant: forging a dataset that lives on the compute server               |
| `/library-extension`                       | Extension: donating an assembler and its column descriptions              |
| `/pipeline`                                | Context: where dataset forging sits in the end-to-end pipeline            |
| `assets:session-discovery`                 | Upstream: the session lists a dataset is composed from                    |
| `mesoscope:mesoscope-vr-processing-schema` | Reference: the Mesoscope-VR columns and their meanings                    |
| `cindra:multi-recording-processing`        | Reference: the upstream stages this pipeline dispatches in process        |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' SKILL.md finds nothing

Dataset forging run, tool-settled (read the dataset state through /dataset-definition):
- [ ] sollertia-forgery MCP server is connected
- [ ] The dataset was defined before any batch was prepared
- [ ] The prepared batch named dataset roots, and named every dataset the run covers
- [ ] Every blocked entry the preparation reported names a prerequisite the same batch queues
- [ ] Every tracked job reads succeeded, or reads failed with an error message that was routed by its stage

Dataset forging run, agent-judged:
- [ ] The three job types were reported under their own names, never under the upstream library's names
- [ ] Cross-recording jobs were reported by animal and assembly jobs by session
- [ ] Resource figures for the cross-recording stages were read from the live resource model, never quoted from memory
- [ ] A failure was retried by reset rather than by cleaning, unless the hierarchy itself had to be rebuilt
- [ ] The user was told that cleaning this pipeline destroys the whole dataset before any clean was called
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
