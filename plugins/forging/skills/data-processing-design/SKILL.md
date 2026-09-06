---
name: data-processing-design
description: >-
  Documents the durable design pattern behind sollertia-forgery data processing. Covers the agnostic worker packages and
  the per-system donations each pipeline dispatches through a resolver, the registry seam with its import-time coverage
  check, the pipeline dispatch table, the plan, prepare, execute, close job model, and the resource admission rules. Use
  when adding a processing stage or pipeline, auditing the agnostic versus per-system split, or deciding whether a
  concern belongs in this library or in one of its upstream dependencies.
user-invocable: false
---

# Data processing design

Documents the durable design pattern behind sollertia-forgery data processing, at the package, registry, and
orchestration layer. A pipeline reads one class of acquired data, writes its outputs beside the session it processed,
and records every job it ran on a per-unit tracker. That shape holds for every acquisition system, and what differs
between systems arrives as registered data rather than as a branch inside a pipeline.

---

## Scope

**Covers:**
- The layers of the library, and the one-way import rule between an agnostic worker package and a per-system package
- The thirteen registries, their key shapes, the resolver accessors that read them, and the two public donation
  Protocols
- The import-time coverage check, and how to read its `RuntimeError` as the remaining wiring checklist
- The `PipelineDispatch` table, `BATCH_PIPELINES` membership, and the import-time check that holds the two in step
- The plan, prepare, execute, close job model, job identity, per-unit trackers, and the four job statuses
- The resource admission model: declared core allocations, hard ceilings, soft reservations, the two memory figures a
  job's footprint carries, and the admission passes
- Feather as the cross-stage interchange format
- The seam between the stages this library implements and the stages it hands to an upstream library in-process
- The agnostic event-stream merging primitive this library owns

**Does not cover:**
- Preparing and executing a batch, and reading its status. Owned by `/batch-processing`.
- Planning a unit, projecting a project plan, and inspecting job resources. Owned by `/job-planning`.
- The step-by-step touch points for adding a system, a pipeline, a stage, or an MCP tool. Owned by `/library-extension`.
- The `slf` command surface the rendered job argument vectors invoke. Owned by `/cli-reference`.
- The scheduler backend, the submission ledger, and the transport settings. Owned by `/remote-execution` and
  `/server-configuration`.
- The on-disk schema of every artifact a stage writes. Owned by `/processing-results`.
- The archives each pipeline's first stage consumes. Owned by `/processing-input-format`.
- The project-level rollups of tracker state. Owned by `/project-state`.
- Any one acquisition system's donated parsers, columns, locators, resolvers, and admission policy. Owned by the
  `mesoscope:mesoscope-vr-*` skill family, whose members the worked example below names one per seam.
- The upstream microcontroller extraction schema, event-code partitioning, and typed value readers. Owned by
  `communication:log-processing-results`.
- The `AcquisitionSystems` member and the per-system session records a new system registers upstream. Owned by
  `assets:library-extension`.

**Handoff rules:** Send a request that runs a job to `/batch-processing` or `/job-planning`, a request that reads an
artifact to `/processing-results` or `/project-state`, and a request for the exact code touch points of an extension
to `/library-extension`. A question about one acquisition system's donated values belongs to the mesoscope plugin,
and this skill answers only what every system's donation must satisfy.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## Two-layer architecture: agnostic pipelines versus per-system donations

The library is one agnostic stack plus one package per registered acquisition system, and the two meet in exactly one
module.

| Layer               | Package                                                                           | What it owns                                                                                                      |
|---------------------|-----------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Agent interface     | `interfaces/`                                                                     | The `slf` CLIs and the `slf mcp` server exposed by installing the library                                         |
| Orchestration       | `orchestration/`                                                                  | Planning, preparation, dispatch, the execution hosts, the local engine, the scheduler backend, the batch registry |
| Remote transport    | `server/`                                                                         | SSH and SLURM execution, non-interactive job assembly, remote project discovery, server configuration             |
| Worker packages     | `managing/`, `runtime/`, `microcontrollers/`, `video/`, `two_photon/`, `forging/` | One pipeline each, running identically for every acquisition system                                               |
| Donation seam       | `registries.py`                                                                   | The `AcquisitionSystems`-keyed dispatch registries and the import-time checks that guard them                     |
| Per-system packages | one package per registered system                                                 | The parsers, locators, resolvers, workers, and policy data that system donates                                    |
| Agnostic substrate  | `shared_assets/`                                                                  | Pipeline identity, tracker resolution, the OpenMP guard, and the shared frame and terminal utilities              |

