# Extension recipes

Completes each `sollertia-forgery` extension scenario touch point by touch point. Every section names the files the
scenario edits and, inside each file, the exact constant, registry, mapping, or function that gains an entry. The
import-time checks that catch an unfinished recipe are documented in [guardrails.md](guardrails.md).

You MUST land the upstream half of a scenario before starting its sollertia-forgery half, because the upstream member
is what this library's coverage checks measure against. The upstream half is owned by `assets:library-extension`.

| Scenario                | Unit the change acts on                       | Upstream half is blocking      |
|-------------------------|-----------------------------------------------|--------------------------------|
| New acquisition system  | Every session and dataset the system produces | Yes, `sollertia-shared-assets` |
| New session type        | One session type of one system                | Yes, `sollertia-shared-assets` |
| New processing stage    | One pipeline that already exists              | No                             |
| New processing pipeline | A new category package                        | Yes, `sollertia-shared-assets` |
| New MCP tool            | One `interfaces/*_tools.py` module            | No                             |

---

## Conventions every scenario shares

**Every donated worker is a picklable module-level function.** The forging assemblers and the microcontroller parsers
are dispatched into spawned worker processes through a `ProcessPoolExecutor`, so a closure, a lambda, or a bound
method fails at pickling time rather than at registration time. The `_ForgingAssemblyAsset.assembler` field docstring
and the `PipelineDispatch.worker` field docstring both state the requirement.

**A pipeline never indexes a registry.** Every registry constant in `registries.py` is module-private, and the only
files that reference one are `registries.py` itself and two test modules. A consumer imports the matching `resolve_*`
accessor from `..registries` instead, which buys string-or-member normalization, a named `ValueError` in place of a
bare `KeyError`, and the freedom to change a registry's internal key shape without touching a pipeline.

**A system package never imports a category package.** The arrows already run category package to `registries.py` to
system package, so the reverse import closes a cycle Python resolves as a partially-initialized module. The one legal
upward import is `..shared_assets`, which is a leaf that imports nothing from this library. Nothing enforces the rule,
because `tox -e lint` runs `ruff format`, `ruff check --fix ./src ./tests`, and `mypy ./src` and no import-linter
contract exists. The enforcement is the circular `ImportError` itself.

**Reuse a dependency's job-name constant rather than minting a local string.** `CAMERA_EXTRACTION_JOB_NAME` comes from
`ataraxis_video_system.orchestration.jobs`, `CONTROLLER_EXTRACTION_JOB_NAME` from
`ataraxis_communication_interface.orchestration.jobs`, and `SingleRecordingJobNames` and `MultiRecordingJobNames` from
`cindra.orchestration.jobs`. A stage that wraps one of their bindings imports the constant it already exports.

**Verify by importing the package that runs each check.** `python -c "import sollertia_forgery.registries"` runs the
donor-registry coverage check, `python -c "import sollertia_forgery.orchestration"` adds the dispatch-table check, and
`python -c "import sollertia_forgery.managing"` adds the manifest status-column check. A bare `import sollertia_forgery`
runs none of them, because the top-level `__init__.py` re-exports no library symbol and its `__all__` is empty.
`slf --help` reaches all three, since `interfaces/entry_points.py` imports `interfaces/manage.py`, which imports both
`..managing` and `..orchestration`.

---

## Adding a new acquisition system

### Upstream first

| Repository                | Touch                   | Identifier                                                                                               |
|---------------------------|-------------------------|----------------------------------------------------------------------------------------------------------|
| `sollertia-shared-assets` | `enums.py`              | The new `AcquisitionSystems` member                                                                      |
| `sollertia-shared-assets` | `<system>/` subpackage  | `<System>HardwareState`, `<System>ExperimentConfiguration`, `<System>RawData` with `build(root) -> Self` |
| `sollertia-shared-assets` | `registries.py`         | `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`, `SYSTEM_RAW_DATA_REGISTRY` entries       |
| `sollertia-shared-assets` | `registries.py`         | `SYSTEM_SESSION_TYPES[<System>]`, which must be a non-empty frozenset                                    |
| `sollertia-experiment`    | The acquisition runtime | The artifacts every donation reads, written under the session's `raw_data`                               |

