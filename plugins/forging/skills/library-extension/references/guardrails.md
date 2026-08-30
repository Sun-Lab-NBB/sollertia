# Extension guardrails

Documents the import-time checks that guard the `sollertia-forgery` registries, the dispatch table, and the project
manifest. It carries the verbatim text of every failure they raise, the touch each message names, and the extension
steps no check covers. The per-scenario touch lists that carry those steps are in
[extension-recipes.md](extension-recipes.md).

Every failure is raised through `ataraxis_base_utilities.console.error`, so it stops the import at the first offender
rather than accumulating a report.

---

## Where each check runs

| Check                              | Module                      | Import that runs it                                                   | What it guards                                      |
|------------------------------------|-----------------------------|-----------------------------------------------------------------------|-----------------------------------------------------|
| `_assert_registry_coverage()`      | `registries.py`             | `sollertia_forgery.registries`, and every module that reaches it      | The thirteen donor registries                       |
| `_assert_dispatch_coverage()`      | `orchestration/dispatch.py` | `sollertia_forgery.orchestration`                                     | The batch dispatch table                            |
| `_assert_status_column_coverage()` | `managing/manifest.py`      | `sollertia_forgery.managing`                                          | The manifest's per-pipeline status columns          |
| A system package's own checks      | `<system>/`                 | `sollertia_forgery.registries`, since it imports every system package | Whatever privately keyed table that system declares |

`import sollertia_forgery` on its own runs none of them, because the top-level `__init__.py` re-exports no library
symbol. `slf --help` runs all three, since `interfaces/entry_points.py` imports `interfaces/manage.py`, which imports
both `..managing` and `..orchestration`.

---

## The donor-registry coverage check

`_assert_registry_coverage()` is the last statement of `registries.py`. It runs four checks in a fixed order and
raises on the first offender, so an extender fixes one gap, re-imports, and reads the next.

### Check 1, a system missing from a registry

The loop iterates a literal tuple of thirteen `(registry_name, registered_systems)` pairs and raises on the first whose
`frozenset(AcquisitionSystems) - registered_systems` is non-empty. The order decides which registry the error names.

| Order | Registry named in the message                                                                                       |
|-------|---------------------------------------------------------------------------------------------------------------------|
| 1     | `_FORGING_ASSEMBLY_REGISTRY`                                                                                        |
| 2     | `_ASSEMBLY_GEOMETRY_REGISTRY`                                                                                       |
| 3     | `_ASSEMBLY_SOURCE_REGISTRY`                                                                                         |
| 4     | `_RUNTIME_PARSER_REGISTRY`                                                                                          |
| 5     | `_TWO_PHOTON_DATA_REGISTRY`                                                                                         |
| 6     | `_VIDEO_TRACKING_REGISTRY`                                                                                          |
| 7     | `_POSE_PREDICTION_REGISTRY`                                                                                         |
| 8     | `_MICROCONTROLLER_EVENT_CODE_REGISTRY`                                                                              |
| 9     | `_MICROCONTROLLER_ELIGIBILITY_REGISTRY`                                                                             |
| 10    | `_CINDRA_CONFIGURATION_REGISTRY`                                                                                    |
| 11    | `_MULTI_RECORDING_SESSION_TYPE_REGISTRY`                                                                            |
| 12    | `_FORGING_ADMISSION_REGISTRY`                                                                                       |
| 13    | `_MICROCONTROLLER_PARSER_REGISTRY`, reached through `frozenset(system for system, _, _ in ...)` on its 3-tuple keys |

```text
Unable to validate donor-registry coverage for {registry_name}. Every acquisition system must register its donated processing and forging assets in this module ('registries.py'), but entries are missing for {missing_names}.
```

`missing_names` is `", ".join(sorted(member.name for member in missing_systems))`, so it reports enum member names.
The thirteenth row is how "at least one parser per system" is enforced without a separate check.

### Check 2, a parseable module that declares no event codes

For each system that donates a parser, the check subtracts the keys the registered event-code accessor returns from
the `(module_type, module_id)` pairs the parser registry carries for that system. It calls the accessor rather than
reading it, since the donation is a zero-argument callable.

```text
Unable to validate donor-registry coverage for _MICROCONTROLLER_EVENT_CODE_REGISTRY. Every module registered in _MICROCONTROLLER_PARSER_REGISTRY must also declare the event codes its parser reads, but {target_system.name} does not declare codes for the following modules: {module_names}.
```