Each worker package owns one pipeline and its job-name constants, with `managing/` holding the two management
pipelines. A worker package resolves its own job universe from the acquisition data it reads, so a completed tracker
already accounts for every source the unit recorded.

**The one-way import rule.** A per-system package never imports a worker package. The arrows already run from a
worker package to `registries.py` to every per-system package, because each worker pipeline does a top-level
`from ..registries import resolve_*` and `registries.py` does a top-level import of each per-system package. An
import in the reverse direction closes that loop, which Python resolves as a partially initialized module at import
time, so the rule states a genuine cycle rather than a style preference.

The one permitted upward import from a per-system package is `..shared_assets`. That import is safe because
`shared_assets/` imports nothing from `sollertia_forgery` at all, reaching only the standard library and the
third-party stack. Being a leaf is the structural reason the substrate exists as its own package rather than as a
module inside a worker package.

Every pipeline that dispatches a parallelized worker pool calls `shared_assets.verify_openmp_runtime()` as the first
statement of its entry point. A host that cannot run a parallelized kernel therefore fails before it has done any work,
rather than partway through a unit. The manifest pipeline runs single-threaded and is the one pipeline that does not.

---

## How system specificity enters as data

System specificity reaches a pipeline only through `registries.py`, and a pipeline reads it through a `resolve_*`
accessor rather than by indexing a registry.

| Registry                                 | Key                                | What it dispatches                                                    |
|------------------------------------------|------------------------------------|-----------------------------------------------------------------------|
| `_MICROCONTROLLER_PARSER_REGISTRY`       | `(system, module_type, module_id)` | One parser per hardware module the system can parse                   |
| `_MICROCONTROLLER_EVENT_CODE_REGISTRY`   | `AcquisitionSystems`               | An accessor returning the event codes each parseable module reads     |
| `_MICROCONTROLLER_ELIGIBILITY_REGISTRY`  | `AcquisitionSystems`               | The modules one loaded session configured for use                     |
| `_FORGING_ASSEMBLY_REGISTRY`             | `AcquisitionSystems`               | The per-session assembly worker, paired with its column descriptions  |
| `_ASSEMBLY_GEOMETRY_REGISTRY`            | `AcquisitionSystems`               | The frame height and the source heights its assembler holds           |
| `_ASSEMBLY_SOURCE_REGISTRY`              | `AcquisitionSystems`               | The height at which its assembler holds each source it reads          |
| `_FORGING_ADMISSION_REGISTRY`            | `AcquisitionSystems`               | The pipelines each session type completes before it joins a dataset   |
| `_CINDRA_CONFIGURATION_REGISTRY`         | `AcquisitionSystems`               | The single-recording and multi-recording configuration resolvers      |
| `_MULTI_RECORDING_SESSION_TYPE_REGISTRY` | `AcquisitionSystems`               | The session types the system tracks across recordings                 |
| `_RUNTIME_PARSER_REGISTRY`               | `AcquisitionSystems`               | The runtime log source identifier, paired with its parser             |
| `_TWO_PHOTON_DATA_REGISTRY`              | `AcquisitionSystems`               | The locator of the raw two-photon imaging directory                   |
| `_POSE_PREDICTION_REGISTRY`              | `AcquisitionSystems`               | The locator of the externally produced pose-prediction file           |
| `_VIDEO_TRACKING_REGISTRY`               | `AcquisitionSystems`               | The pass that reads those predictions and writes the tracking outputs |

Twelve registries key on `AcquisitionSystems` alone. `_MICROCONTROLLER_PARSER_REGISTRY` is the only one keyed on a
three-part tuple, and `resolve_microcontroller_parsers` is what flattens it into the `(module_type, module_id)` mapping
a pipeline consumes. Three donations carry no callable at all, since `_FORGING_ADMISSION_REGISTRY` holds a
`dict[SessionTypes, frozenset[ProcessingPipelines]]`, `_MULTI_RECORDING_SESSION_TYPE_REGISTRY` holds a
`frozenset[SessionTypes]`, and the column-description half of `_FORGING_ASSEMBLY_REGISTRY` is a `dict[str, str]`.