The moment the upstream member lands, sollertia-forgery stops importing. `_assert_registry_coverage()` in
`registries.py` fails, and every module that reaches it fails with it, which is every category pipeline,
`orchestration/dispatch.py`, `orchestration/footprints.py`, and therefore every `slf` command and every MCP tool. That
break is the remaining checklist, and it clears one registry at a time.

### Step 1: create `src/sollertia_forgery/<system>/`

Mirror the module split of the registered acquisition-system package. Each module donates the callables and the data
its registry expects, and the signatures are fixed by the six donation Protocols in `registries.py`.

| Module                  | What it declares                                                                                                                                                                                                                                                                                                            |
|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `microcontrollers.py`   | One module-level `parse_*` function per parsed hardware module, each `(event_partition: dict[int, pl.DataFrame], output_directory: Path, session: SessionData) -> None`, plus `get_module_event_codes() -> dict[tuple[int, int], tuple[int, ...]]` and `get_eligible_modules(session: SessionData) -> set[tuple[int, int]]` |
| `runtime.py`            | `RUNTIME_SOURCE_ID: str`, which locates the `{source_id}_log.npz` archive, and `parse_runtime(decoded_messages: pl.DataFrame, output_directory: Path, session: SessionData) -> None`                                                                                                                                        |
| `video_tracking.py`     | `locate_<system>_pose_predictions(session: SessionData) -> Path \| None` and `process_<system>_video_tracking(session: SessionData, output_directory: Path) -> None`. A system that tracks nothing donates a locator returning `None` and a no-op function                                                                  |
| `two_photon.py`         | `locate_two_photon_data(session: SessionData) -> Path`, `resolve_single_recording_configuration(session) -> SingleRecordingConfiguration`, `resolve_multi_recording_configuration(session) -> MultiRecordingConfiguration \| None`, and `<SYSTEM>_MULTI_RECORDING_SESSION_TYPES: frozenset[SessionTypes]`                   |
| `forging.py`            | `assemble_<system>_session(source_session_path: Path, output_path: Path, dataset_name: str) -> None` and `<SYSTEM>_ADMISSION_PIPELINES: dict[SessionTypes, frozenset[ProcessingPipelines]]`                                                                                                                                 |
| `metadata.py`           | The system's dataset-column `StrEnum`, its private per-column description mapping, and `<SYSTEM>_COLUMN_DESCRIPTIONS: dict[str, str]` derived from both                                                                                                                                                                     |
| Per-sub-dataset modules | The assemblers `<system>/forging.py` routes to. These face no registry, so their shape is the system's own                                                                                                                                                                                                                  |

`/data-processing-design` owns what each donation computes. This recipe owns only the wiring.

### Step 2: export from `<system>/__init__.py`

Every symbol `registries.py` will import appears in that package's `__all__`. The registered system's `__init__.py`
is the template, and its export list is exactly the import block `registries.py` carries.

### Step 3: register in `registries.py`

Add the import block for the new package, then add one entry to each of the eleven registries.
`_MICROCONTROLLER_PARSER_REGISTRY` takes one entry per parsed hardware module, keyed
`(AcquisitionSystems.<SYSTEM>, module_type, module_id)`. Every other registry takes exactly one entry keyed on the
member. `_FORGING_ASSEMBLY_REGISTRY` takes a `_ForgingAssemblyAsset(assembler=..., column_descriptions=...)` and
`_CINDRA_CONFIGURATION_REGISTRY` takes a
`_CindraConfigurationAsset(resolve_single_recording=..., resolve_multi_recording=...)`.

### Step 4: change nothing under `orchestration/` or `interfaces/`

Every consumer resolves the system from `SessionData.acquisition_system` or `DatasetData.acquisition_system` at the
call site, and the `slf_cli` docstring in `interfaces/entry_points.py` states that no command takes a system selector.
A new system therefore adds no CLI option, no MCP parameter, and no dispatch entry.