`module_names` renders each pair as `(module_type, module_id)`. The gap matters because the extraction stage filters
each module by the codes this registry resolves, so a parseable module absent from it leaves its parse job
undiscovered.

### Check 3, a cross-recording type the system does not record

Subtracts `SYSTEM_SESSION_TYPES[target_system]`, imported from `sollertia_shared_assets`, from the frozenset the
system declared in `_MULTI_RECORDING_SESSION_TYPE_REGISTRY`.

```text
Unable to validate donor-registry coverage for _MULTI_RECORDING_SESSION_TYPE_REGISTRY. Every session type a system tracks across recordings must be a session type that system records, but {target_system.name} declares the following unrecorded type(s): {type_names}.
```

### Check 4, an admitted type the system does not record

Subtracts the same upstream set from the keys of the system's `_FORGING_ADMISSION_REGISTRY` mapping.

```text
Unable to validate donor-registry coverage for _FORGING_ADMISSION_REGISTRY. Every session type a system admits into a forged dataset must be a session type that system records, but {target_system.name} declares admission requirements for the following unrecorded type(s): {type_names}.
```

`type_names` joins `session_type.value` in both checks 3 and 4, which is the opposite of checks 1 and 2. Reading a
message for the wrong one of the two spellings is the usual reason an extender cannot find the offending entry.

### Two properties that decide how far to trust the check

**The admission relation is deliberately asymmetric.** Declaring a type the system does not record fails. Omitting a
type the system does record is the supported opt-out and means the type joins no dataset, which is indistinguishable
from an accidental omission.

**Coverage is a check on wiring, not on capability.** A system that produces none of a data class still donates an entry
for it. A no-op tracking function, a locator returning `None`, an empty `frozenset[SessionTypes]`, and a two-photon
locator returning the path the system would use each satisfy the check.

---

## The dispatch-table check

`_assert_dispatch_coverage()` compares `frozenset(_pipeline_dispatch())` against `BATCH_PIPELINES` symmetrically, so
it catches a member with no entry and an entry naming a non-member alike.

```text
Unable to validate the pipeline dispatch table. Every pipeline named in BATCH_PIPELINES must have a dispatch entry and no entry may name a pipeline outside it, but the sets differ by {sorted(member.value for member in entries ^ BATCH_PIPELINES)}.
```

---

## The manifest status-column check

`_assert_status_column_coverage()` compares `frozenset(_PIPELINE_STATUS_COLUMNS)` against
`frozenset(SESSION_PIPELINES)`, also symmetrically. It fires the moment `managing/manifest.py` loads rather than
partway through a generation pass over a project.

```text
Unable to validate the manifest's pipeline status columns. Every pipeline in SESSION_PIPELINES must declare a status column and no column may name a pipeline outside it, but the sets differ by {sorted(member.value for member in declared ^ carried)}.
```

The column name may differ from the pipeline value, so the mapping's values are not derivable from its keys. The
matching `pl.UInt8` column in `_PROJECT_MANIFEST_SCHEMA` is a separate touch that this check does not reach.

---

## Checks a system package authors for itself

A `<system>/` package may declare its own module-scope coverage check over a privately keyed table it owns, and that
check runs whenever `registries.py` imports the package. Author one when the system carries a private mapping whose
keys must stay paired with an enum it also declares.

The column-description mapping is the one such guard the pattern already requires. Building
`<SYSTEM>_COLUMN_DESCRIPTIONS` as a comprehension over the system's dataset-column enum raises a bare `KeyError` at
import when a column carries no description. It has no message and no named assertion function, so a failure with an
empty `KeyError` and a `<system>/metadata.py` frame is that omission and nothing else.

---

## Runtime errors that stand in for an absent import check

Each of these fires while a pipeline runs rather than at import, and each names exactly one missing touch.