See [registry-accessors.md](references/registry-accessors.md) for the `resolve_*` accessor roster with the return type
of each accessor, for the donation Protocols, and for the entry a system donates when it produces none of a data class.

---

## The import-time coverage check

`registries.py` ends with a bare call to `_assert_registry_coverage()`, so the checks run on the first import of the
module, which every pipeline import transitively triggers.

| Check | What it requires                                                                            |
|-------|---------------------------------------------------------------------------------------------|
| 1     | Every registered system appears in each of the thirteen registries, tested in a fixed order |
| 2     | Every module registered for a parser also declares the event codes that parser reads        |
| 3     | Every session type a system tracks across recordings is a session type that system records  |
| 4     | Every session type a system admits into a dataset is a session type that system records     |

Checks 3 and 4 both subtract the per-system session types that `sollertia-shared-assets` declares, since the types a
system records are that library's to define. Check 4 is deliberately asymmetric, so a declared type the system does
not record fails, while a recorded type the admission mapping omits is the supported way to say that the type joins
no dataset.

Read the raised `RuntimeError` as the remaining wiring checklist. Every raise goes through `console.error` and stops
the import, so one import reports one problem, an extender fixes it, imports again, and reads the next. Discovery
order is check 1 over its thirteen registries, then checks 2, 3, and 4. The first check names the offending systems by
enum member name, and the last two name the offending session types by enum value.

```text
Unable to validate donor-registry coverage for {registry_name}. Every acquisition system must register its donated processing and forging assets in this module ('registries.py'), but entries are missing for {missing_names}.
```

Check 2 exists because the extraction stage filters each module by the codes the event-code registry resolves.
Dropping a parseable module from that registry would remove it from its controller's extraction configuration, which
would leave its parse job undiscovered rather than failing outright.

---

## The pipeline dispatch table

`orchestration/dispatch.py` binds each batch pipeline to one frozen `PipelineDispatch` entry, and `resolve_dispatch`
is the only lookup into that table.

| Field           | What the entry supplies                                                                              |
|-----------------|------------------------------------------------------------------------------------------------------|
| `pipeline`      | The `ProcessingPipelines` member this entry dispatches                                               |
| `unit_kind`     | `SESSION_UNIT` or `DATASET_UNIT`, the unit on which its jobs operate                                 |
| `load`          | Loads the unit from its root, reading its markers alone                                              |
| `discover`      | The job resolver, returning the loaded unit, the job universe, and the possible subset               |
| `worker`        | The picklable module-level worker the process pool invokes with one planned job                      |
| `prerequisites` | Resolves each job's upstream jobs, taking the loaded unit because specifiers sit at differing scopes |
| `tracker_path`  | Resolves the pipeline's tracker path from a loaded unit                                              |
| `output_path`   | The directory the pipeline owns outright, or `None` when it writes beside the acquired data          |
| `unit_name`     | The name by which every tool response reports the unit                                               |
| `size_jobs`     | Sizes each job of the universe from its input, given the declared cores                              |
| `command`       | Renders the argument vector that runs one job on a host holding the data                             |
| `prime`         | Materializes state the unit needs before its jobs resolve, defaulting to `None`                      |

`prime` and `external_output_paths` are the two fields carrying a default, each of them `None`. The two-photon pipeline
is the one entry that supplies a `prime`, because `cindra` requires a single-threaded step that writes the shared
configuration before any of its jobs reads it. The forging pipeline is the one entry that names external output paths,
because its cross-recording stages write outside the dataset the pipeline processes.

`BATCH_PIPELINES` holds six members, namely `checksum`, `runtime`, `microcontroller`, `video`, `two_photon`, and
`forging`. The `manifest` pipeline is a `ProcessingPipelines` member that operates on a project, and it carries no
dispatch entry. `_assert_dispatch_coverage()` runs at the bottom of the module and compares the dispatch table's keys
against `BATCH_PIPELINES` on their symmetric difference, so a pipeline named in one and absent from the other fails
the import from either side. Adding a batch pipeline is therefore a single change that touches both.