### Cross-repository coordination

- `sollertia-shared-assets` is blocking and lands first, through `assets:library-extension`.
- `sollertia-experiment` must write the artifacts the donations read, which is a `SystemConfiguration` subclass, its
  binding classes, a runtime controller written from scratch, an `interfaces/<system>.py` CLI group added to
  `_register_subcommands`, and an `interfaces/<system>_tools.py` module. That library runs no import-time check, so a
  half-wired system there surfaces only when an operator runs its configure command. Hand off to
  `experiment:library-extension`, and to `experiment:pipeline` for the acquisition run that produces the first
  processable session.
- `cindra` needs nothing when the system reuses `SingleRecordingConfiguration`, `MultiRecordingConfiguration`,
  `SingleRecordingJobNames`, and `MultiRecordingJobNames`. A system needing a new cindra stage needs a cindra release
  first, then a `_MULTIDAY_JOB_NAMES` entry in `forging/pipeline.py` or a `_JOB_CORE_ALLOCATIONS` entry in
  `orchestration/dispatch.py`.
- The `sollertia` marketplace gains a `plugins/<system>/` companion plugin and its `marketplace.json` entry, which
  `assets:library-extension` and `experiment:library-extension` both own. No skill in the forging plugin gains a
  per-system section, since every one of them is acquisition-system-agnostic.

---

## Adding a new session type

### Upstream first

The `SessionTypes` member, its descriptor dataclass declaring `incomplete`, its `DESCRIPTOR_REGISTRY` entry, and its
`SYSTEM_SESSION_TYPES` pairing all land in `sollertia-shared-assets` through `assets:library-extension`. A type no
system claims fails the upstream check before this library is reached.

### Ordered touch list

| #   | Touch point                 | File and identifier                                                                                                                                                                         | Covered by an import check                                                                                                             |
|-----|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Admission policy            | Add the `SessionTypes` key to `<SYSTEM>_ADMISSION_PIPELINES` in `<system>/forging.py`, mapping it to the `frozenset[ProcessingPipelines]` a session must complete before it joins a dataset | Partly. The check rejects a key outside `SYSTEM_SESSION_TYPES` and never flags an omitted key, since omission is the supported opt-out |
| 2   | Assembly routing            | Add the branch to `assemble_<system>_session` in `<system>/forging.py`, and write the sub-dataset assembler it calls                                                                        | No. An unrouted type raises at forging time                                                                                            |
| 3   | Cross-recording declaration | Add the type to `<SYSTEM>_MULTI_RECORDING_SESSION_TYPES` in `<system>/two_photon.py`, when the system tracks its animals across recordings for that type                                    | Partly. Only that the declared type appears in `SYSTEM_SESSION_TYPES`                                                                  |
| 4   | Multi-recording resolver    | Widen `resolve_multi_recording_configuration` in `<system>/two_photon.py` so it stops returning `None` for the new type                                                                     | No. The check validates the declared set against `SYSTEM_SESSION_TYPES` alone, never against the resolver                              |
| 5   | Dataset columns             | Give every new column a member in the system's dataset-column enum and an entry in the private description mapping, both in `<system>/metadata.py`                                          | Yes, but only as a bare `KeyError` raised by the `<SYSTEM>_COLUMN_DESCRIPTIONS` comprehension                                          |

Touch 1 and touch 3 are independent questions. Admission decides whether sessions of the type join a dataset at all,
and the cross-recording declaration decides whether the dataset tracks its animals across recordings.
`/dataset-definition` owns both as operating surfaces.

### Cross-repository coordination

- `sollertia-shared-assets` is blocking, through `assets:library-extension`.
- `sollertia-experiment` must run and record sessions of the new type before anything here can process one. Hand off
  to `experiment:library-extension` for the runtime mode and to `experiment:pipeline` for the acquisition run.
- `cindra` needs nothing.

---

## Adding a new processing stage