| Function                          | Module                              | Touch it names                                                                                                     |
|-----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `_resolve_system`                 | `registries.py`                     | The caller passed a system outside `AcquisitionSystems`, so the message lists the valid values                     |
| `resolve_job_cores`               | `orchestration/dispatch.py`         | The `_JOB_CORE_ALLOCATIONS` entry for a job type a pipeline resolves                                               |
| `size_session_jobs`               | `orchestration/footprints.py`       | The sizing model and its routing branch, for a session pipeline's job type                                         |
| `size_dataset_jobs`               | `orchestration/footprints.py`       | The sizing model and its routing branch, for the dataset pipeline's job type                                       |
| `run_batch_job`                   | `orchestration/dispatch.py`         | The dispatch entry for the pipeline a job names                                                                    |
| `resolve_job_command`             | `orchestration/dispatch.py`         | The command renderer for the pipeline a job names                                                                  |
| `resolve_session_tracker_path`    | `shared_assets/pipelines.py`        | The `_SESSION_TRACKER_LOCATIONS` entry, or a caller asking a project or dataset pipeline for a per-session tracker |
| The pipeline's private dispatcher | `<category>/pipeline.py`            | The execution branch for a stage the pipeline discovered                                                           |
| `verify_session_admissibility`    | `forging/admission.py`              | The admission entry for a session type, or a pipeline the session has not completed                                |
| `assemble_<system>_session`       | `<system>/forging.py`               | The assembly-routing branch for a session type                                                                     |
| `_unsupported_message`            | `interfaces/processing_tools.py`    | The `BATCH_PIPELINES` membership of a pipeline an MCP caller named                                                 |
| `read_resource_model_tool`        | `interfaces/orchestration_tools.py` | The `_PIPELINE_JOB_NAMES` entry for a pipeline or job type a caller named                                          |

The two sizing refusals share one wording, which states that a job whose resources nothing resolves cannot be
admitted to a batch. The core-allocation refusal states that every job type a pipeline resolves must declare its
cores in `_JOB_CORE_ALLOCATIONS`.

---

## What no check covers

Everything below passes every import and fails later, or silently. Each one is covered by a test rather than by a
guardrail.

| Uncovered touch point                                                 | Scenario        | How the omission surfaces                                                                                    | Where to cover it                              |
|-----------------------------------------------------------------------|-----------------|--------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| The assembly-routing branch for a session type                        | Session type    | `ValueError` when the forging pipeline reaches a session of that type                                        | The system package's forging tests             |
| A type declared cross-recording whose resolver still returns `None`   | Session type    | Silent. Dataset definition writes no multi-recording configuration and the cross-recording jobs never appear | The system package's two-photon tests          |
| A recorded session type omitted from the admission mapping            | Session type    | Silent by design, since omission is the opt-out                                                              | The system package's admission tests           |
| A dataset column with no description entry                            | Session type    | A bare `KeyError` at import of the system's metadata module, with no message                                 | The system package's metadata tests            |
| The job emitted from discovery, and its prerequisite ordering         | Stage           | Silent. The job is never planned, or it runs out of order                                                    | The category package's discovery tests         |
| The `_JOB_CORE_ALLOCATIONS` entry                                     | Stage, pipeline | `ValueError` during preparation of the unit that resolves the job type                                       | The orchestration dispatch tests               |
| The sizing model and its routing branch                               | Stage, pipeline | `ValueError` during the sizing pass                                                                          | The orchestration footprint tests              |
| The execution branch inside the pipeline entry point                  | Stage           | `ValueError` when the job identifier reaches the dispatcher                                                  | The category package's pipeline tests          |
| The `_PIPELINE_JOB_NAMES` entry                                       | Stage, pipeline | Silent. `read_resource_model_tool` never reports the job type                                                | The interface tests, or a manual tool call     |
| The CLI stage flag or the `slf process` subcommand                    | Stage, pipeline | The command simply does not exist                                                                            | A manual `slf --help` pass                     |
| The `_PROJECT_MANIFEST_SCHEMA` column beside a declared status column | Pipeline        | The manifest frame rejects the row, since the schema names no such column                                    | The manifest tests                             |
| A tool module misnamed or nested below `interfaces/`                  | MCP tool        | The glob never imports it and the tools silently do not exist                                                | A manual tool listing against a started server |
| A new `*_tools.py` missing from the coverage omit list                | MCP tool        | `tox -e coverage` fails the 100 percent gate                                                                 | The gate itself                                |
| A system package importing a category package                         | System          | A circular `ImportError` at import. Nothing else checks the layering                                         | The import gate itself                         |
| A donated worker that is not a picklable module-level function        | System          | The pool fails at pickling time when the stage first dispatches                                              | The system package's worker tests              |