---

## The plan, prepare, execute, close job model

Work reaches a host as a job, and every pipeline models its jobs the same way.

1. **Plan.** Per unit, `orchestration/planning.py` registers on the unit's tracker every job that unit is able to run
   and records each job's cores, memory, and upstream job identifiers into a per-unit `job_plan.yaml`. A unit runs
   only the jobs registered on its tracker. `generate_project_plan` projects every cache under a project into one
   table for a host that holds none of the data.
2. **Prepare.** `orchestration/preparation.prepare_batch` joins the plan rows to the recorded state rows into one
   descriptor per job and returns a `BatchDocument`, which `orchestration/batches.record_prepared_batch` stores under
   a batch identifier.
3. **Execute.** Two backends run one prepared document. The local engine drives `job_execution_manager` over a
   `JobExecutionState`, admitting jobs against a core budget and an anonymous memory budget and dispatching them onto
   a shared `ProcessPoolExecutor`. The remote engine submits one scheduler allocation per job, each declaring the
   resident memory its own estimate reported and sequenced through an `afterok` dependency.
4. **Close.** `orchestration/closure.close_batch` snapshots what a finished batch's jobs recorded, records that
   outcome beside the batch, and retires the prepared document. It touches no submission ledger.

A job is reported **blocked** rather than dispatched when the run can neither queue its upstream stage nor confirm
that the stage already succeeded. Blocking propagates to the dependents through a fixed-point pass, so a blocked job
never reaches an execution backend and closure counts it toward the batch total without it ever running. A
prerequisite that already succeeded satisfies its dependents even though it is absent from the batch.

---

## Job identity, trackers, and the four statuses

A job identifier is derived from the job name and the specifier alone, through `ProcessingTracker.generate_job_id`,
so the same stage of two different units shares one identifier. `PendingJob.dispatch_key` pairs that identifier with
the unit path, and the pairing is what keeps one unit's completed stage from satisfying every unit's.

Each unit records its jobs in a per-unit tracker YAML written under a file lock, so a unit's state travels with the
unit's data rather than with the host that processed it. `shared_assets.resolve_session_tracker_path` resolves the
tracker of each per-session pipeline and raises for a pipeline outside `SESSION_PIPELINES`, since the manifest
pipeline tracks a project and the forging pipeline tracks a dataset.

| Status      | Meaning                                                  |
|-------------|----------------------------------------------------------|
| `SCHEDULED` | The job is registered on the tracker and has not started |
| `RUNNING`   | The job is executing, and the record names its executor  |
| `SUCCEEDED` | The job completed and its output is on disk              |
| `FAILED`    | The job raised, and a reset returns it to `SCHEDULED`    |

Every artifact stores the status as the enum member name, so those four uppercase strings are the literal values a
status column carries. Discovery aligns a requested subset against the full universe, through the tracker's
`align_jobs` call, so a partial invocation never retires a sibling job the unit can still run. The tracker is the
authority on whether a stage ran and the files are the authority on what it produced, and the library never
re-derives success from the presence of an output file.

---

## The resource admission model

Three tables in `orchestration/dispatch.py` describe a job type, and all three key on the tracker job name.

| Table                           | Holds                                                   | Absence means                            |
|---------------------------------|---------------------------------------------------------|------------------------------------------|
| `_JOB_CORE_ALLOCATIONS`         | The cores one job of the type occupies                  | A refusal, described below               |
| `_JOB_CONCURRENCY_LIMITS`       | A hard ceiling on how many jobs of the type run at once | The type is bounded by the budgets alone |
| `_JOB_CONCURRENCY_RESERVATIONS` | A soft reservation the first admission pass honors      | The type takes whatever a pass leaves    |

Every stage this library owns dispatches at its declared width, because each holds one shape whatever data it reads. A
stage a dependency owns is sized whole by that dependency, which answers with the width it picked for the job's own
input. The entry here therefore restates the dependency's figure rather than deciding it, so a retune upstream reaches
the table without an edit.

A job type declaring no core figure is refused at two layers. `resolve_job_cores` raises a `ValueError` during
planning, so the unit's plan fails before any batch is prepared, and `resolve_core_allocations` raises at local
execution and lists every unregistered name. There is no matching error for a missing ceiling or reservation, since
absence there is the ordinary case.