A stage rides its pipeline's existing routing. It needs no `ProcessingPipelines` member, no `BATCH_PIPELINES` change,
no `PipelineDispatch` entry, no CLI subcommand, and no MCP tool.

| #  | Touch point             | File and identifier                                                                                                                                                                                                            |
|----|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1  | Declare the job name    | A module-level `str` constant in `<category>/pipeline.py`, or the constant the dependency already exports. The current local constants are listed below the table                                                              |
| 2  | Export the name         | The `__all__` of `<category>/__init__.py`                                                                                                                                                                                      |
| 3  | Emit from job discovery | `discover_<category>_jobs(session_path)` returns the unit, the universe, and the possible subset. Append the stage to the universe unconditionally, and to the possible list only when the unit's data supports it             |
| 4  | Choose the specifier    | A per-source stage carries the source identifier, and a per-session stage carries the empty string. Module parse jobs use `f"{controller_id}-{module_type}-{module_id}"`                                                       |
| 5  | Order the prerequisites | `<category>_job_prerequisites(unit, universe)` returns `dict[tuple[str, str], tuple[tuple[str, str], ...]]`. An independent stage maps to `()`                                                                                 |
| 6  | Execute the stage       | Add a branch to the private dispatcher the pipeline entry point calls, following `_dispatch_job` in `video/pipeline.py`                                                                                                        |
| 7  | Declare cores           | `_JOB_CORE_ALLOCATIONS` in `orchestration/dispatch.py`                                                                                                                                                                         |
| 8  | Add a sizing model      | A `_size_*_job` function in `orchestration/footprints.py`, plus a routing branch inside `size_session_jobs` for a session pipeline or `size_dataset_jobs` for the dataset pipeline                                             |
| 9  | Declare a ceiling       | `_JOB_CONCURRENCY_LIMITS` in `orchestration/dispatch.py`, or the pipeline's own exported dict splatted into it, as `FORGING_JOB_CONCURRENCY_LIMITS` in `forging/pipeline.py` is. Only when throughput plateaus before cores do |
| 10 | Declare a reservation   | `_JOB_CONCURRENCY_RESERVATIONS` in `orchestration/dispatch.py`. Only when the type should hand capacity back                                                                                                                   |
| 11 | Report the stage        | `_PIPELINE_JOB_NAMES` in `interfaces/orchestration_tools.py`, whose own docstring states that a stage reaches the report once its name is listed there                                                                         |
| 12 | Optional CLI flag       | A `@click.option` on the pipeline's `slf process <name>` subcommand in `interfaces/process.py`, plus the matching keyword on `run_<category>_processing_pipeline`                                                              |

The job-name constants this library declares today:

- `ENERGY_JOB_NAME`, `RENAME_JOB_NAME`, and `TRACKING_JOB_NAME` in `video/pipeline.py`.
- `PARSE_JOB_NAME` in `microcontrollers/pipeline.py`.
- `RUNTIME_JOB_NAME` in `runtime/pipeline.py`.
- `CHECKSUM_JOB_NAME` in `managing/checksum.py`, and `MANIFEST_JOB_NAME` in `managing/manifest.py`.
- `MULTIDAY_DISCOVERY_JOB_NAME`, `MULTIDAY_EXTRACTION_JOB_NAME`, and `FORGING_JOB_NAME` in `forging/pipeline.py`.

### Current core allocations

`_JOB_CORE_ALLOCATIONS` in `orchestration/dispatch.py` keys every allocation by job name alone, so a new stage adds one
row here regardless of which pipeline owns it.