A job's footprint carries two memory figures, and which one binds depends on the host that runs the job. `memory_mb`
is the anonymous memory the job holds at its peak, and the local pool budgets against it, because anonymous pages are
the ones a host cannot reclaim under pressure. `resident_mb` adds to that term the bytes the job memory-maps and the
file-backed library image every job holds, carries a tolerance above the sum, and rounds to a gigabyte. One scheduler
allocation declares that second figure, because a job declaring less than it holds resident drives the node into
reclaim. A stage reading its input through the file interface maps nothing, so its two figures part by the image and
the tolerance alone, while a stage that maps its input reports a resident figure well above its anonymous one.

Admission recomputes the committed cores and memory from the running set on every pass, and orders the candidates by
descending dispatch priority with ties broken on the larger memory footprint. Dispatch priority is the summed core
weight of a job's transitive dependents, so the root of a long chain outranks a crowd of leaves that would otherwise
hold the budget while the host idles behind them. A ceiling stands in every pass however much capacity is idle. A
reservation binds only in the first of two passes, so the reserved room is offered to every other runnable job before
the reservation is released over whatever remains. The scan continues past a job that does not fit, letting smaller
jobs backfill spare capacity, and a job larger than the whole budget is admitted only when nothing is running and the
pass has admitted nothing. That floor is applied after the prerequisite and ceiling checks, so it never dispatches a
job whose input does not exist and never breaches a ceiling.

---

## Feather as the cross-stage interchange format

Every stage that hands data to another stage writes an uncompressed Arrow IPC feather. Polars defaults `write_ipc` to
`compression="uncompressed"`, and each writer in the library either passes that value or accepts the default, so every
feather it produces is memory-mappable. A downstream stage memory-maps it on read rather than paying a decompression
pass. The project-level rollups use the same format, so a submitting host that holds none of the data reads a project's
plan, manifest, jobs, and dataset-state tables exactly as a worker reads a stage output.

The pipeline owns the interchange path and the schema of every table it writes itself, while the acquisition system's
donations name the tables their parsers and assembly worker produce. A stage therefore stays readable by a tool that
knows the format without knowing the system, which is what lets one status and one verification surface serve every
pipeline.

---

## Cross-stage seams: implemented here versus delegated in-process

| Stage                                                       | Implemented by                                              | Job-name constant                          |
|-------------------------------------------------------------|-------------------------------------------------------------|--------------------------------------------|
| `checksum_resolution`                                       | This library, hashing through `ataraxis-data-structures`    | `managing.checksum.CHECKSUM_JOB_NAME`      |
| `runtime_processing`                                        | This library decodes, the donated runtime parser interprets | `runtime.pipeline.RUNTIME_JOB_NAME`        |
| `microcontroller_data_extraction`                           | `ataraxis-communication-interface`, called in-process       | Its `CONTROLLER_EXTRACTION_JOB_NAME`       |
| `module_parsing`                                            | This library partitions, the donated module parser writes   | `microcontrollers.pipeline.PARSE_JOB_NAME` |
| `camera_timestamp_extraction`                               | `ataraxis-video-system`, called in-process                  | Its `CAMERA_EXTRACTION_JOB_NAME`           |
| `camera_timestamp_rename`                                   | This library                                                | `video.pipeline.RENAME_JOB_NAME`           |
| `pose_tracking`                                             | The donated video-tracking function                         | `video.pipeline.TRACKING_JOB_NAME`         |
| `motion_energy`                                             | This library                                                | `video.pipeline.ENERGY_JOB_NAME`           |
| `binarization`, `registration`, `processing`, `combination` | `cindra`'s single-recording binding, called in-process      | Its `SingleRecordingJobNames` enum         |
| `multiday_discovery`, `multiday_extraction`                 | `cindra`'s multi-recording binding, called in-process       | This library's own two job names           |
| `session_data_assembly`                                     | The donated forging assembly worker                         | `forging.pipeline.FORGING_JOB_NAME`        |
| `manifest_generation`                                       | This library                                                | `managing.manifest.MANIFEST_JOB_NAME`      |