| Job name                              | Cores                                                             |
|---------------------------------------|-------------------------------------------------------------------|
| `checksum_resolution`                 | `8`                                                               |
| `runtime_processing`                  | `1`                                                               |
| `microcontroller_data_extraction`     | `CONTROLLER_EXTRACTION_JOB_CORES`, which is `4`                   |
| `module_parsing`                      | `1`                                                               |
| `camera_timestamp_extraction`         | `CAMERA_EXTRACTION_JOB_CORES`, which is `8`                       |
| `camera_timestamp_rename`             | `1`                                                               |
| `pose_tracking`                       | `1`                                                               |
| `motion_energy`                       | `16`                                                              |
| Each `SingleRecordingJobNames` member | `resolve_stage_workers(job_name=...)`, taken from cindra          |
| `multiday_discovery`                  | `resolve_stage_workers(job_name=MultiRecordingJobNames.DISCOVER)` |
| `multiday_extraction`                 | `resolve_stage_workers(job_name=MultiRecordingJobNames.EXTRACT)`  |
| `session_data_assembly`               | `1`                                                               |

`_JOB_CONCURRENCY_LIMITS` currently holds `checksum_resolution: 3`, `motion_energy: 3`, cindra's single-recording
ceilings, and `**FORGING_JOB_CONCURRENCY_LIMITS`. `_JOB_CONCURRENCY_RESERVATIONS` is derived entirely from cindra's
`RESOURCE_CLASS_BY_JOB_NAME`.

### Cross-repository coordination

- `sollertia-shared-assets` needs nothing. A stage shares its pipeline's existing tracker and output directory.
- `sollertia-experiment` is involved only when the stage reads an artifact acquisition does not yet write.
- A dependency is involved only when the stage is one of theirs, in which case reuse its exported job-name constant
  and its declared core and ceiling figures rather than inventing widths.

---

## Adding a new processing pipeline

### Upstream first

| Touch in `sollertia_shared_assets` | Identifier                                                                                                                                                                            |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `data_hierarchy/session_data.py`   | A new `ProcessingTrackers` member, whose value is the tracker filename                                                                                                                |
| `data_hierarchy/session_data.py`   | A new `ProcessedData` field pair, one output-directory field and one tracker field, both resolved in `ProcessedData.build`, plus the `Directories` member naming the output directory |

`shared_assets/pipelines.py` cannot resolve a tracker the upstream enum does not name, so this half is blocking and
lands through `assets:library-extension`. The three surfaces it adds to are documented by `assets:session-data`.

### Ordered touch list

| #   | Touch point                       | File and identifier                                                                                                                                                                                                                                                                                         |
|-----|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Pipeline identity                 | A new `ProcessingPipelines` member in `shared_assets/pipelines.py`. Its member name must mirror the upstream `ProcessingTrackers` member name, which that enum's own Notes state                                                                                                                            |
| 2   | Tracker location                  | A new entry in `_SESSION_TRACKER_LOCATIONS` in `shared_assets/pipelines.py`, for a per-session pipeline only. `SESSION_PIPELINES` is `tuple(_SESSION_TRACKER_LOCATIONS.keys())` and is derived, never edited. A project or dataset pipeline is deliberately absent from both                                |
| 3   | Manifest status column            | A new entry in `_PIPELINE_STATUS_COLUMNS` and a matching `pl.UInt8` column in `_PROJECT_MANIFEST_SCHEMA`, both in `managing/manifest.py`. Required for a per-session pipeline, and guarded by `_assert_status_column_coverage()`                                                                            |
| 4   | Category package                  | `src/sollertia_forgery/<category>/` with `pipeline.py` and `__init__.py`. The `__all__` exports the entry point, every stage job name, the discovery callable, and the prerequisite callable                                                                                                                |
| 5   | Batch membership                  | Add the member to `BATCH_PIPELINES` in `orchestration/dispatch.py`                                                                                                                                                                                                                                          |
| 6   | Dispatch entry                    | Add a `PipelineDispatch[UnitT]` entry in `_pipeline_dispatch()` in `orchestration/dispatch.py`. Both halves are required, since `_assert_dispatch_coverage` compares the two sets symmetrically                                                                                                             |
| 7   | Batch worker and command renderer | A private `_run_<name>_job(job: GenericPendingJob) -> None` and `_<name>_command(job) -> tuple[str, ...]` in `orchestration/dispatch.py`. A session pipeline's renderer reuses `_session_command_preamble`                                                                                                  |
| 8   | Cores and sizing                  | Every job type of the pipeline needs a `_JOB_CORE_ALLOCATIONS` entry and a sizing-model branch, per touches 7 and 8 of the stage recipe                                                                                                                                                                     |
| 9   | Resource-model report             | A `_PIPELINE_JOB_NAMES` entry in `interfaces/orchestration_tools.py`                                                                                                                                                                                                                                        |
| 10  | CLI                               | A new `@process_cli.command("<name>")` in `interfaces/process.py` for a pipeline that derives processed data from an acquired session, or a new top-level command registered in `_register_subcommands` in `interfaces/entry_points.py`, which is where `slf checksum`, `slf manifest`, and `slf forge` sit |
| 11  | MCP                               | Only when the pipeline needs a bespoke tool. Every batch tool validates `pipeline` against `{member.value for member in BATCH_PIPELINES}`, and `slf reset` and `slf clean` build their `click.Choice` from the same frozenset in `interfaces/manage.py`, so both surfaces auto-extend                       |
| 12  | Admission                         | Add the new member to `<SYSTEM>_ADMISSION_PIPELINES` in every `<system>/forging.py` whose sessions must complete it before forging                                                                                                                                                                          |