A delegated stage resolves its inputs here, calls the upstream library's job binding in-process, and records its work
under that library's own exported job-name constant, so a tracker identifier stays aligned with the library that
produced the work. The cross-recording pair is the deliberate exception, since those two stages are also
dependency-owned yet are recorded under `MULTIDAY_DISCOVERY_JOB_NAME` and `MULTIDAY_EXTRACTION_JOB_NAME` through
`forging/pipeline.py::_MULTIDAY_JOB_NAMES`. That mapping is the only place the two vocabularies meet, so composing
another upstream stage into the forging graph is a matter of naming it there.

The same seam governs the job-model helpers. This library reuses each dependency's job resolvers, output-path
resolvers, priming entry points, and the shared `ProcessingTracker` rather than reimplementing any of them, and it
composes an output path deterministically instead of discovering one by glob.

---

## The agnostic microcontroller primitive

`shared_assets/microcontroller.py` holds one public asset, and a donated module parser calls it whenever one output
column draws on two event codes.

```python
def merge_event_streams[ScalarT: np.generic](
    timestamps_a: NDArray[np.uint64],
    values_a: NDArray[ScalarT],
    timestamps_b: NDArray[np.uint64],
    values_b: NDArray[ScalarT],
) -> tuple[NDArray[np.uint64], NDArray[ScalarT]]:
```

It concatenates both pairs and reorders them on a stable `argsort` of the `uint64` timestamps, keeping the first
stream ahead of the second on ties and returning the sorted timestamps and the values reordered to match.

The remaining microcontroller primitives belong to `ataraxis-communication-interface`. That library owns the extracted
message schema, the event-code partitioning a parse job runs before dispatching to its donated parser, the typed
timestamp and value readers a parser calls, and the module output path this library composes. Read
`communication:log-processing-results` for that surface and cite it there rather than restating it here.

---

## Worked example

`AcquisitionSystems` holds one member today, so a single acquisition system is the only registered donor and the only
concrete consumer of every pattern above. Its donations are documented by the mesoscope companion plugin, one skill per
seam group. Read `registries.py` itself for the donation table, which names every donated symbol of every registered
system in one place.

| Registry                                                         | Mesoscope-VR skill that documents the donation |
|------------------------------------------------------------------|------------------------------------------------|
| The three `_MICROCONTROLLER_*` registries                        | `mesoscope:mesoscope-vr-module-parsing`        |
| `_RUNTIME_PARSER_REGISTRY`                                       | `mesoscope:mesoscope-vr-trial-decomposition`   |
| `_POSE_PREDICTION_REGISTRY` and `_VIDEO_TRACKING_REGISTRY`       | `mesoscope:mesoscope-vr-video-tracking`        |
| `_TWO_PHOTON_DATA_REGISTRY` and `_CINDRA_CONFIGURATION_REGISTRY` | `mesoscope:mesoscope-vr-imaging-configuration` |
| `_MULTI_RECORDING_SESSION_TYPE_REGISTRY`                         | `mesoscope:mesoscope-vr-imaging-configuration` |
| `_FORGING_ASSEMBLY_REGISTRY` and `_FORGING_ADMISSION_REGISTRY`   | `mesoscope:mesoscope-vr-dataset-assembly`      |

The column-description half of `_FORGING_ASSEMBLY_REGISTRY` and the file name and column rosters every donation writes
against are documented by `mesoscope:mesoscope-vr-processing-schema`, and the fluorescence sub-assembly the assembly
worker calls by `mesoscope:mesoscope-vr-fluorescence-alignment`. `_ASSEMBLY_GEOMETRY_REGISTRY` and
`_ASSEMBLY_SOURCE_REGISTRY` have no companion skill of their own and are owned here, so read those two donations from
this skill, from `registries.py`, and from the sizing pass in `orchestration/footprints.py` that consumes them.

---

## Maintenance contract

This skill is updated when:

- A registry is added to or removed from `registries.py`, or a donation Protocol changes its call signature.
- A check is added to `_assert_registry_coverage` or `_assert_dispatch_coverage`, or an existing check changes what
  it requires.
- A `PipelineDispatch` field is added, removed, or given a new meaning.
- A pipeline joins or leaves `BATCH_PIPELINES`, or the plan, prepare, execute, close model gains a step.
- A resource table changes its meaning, such as a reservation binding in every admission pass rather than the first.
- The interchange format changes, or a stage moves across the implemented-versus-delegated seam.