### The `PipelineDispatch` field contract

Every field of the frozen, slotted `PipelineDispatch[UnitT]` dataclass in `orchestration/dispatch.py` must be
supplied, and `prime` is the only one carrying a default.

| Field           | Type                                                                                           | What it supplies                                                                                        |
|-----------------|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `pipeline`      | `ProcessingPipelines`                                                                          | The member this entry dispatches                                                                        |
| `load`          | `Callable[[Path], UnitT]`                                                                      | `_load_session` or `_load_dataset`                                                                      |
| `discover`      | `Callable[[Path], tuple[UnitT, list[tuple[str, str]], list[tuple[str, str]]]]`                 | The unit, the job universe, and the possible subset                                                     |
| `worker`        | `Callable[..., None]`                                                                          | The picklable module-level worker the pool invokes with one planned job                                 |
| `prerequisites` | `Callable[[UnitT, list[tuple[str, str]]], dict[tuple[str, str], tuple[tuple[str, str], ...]]]` | Intra-pipeline ordering                                                                                 |
| `tracker_path`  | `Callable[[UnitT], Path]`                                                                      | `_session_tracker(pipeline=...)` or the pipeline's own tracker resolver                                 |
| `output_path`   | `Callable[[UnitT], Path \| None]`                                                              | The directory a cleanup may remove, or `None` when the pipeline writes into a directory it does not own |
| `unit_name`     | `Callable[[UnitT], str]`                                                                       | The unit's reported name                                                                                |
| `size_jobs`     | `Callable[[UnitT, list[tuple[str, str, int]]], dict[tuple[str, str], JobFootprint]]`           | `_session_sizer(pipeline=...)` or `size_dataset_jobs`                                                   |
| `command`       | `Callable[[GenericPendingJob], tuple[str, ...]]`                                               | The remote argv renderer                                                                                |
| `prime`         | `Callable[[Path], None] \| None`                                                               | An optional idempotent bootstrap run before `discover`                                                  |

### Cross-repository coordination

- `sollertia-shared-assets` is blocking, through `assets:library-extension`.
- `sollertia-experiment` is involved only when the pipeline reads an artifact acquisition does not yet write. Confirm
  the runtime writes it before the first batch runs, through `experiment:library-extension`.
- A dependency is involved only when the pipeline wraps one of its stages, in which case reuse its job-name enum and
  its resource figures.

---

## Adding an MCP tool

### The discovery seam

`_register_tool_modules()` in `interfaces/mcp_server.py` computes its package name, globs `*_tools.py` inside its own
directory in `sorted()` order, and imports every match. It is called at module scope immediately after its own
definition, so `@mcp.tool()` decorators register purely as an import side effect and `mcp_server.py` needs no edit for
a new module. The `_tools.py` suffix is load-bearing, and the call uses `glob` rather than `rglob`, so the module must
sit directly in `src/sollertia_forgery/interfaces/`.

### Tool contract

Import the shared instance as `from .mcp_instance import mcp`, which is `MCPServer(name="sollertia-forgery")`.
Decorate each function with `@mcp.tool()` and return `dict[str, Any]` built by `ok_response(**payload)` or
`error_response(message)` from `interfaces/responses.py`. The paging and projection helpers live in the same module
and are `resolve_page`, `page_fields`, `resolve_detail_limit`, `project_item`, `count_values`, `bounded_counts`,
`frame_breakdown`, `resolve_elapsed_seconds`, and `reject_unknown`. Document the response key shape in the tool's
`Returns:` docstring section, since that shape is the public contract.

### Coverage-omit obligation

`[tool.coverage.run]` in `pyproject.toml` sets `branch = true` and lists fifteen `interfaces/` modules in its `omit`
list one by one rather than by a directory glob, and `[tool.coverage.report]` sets `fail_under = 100`. A new
`<name>_tools.py` MUST be added to that list. `interfaces/mcp_instance.py`, `interfaces/responses.py`, and
`interfaces/__init__.py` are deliberately not omitted, so a new non-tool interface helper is either covered by tests
or added to the list deliberately.

### Cross-repository coordination

None. An MCP tool is local to sollertia-forgery, and the forging plugin's `.claude-plugin/plugin.json` registers the
server as `{"command": "slf", "args": ["mcp"]}` with no per-tool entry. Only the skill that owns the tool's domain
changes.

---

## Documentation, coverage, and test obligations

| Scenario                | `docs/source/api.rst`                                                                                                                                                                                                                  | Coverage                                                           | Tests                                                                                                                                                                                          |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| New acquisition system  | A `<System> Assets` section carrying `.. automodule:: sollertia_forgery.<system>`, one `.. autoclass::` per column enumeration, and one `.. autodata::` per exported constant, each naming the defining module rather than the package | Fully measured. The omit list covers `interfaces/` modules only    | A `tests/<system>/` package, plus the new registry names added to `_DONOR_REGISTRY_NAMES` in `tests/registry_coverage_test.py` and the system added to that file's parametrized coverage tests |
| New session type        | Nothing new, unless the type adds an exported constant                                                                                                                                                                                 | Fully measured                                                     | The assembly-routing branch, the multi-recording resolver's answer for the type, and the column-description entry                                                                              |
| New processing stage    | One `.. autodata::` for the job-name constant in the owning section, naming the defining module. A dependency's constant is named through that dependency's own module path                                                            | Fully measured                                                     | Discovery emission, prerequisite ordering, the dispatcher branch, the core allocation, and the sizing model                                                                                    |
| New processing pipeline | A new section with `.. automodule:: sollertia_forgery.<category>` plus one `.. autodata::` per exported job-name constant                                                                                                              | Fully measured                                                     | A `tests/<category>/` package mirroring the new source package                                                                                                                                 |
| New MCP tool            | Nothing. `interfaces/` carries no `automodule` section                                                                                                                                                                                 | The new `*_tools.py` module is added to `[tool.coverage.run] omit` | None required by the gate, since the module is omitted                                                                                                                                         |

`.. automodule::` runs on the package and skips a constant the package merely re-exports, which is why every constant
carries an explicit `.. autodata::` naming its defining module. The CLI section is a `.. click::` directive on
`sollertia_forgery.interfaces.entry_points:slf_cli` with `:nested: full`, so a new command appears without an edit.

`tests/` mirrors `src/sollertia_forgery/` one directory per package. A test that spawns a process pool or mutates
process-wide state carries `@pytest.mark.xdist_group`, because the suite runs under `pytest-xdist --dist loadgroup`.
The fixtures in `tests/conftest.py` are all shaped by the one registered acquisition system today, so a new system
adds its own rather than reusing them. `[tool.hatch.build.targets.wheel]` names `src/sollertia_forgery`, so a new
subpackage needs no build-configuration edit.