This skill is NOT updated when:

- An acquisition system donates a new parser, column, locator, or admission entry. That belongs to the per-system
  instance skill.
- A worker package changes the internals of one stage. This skill documents the pattern that stage follows.
- A job type is retuned in a resource table. The figures live in the source, and a dependency's retune reaches the
  table without an edit.
- A tool is added to the MCP surface. That belongs to the skill that owns the tool and to `/library-extension`.

---

## Related skills

The `communication:` and `cindra:` entries below resolve through the ataraxis and cindra marketplaces. Every other
entry resolves inside the sollertia marketplace.

| Skill                                           | Relationship                                                               |
|-------------------------------------------------|----------------------------------------------------------------------------|
| `experiment:external-tool-bindings`             | Owns the convention behind an externally produced input this library reads |
| `/pipeline`                                     | Context: where each pattern here sits in the end-to-end route              |
| `/batch-processing`                             | Downstream: the tools that drive the prepare and execute steps             |
| `/job-planning`                                 | Downstream: the tools that drive the plan step and read the resource model |
| `/library-extension`                            | The exact code touch points for each extension scenario                    |
| `/dataset-definition`                           | Consumer: the admission policy applied when a dataset is defined           |
| `/dataset-forging`                              | Consumer: the dataset pipeline's own three-stage shape                     |
| `/processing-input-format`                      | The archives each pipeline's first stage consumes                          |
| `/processing-results`                           | The on-disk schema of every artifact a stage writes                        |
| `/project-state`                                | The project-level rollups of tracker state                                 |
| `/remote-execution`                             | The scheduler backend behind the remote execute step                       |
| `/server-configuration`                         | The transport settings the remote backend reads                            |
| `/cli-reference`                                | The `slf` commands a rendered job argument vector invokes                  |
| `/forging-mcp-environment-setup`                | Owner: the response contract and the server-health diagnostics             |
| `assets:library-extension`                      | The upstream enum and session-record side of registering a system          |
| `experiment:acquisition-system-design`          | Peer: the acquisition-side counterpart of this pattern skill               |
| `mesoscope:mesoscope-vr-processing-schema`      | The target file names and column rosters of every Mesoscope-VR donation    |
| `mesoscope:mesoscope-vr-module-parsing`         | The Mesoscope-VR donations behind the three microcontroller registries     |
| `mesoscope:mesoscope-vr-trial-decomposition`    | The Mesoscope-VR donation behind the runtime parser registry               |
| `mesoscope:mesoscope-vr-video-tracking`         | The Mesoscope-VR donations behind the two video registries                 |
| `mesoscope:mesoscope-vr-imaging-configuration`  | The Mesoscope-VR donations behind the three two-photon registries          |
| `mesoscope:mesoscope-vr-dataset-assembly`       | The Mesoscope-VR donations behind the two forging registries               |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | The Mesoscope-VR sub-assembly the forging assembly worker calls            |
| `communication:log-processing-results`          | The upstream microcontroller primitives and extracted-message schema       |
| `cindra:single-recording-processing`            | The dependency whose job bindings the imaging stages call in-process       |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Agnosticism:
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
- [ ] Every concrete donation is deferred to a mesoscope skill by name rather than carrying its value

Design review of a change to the library:
- [ ] The new or changed pipeline lives in its own agnostic worker package, with no per-system branch inside it
- [ ] Every per-system value the pipeline needs is reached through a registries.py resolver, never a registry index
- [ ] No per-system package imports a worker package, and its only upward import is shared_assets
- [ ] A new registry is added to the import-time coverage check and its accessor is exported from registries.py
- [ ] A new batch pipeline is added to BATCH_PIPELINES and given a PipelineDispatch entry in the same change
- [ ] Every new job type declares its cores in _JOB_CORE_ALLOCATIONS before any unit is planned
- [ ] A stage delegated to a dependency records its work under that dependency's exported job-name constant, the
      cross-recording pair excepted
- [ ] A stage handing data to another stage writes uncompressed Arrow IPC
- [ ] The import-time RuntimeError was read as the remaining wiring checklist, one problem per import
```
