---
name: library-extension
description: >-
  Owns the extension path of sollertia-forgery: adding an acquisition system, a session type, a processing stage, a
  processing pipeline, or an MCP tool. Covers the thirteen donor registries, the three import-time coverage checks and
  their verbatim errors, the touch points no check reaches, and the cross-repository ordering the upstream libraries
  impose. Use when wiring a new acquisition system into the registries, adding a processing stage or pipeline, adding an
  slf mcp tool, or when an import-time RuntimeError names a registry, a dispatch table, or a status column.
user-invocable: false
---

# Sollertia forgery library extension

Extends `sollertia-forgery` along the seams by which an acquisition system, a session type, a processing stage, a
processing pipeline, and an MCP tool enter it. Every system-specific behavior enters this library through
`registries.py` alone, so extending it is a wiring exercise, and an import error names the work that remains. The file
set of a forged dataset is the one seam a donation does not reach.

You MUST read this entire skill before extending the library, then read
[references/extension-recipes.md](references/extension-recipes.md) for the scenario you are applying and
[references/guardrails.md](references/guardrails.md) for the checks that verify it. You MUST run the verification
checklist before reporting an extension complete.

---

## Scope

**Covers:**
- Adding an acquisition system, which is one `<system>/` package plus an entry in each of the thirteen donor registries
- Adding a session type's forging half, which is the admission policy, the assembly routing, the assembly-source
  routing, the cross-recording declaration, the multi-recording resolver, and the dataset columns
- Adding a processing stage to a pipeline that already exists
- Minting or joining a per-system registry for a stage whose input only an acquisition system can supply
- Adding a processing pipeline, which is a new `ProcessingPipelines` member and a new category package
- Adding an MCP tool module to the `slf mcp` server
- The three import-time coverage checks, their verbatim errors, and the touch points no check reaches
- The dataset file set fixed in the agnostic layer, which no donation widens
- The cross-repository ordering the upstream libraries impose, and the handoff each scenario carries
- The documentation, coverage, and test obligations each scenario carries

**Does not cover:**
- What a donated worker computes, and how a per-system processing stage is designed. Owned by `/data-processing-design`.
- Operating the admission policy and the column descriptions once they are wired. Owned by `/dataset-definition`.
- The upstream `AcquisitionSystems` and `SessionTypes` members and the `SYSTEM_SESSION_TYPES` pairing. Owned by
  `assets:library-extension`.
- The `ProcessingTrackers` members, the `Directories` members, and the `ProcessedData` fields a new pipeline needs.
  Owned by `assets:library-extension`, under "Adding the session-record surfaces of a new processing pipeline" in its
  `references/extension-recipes.md`, and documented by `assets:session-data`.
- The acquisition runtime that records the artifacts a donation reads. Owned by `experiment:library-extension`.
- The concrete donations the one registered acquisition system makes. Owned by the `mesoscope:mesoscope-vr-*` skill
  family.
- The `slf mcp` server a new tool module joins, and the response contract that module satisfies. Owned by
  `/forging-mcp-environment-setup`.
- Running the pipelines an extension adds. Owned by `/pipeline` and `/batch-processing`.

---

## What each repository owns

An extension spans up to four repositories, and this library sits at the downstream end of the chain.

| Repository                                                            | What it owns for an extension                                                                                                                                                     | Owning skill                     |
|-----------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------|
| `sollertia-shared-assets`                                             | The `AcquisitionSystems` and `SessionTypes` members, the `SYSTEM_SESSION_TYPES` pairing, the `ProcessingTrackers` filenames, and the `ProcessedData` directory and tracker fields | `assets:library-extension`       |
| `sollertia-experiment`                                                | The acquisition runtime that records the artifacts every donation reads                                                                                                           | `experiment:library-extension`   |
| `sollertia-forgery`                                                   | The thirteen donor registries, the category packages, the dispatch table, the resource model, the `slf` CLI, and the MCP tools                                                    | This skill                       |
| `cindra`, `ataraxis-video-system`, `ataraxis-communication-interface` | The stages this library delegates in-process, and the job-name constants and resource figures a wrapper reuses rather than mints                                                  | The owning library's maintainers |

---

## The registry model

`registries.py` holds thirteen module-private registries, twelve of them keyed on an `AcquisitionSystems` member and one
keyed on a `(system, module_type, module_id)` triplet. Each maps a system to the callables and data it donates for one
processing concern. The module's `__all__` exports the two public donation Protocols, `ForgingAssembler` and
`MicrocontrollerParser`, and the fifteen `resolve_*` accessors, and exports no registry. Eight donation Protocols are
declared in total, and two registries hold no callable at all.

| Registry                                 | What a system donates                                                                                   |
|------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `_MICROCONTROLLER_PARSER_REGISTRY`       | One `MicrocontrollerParser` per parsed hardware module, keyed `(system, module_type, module_id)`        |
| `_MICROCONTROLLER_EVENT_CODE_REGISTRY`   | A zero-argument accessor returning `{(module_type, module_id): (code, ...)}`                            |
| `_MICROCONTROLLER_ELIGIBILITY_REGISTRY`  | `(session: SessionData) -> set[tuple[int, int]]`                                                        |
| `_RUNTIME_PARSER_REGISTRY`               | `(source_id, parser)`, where the source id locates the runtime log archive                              |
| `_POSE_PREDICTION_REGISTRY`              | `(session: SessionData) -> Path \| None`                                                                |
| `_VIDEO_TRACKING_REGISTRY`               | `(session: SessionData, output_directory: Path) -> None`                                                |
| `_TWO_PHOTON_DATA_REGISTRY`              | `(session: SessionData) -> Path`                                                                        |
| `_CINDRA_CONFIGURATION_REGISTRY`         | A `_CindraConfigurationAsset` bundling the single-recording and multi-recording configuration resolvers |
| `_FORGING_ASSEMBLY_REGISTRY`             | A `_ForgingAssemblyAsset` bundling the per-session `assembler` and its `column_descriptions` mapping    |
| `_ASSEMBLY_GEOMETRY_REGISTRY`            | `(session: SessionData) -> AssemblyGeometry`, the heights its assembler holds a frame and its sources   |
| `_ASSEMBLY_SOURCE_REGISTRY`              | `(session: SessionData) -> tuple[int, ...]`, the height its assembler holds each source it reads        |
| `_FORGING_ADMISSION_REGISTRY`            | `dict[SessionTypes, frozenset[ProcessingPipelines]]`, data rather than a callable                       |
| `_MULTI_RECORDING_SESSION_TYPE_REGISTRY` | `frozenset[SessionTypes]`, data rather than a callable                                                  |

The key shape of each registry, and the fifteen `resolve_*` accessors through which a pipeline reads them, are
documented by `/data-processing-design`.

Three rules govern every entry in that table.

**Coverage is a check on wiring, not on capability.** A system that produces none of a data class still donates an entry
for it. That entry is a no-op tracking function, a locator returning `None`, an empty frozenset, or a two-photon locator
returning the path the system would use. The video-tracking, pose-prediction, two-photon, and multi-recording registries
state that null shape in their own docstrings.

**The accessor is the API and the dict is an implementation detail.** No category package imports a registry constant,
and each imports the matching accessor from `..registries` instead. `_resolve_system` normalizes a `str` or an
`AcquisitionSystems` member behind every accessor, so a system resolves to the same asset whichever spelling reaches
it, and an unknown system raises a named `ValueError` rather than a bare `KeyError`.

**A system package never imports a category package.** The arrows already run category package to `registries.py` to
system package, so the reverse import is a cycle rather than a style preference. The one legal upward import is
`..shared_assets`, which imports nothing from this library. Nothing enforces the rule, so the circular `ImportError`
is the enforcement.

Minting a fourteenth registry is a separate act from adding an entry to the thirteen above. A stage whose input only an
acquisition system can supply joins the registry that already names its concern, or mints one. Minting is a donation
Protocol, a module-private dict, a `resolve_*` accessor, its `__all__` export, and its name in the tuple
`_assert_registry_coverage()` iterates. That last touch is the one nothing else implies. A registry outside the tuple
admits a system with no entry, so the seam raises a bare `KeyError` from its accessor at runtime rather than a named
`RuntimeError` at import. The ordered touch list of each branch is under "Minting or joining a per-system registry" in
[references/extension-recipes.md](references/extension-recipes.md). `_POSE_PREDICTION_REGISTRY` is the worked pattern
there for every touch, including its entry in the coverage test.

The Mesoscope-VR donations that fill these seams are documented by the `mesoscope:mesoscope-vr-*` skill family, one
member per seam group. `forging:data-processing-design` carries the registry-to-skill map, and the related-skills table
below repeats it as routing.

---

## No command takes a system selector

Nothing under `orchestration/` or `interfaces/` changes when an acquisition system is added. Every consumer resolves the
system from `SessionData.acquisition_system` or `DatasetData.acquisition_system` at the call site, and the `slf_cli`
docstring in `interfaces/entry_points.py` states that as a property of the whole command surface. A new system therefore
adds no CLI option, no MCP parameter, no dispatch entry, and no sizing branch, and every skill in this plugin stays
acquisition-system-agnostic across the change.

---

## What a donation cannot widen

**Autonomy boundary.** The file set of a forged dataset is fixed in the agnostic layer, and no registry reaches it. That
closure is deliberate, because a forged dataset is read by consumers that know no acquisition system, so its file set is
a platform contract rather than a per-system choice. `_forge_session` in `forging/pipeline.py` re-exports exactly the
raw assets its `reexported_assets` mapping names, and `DatasetFiles` in
`sollertia_shared_assets.data_hierarchy.dataset_data` declares exactly `DATA` and `DESCRIPTIONS`. Folding a new
per-session artifact into `data.feather` columns through the assembly worker that every system already donates is
agent-ownable, and you complete it autonomously. Widening the file set itself has no recipe, because it edits
`reexported_assets` and `DatasetFiles` across two repositories and reaches every dataset already forged against the
current set. Neither routing the artifact around the seam nor minting a per-system copy of either symbol substitutes for
that change. Escalate those to the human supervisor and co-design them in a generative, collaborative mode. What is
missing there is a platform-contract decision that binds consumers outside this library, rather than capability, so the
work must be human-supervised.

---

## Import-time guardrails

Three module-scope checks guard this library, and each raises a `RuntimeError` through
`ataraxis_base_utilities.console.error`, which stops the import at the first offender.

| Check                              | Module                      | Import that runs it                                        | What it guards                                                                                                                                              |
|------------------------------------|-----------------------------|------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `_assert_registry_coverage()`      | `registries.py`             | `sollertia_forgery.registries`, and everything reaching it | Every system's entry in each of the thirteen registries, every parseable module's event codes, and every declared session type against the upstream pairing |
| `_assert_dispatch_coverage()`      | `orchestration/dispatch.py` | `sollertia_forgery.orchestration`                          | The dispatch table against `BATCH_PIPELINES`, symmetrically, and every entry's `unit_kind` against the two this library resolves                            |
| `_assert_status_column_coverage()` | `managing/manifest.py`      | `sollertia_forgery.managing`                               | `_PIPELINE_STATUS_COLUMNS` against `SESSION_PIPELINES`, and every roster that names a status column, all symmetrically                                      |

`import sollertia_forgery` alone runs none of them, because the top-level `__init__.py` re-exports no library symbol
and its `__all__` is empty. `slf --help` runs all three, since `interfaces/entry_points.py` imports
`interfaces/manage.py`, which imports both `..managing` and `..orchestration`.

**The error is the remaining checklist.** The coverage check reports one problem per import attempt, so an extender
fixes the registry it names, re-imports, and reads the next. The order is fixed, which makes the sequence of errors a
worklist rather than a surprise. [references/guardrails.md](references/guardrails.md) carries the verbatim text of
every message and maps each onto the touch it names. The three checks raise nine stems between them, four from the
registries, two from the dispatch table, and three from the manifest:

```text
Unable to validate donor-registry coverage for <REGISTRY>. Every acquisition system must register ...
Unable to validate donor-registry coverage for _MICROCONTROLLER_EVENT_CODE_REGISTRY. Every module registered ...
Unable to validate donor-registry coverage for _MULTI_RECORDING_SESSION_TYPE_REGISTRY. Every session type ...
Unable to validate donor-registry coverage for _FORGING_ADMISSION_REGISTRY. Every session type a system admits ...
Unable to validate the pipeline dispatch table. Every pipeline named in BATCH_PIPELINES must have ...
Unable to validate the pipeline dispatch table. Every entry must declare one of ['dataset', 'session'] as ...
Unable to validate the manifest's pipeline status columns. Every pipeline in SESSION_PIPELINES must declare ...
Unable to validate the manifest's pipeline status columns. Every status column declared in ...
Unable to validate the manifest's column rosters. Every roster that lists the manifest's columns must ...
```

A message reporting a system interpolates the enum member name, and a message reporting a session type interpolates the
enum value, so a search for the offending entry uses the spelling carried by the message that fired.

### What the checks do not catch

The three checks cover registry membership, the dispatch table, and the manifest columns together with every roster that
names one. Seventeen further touch points reach no guardrail, so tests rather than a check cover each one. All but two
pass every import and fail later or silently. A dataset column with no description entry raises a bare `KeyError` with
no message at import of the system's own metadata module, and a system package importing a category package raises a
circular `ImportError` at import. One is silent by design, because omitting a recorded session type from the admission
mapping is the supported opt-out. [references/guardrails.md](references/guardrails.md) lists all seventeen, the scenario
that owns each one, how it surfaces, and where to cover it.

---

## Extension scenarios

Pick exactly one row and apply its recipe. A change spanning several scenarios applies their recipes sequentially
rather than interleaved, because the coverage check reports one structure at a time.

| Scenario                | Blocking upstream half                                                               | Recipe                                                                                          |
|-------------------------|--------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| New acquisition system  | `AcquisitionSystems` member, per-system record classes, `SYSTEM_SESSION_TYPES` claim | [Acquisition system](references/extension-recipes.md#adding-a-new-acquisition-system)           |
| New session type        | `SessionTypes` member, its descriptor, its `SYSTEM_SESSION_TYPES` pairing            | [Session type](references/extension-recipes.md#adding-a-new-session-type)                       |
| New processing stage    | None                                                                                 | [Processing stage](references/extension-recipes.md#adding-a-new-processing-stage)               |
| Per-system registry     | None                                                                                 | [Per-system registry](references/extension-recipes.md#minting-or-joining-a-per-system-registry) |
| New processing pipeline | `ProcessingTrackers` member, `ProcessedData` field pair, `Directories` member        | [Processing pipeline](references/extension-recipes.md#adding-a-new-processing-pipeline)         |
| New MCP tool            | None                                                                                 | [MCP tool](references/extension-recipes.md#adding-an-mcp-tool)                                  |

A new MCP tool module registers itself, since `_register_tool_modules()` in `interfaces/mcp_server.py` globs
`*_tools.py` inside its own directory and imports every match.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## Cross-repository ordering

Ordering is the half of an extension an agent gets wrong. The rule comes from the "Companion library synchronization"
section of `sollertia-forgery/CLAUDE.md`, and it runs strictly upstream to downstream.

1. **The shared-assets change lands first.** The `AcquisitionSystems` member, the `SessionTypes` members, the
   `SYSTEM_SESSION_TYPES` pairing, the `ProcessingTrackers` filenames, and the `ProcessedData` directory and tracker
   fields are all owned upstream. Hand off to `assets:library-extension` and wait for its checklist to complete.
2. **The moment an acquisition-system member lands upstream, this library stops importing.** Its coverage check
   measures against `frozenset(AcquisitionSystems)`, so every import path reaching `registries.py` fails until the
   thirteen donations are wired. That break is expected and is the checklist for step 3.
3. **Wire the donations here.** Apply the recipe for the scenario until the import gates in the verification
   checklist below all pass.
4. **The acquisition side must record the data before anything here can process it.** A donation reads artifacts the
   acquisition runtime writes, so a system or a session type is not processable until `sollertia-experiment` runs it.
   That library carries no import-time check, so this handoff is verified by hand. Hand off to
   `experiment:library-extension` for the runtime, and to `experiment:pipeline` for the acquisition run that produces
   the first processable session.
5. **A dependency's release lands before a stage that wraps it.** A stage backed by a cindra, video-system, or
   communication-interface binding needs that library released first, then reuses its exported job-name constant and
   its declared resource figures rather than minting local ones.

The gate between steps 1 and 3 is that `python -c "import sollertia_shared_assets"` succeeds and the new member
appears in `assets:library-extension`'s introspection tools. The gate between steps 3 and 4 is that `slf --help`
prints, which is what runs all three of this library's checks.

---

## Cross-skill touch table

Each recipe names the handoffs its scenario carries. This table is the inverse view, and a review of a finished
extension checks against it.

| Skill                                      | Touched by                                    | What changes                                                                                                                                 |
|--------------------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `/data-processing-design`                  | New system, new stage                         | The per-system donation design behind a registry entry, and the design of what a new stage computes                                          |
| `/dataset-definition`                      | New system, new session type                  | The admission policy a session satisfies before it joins a dataset, and the column descriptions a dataset records                            |
| `/dataset-forging`                         | New system, new session type                  | The reach of the forging pipeline over a system's session types                                                                              |
| `/batch-processing`                        | New pipeline, new stage                       | The per-pipeline table, and the stages a pipeline dispatches                                                                                 |
| `/job-planning`                            | New pipeline, new stage                       | The resource model, since a job type reaches the report only through `_PIPELINE_JOB_NAMES`                                                   |
| `/project-state`                           | New per-session pipeline                      | The manifest status column that pipeline adds, and the five rosters that name it                                                             |
| `/processing-input-format`                 | New pipeline, new system                      | The acquired artifacts a pipeline requires before it runs                                                                                    |
| `/processing-results`                      | New pipeline, new stage                       | The outputs a stage writes and the directory that owns them                                                                                  |
| `/cli-reference`                           | New pipeline, new stage flag                  | The `slf` command surface and its option roster                                                                                              |
| `/pipeline`                                | New pipeline                                  | The phase map and the routing into the new pipeline                                                                                          |
| `/forging-mcp-environment-setup`           | New MCP tool                                  | Nothing structural. The new module joins the server that skill documents                                                                     |
| `assets:library-extension`                 | Every blocking scenario                       | The upstream `AcquisitionSystems` or `SessionTypes` member, its `SYSTEM_SESSION_TYPES` pairing, and a new pipeline's session-record surfaces |
| `assets:session-data`                      | New pipeline                                  | The `instance.processed_data` field list, its asset count, the tracker-member count, and the session directory tree                          |
| `experiment:library-extension`             | New system, new session type, some pipelines  | The acquisition runtime that writes what a donation reads                                                                                    |
| `experiment:external-tool-bindings`        | New stage, per-system registry                | The bound tool that writes what a donated locator finds and a donated worker reads                                                           |
| `mesoscope:mesoscope-vr-processing-schema` | A change to the registered system's donations | The concrete donations that system makes                                                                                                     |
| `mesoscope:mesoscope-vr-dataset-assembly`  | A new session type on the registered system   | The assembly routing and the sub-dataset it produces                                                                                         |
| The new system's companion plugin          | New acquisition system                        | The per-system schema skills, which `assets:library-extension` and `experiment:library-extension` own                                        |

---

## Workflow

### Step 1: Identify the scenario and its blocking upstream half

Pick exactly one row of the scenario table and read the upstream column. A scenario with a blocking half does not start
here, and attempting the forgery half first produces an unimportable library that has no member to wire.

### Step 2: Land the upstream change

Hand off to `assets:library-extension` and confirm its checklist completed. Confirm the acquisition-side plan with
`experiment:library-extension` in parallel, since that half is required before the first session is processable even
though it blocks no import here.

### Step 3: Apply the code touches

Work through [references/extension-recipes.md](references/extension-recipes.md) for the scenario, in the order it
lists. Re-import after each registry entry, since the coverage check names one gap at a time. Then run the test
suite, because every touch point under "What the checks do not catch" needs explicit coverage.

### Step 4: Apply the documentation and skill touches

Add the `docs/source/api.rst` section or `autodata` directive the recipe names, add a new `*_tools.py` module to the
coverage omit list in `pyproject.toml`, then walk the cross-skill touch table and apply every update the scenario
names. Record concrete per-system material in that system's own schema skill rather than in a skill of this plugin,
which is acquisition-system-agnostic by contract.

### Step 5: Verify

Run the verification checklist below. The import gates are the safety net for registry membership, the dispatch table,
and the manifest columns and their rosters, and the manual items cover everything the gates do not reach.

### Citing source in a skill edit

A citation in this skill, and in every skill edit a recipe names, gives the asset name rather than its line number.
Line numbers drift as unrelated code above them moves, and a drifted citation points at the wrong asset while still
reading as authoritative. Naming the asset means naming the module plus one of the identifiers it declares, which is
a class, a method, a function, a dataclass field, an enum, an enum member, or a constant. The module path alone
suffices when the whole module is the subject.

| Rejected                | Correct                                                            |
|-------------------------|--------------------------------------------------------------------|
| `registries.py:530-560` | `_assert_registry_coverage()` in `registries.py`                   |
| `dispatch.py:90`        | the `_JOB_CORE_ALLOCATIONS` mapping in `orchestration/dispatch.py` |

Cross-document references follow the same rule. Cite a README or a CLAUDE.md by its section heading, never by a line.

---

## Pitfalls

| Pitfall                                                      | Why it bites                                                                                                                                                                |
|--------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Starting the forgery half before the upstream member lands   | See **Cross-repository ordering**                                                                                                                                           |
| Reading a clean import as a finished extension               | See **What the checks do not catch**                                                                                                                                        |
| Treating a system that produces no data of a class as exempt | See **The registry model**                                                                                                                                                  |
| Adding a stage and stopping at the pipeline                  | A stage also needs a core allocation, a sizing model with its routing branch and its mapped term, and a `_PIPELINE_JOB_NAMES` entry, none of which any check reaches        |
| Adding a per-session pipeline without its manifest column    | The declaring mapping and five further rosters name the column, and `_assert_status_column_coverage()` refuses the import until every one carries it, one message at a time |
| Minting a local job-name string for a dependency's stage     | The dependency exports the constant and its resource figures, and a local copy drifts the moment either is retuned                                                          |
| Registering a closure or a bound method as a donated worker  | The forging assemblers and the module parsers cross a process boundary, so a donation that is not a picklable module-level function fails at dispatch                       |
| Adding a per-system section to a skill in this plugin        | See **Step 4: Apply the documentation and skill touches**                                                                                                                   |
| Minting a per-system registry with no coverage-tuple row     | See **The registry model**                                                                                                                                                  |
| Planning a new per-session dataset file as a donation        | See **What a donation cannot widen**                                                                                                                                        |

---

## Related skills

The `video:`, `communication:`, `cindra:`, and `automation:` entries below resolve through the ataraxis and cindra
marketplaces. Every other entry resolves inside the sollertia marketplace.

| Skill                                           | Relationship                                                                                            |
|-------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `experiment:external-tool-bindings`             | Owns the binding convention whose locator and worker mint or join a registry here                       |
| `/forging-mcp-environment-setup`                | Owns the `slf mcp` server a new tool module joins, and the response contract that module returns        |
| `/data-processing-design`                       | Owns the per-system donation design behind a registry entry, and the doctrine a new stage follows       |
| `/dataset-definition`                           | Owns the admission policy and the column descriptions a new session type or system changes              |
| `/dataset-forging`                              | Owns the forging pipeline whose reach a new admission entry widens                                      |
| `/batch-processing`                             | Owns running the pipelines and stages an extension adds                                                 |
| `/job-planning`                                 | Owns the resource model a new job type joins through `_PIPELINE_JOB_NAMES`                              |
| `/project-state`                                | Owns the project manifest a new per-session pipeline adds a status column to                            |
| `/processing-input-format`                      | Owns the acquired artifacts a new donation or pipeline reads                                            |
| `/processing-results`                           | Owns the outputs a new stage or pipeline writes                                                         |
| `/cli-reference`                                | Owns the `slf` surface a new subcommand or stage flag joins                                             |
| `/pipeline`                                     | Routes an operator to the right skill once the extension lands                                          |
| `assets:library-extension`                      | Owns the blocking upstream half of every scenario, and gates it on its own import check                 |
| `assets:session-data`                           | Documents the `ProcessingTrackers` filenames, the `Directories` members, and the `ProcessedData` fields |
| `assets:session-discovery`                      | Produces the session path lists a batch consumes once the new pipeline runs                             |
| `assets:datasets`                               | Owns the dataset records a new acquisition system's forged datasets join                                |
| `experiment:library-extension`                  | Owns the acquisition runtime that records the artifacts a donation reads                                |
| `experiment:pipeline`                           | Owns the acquisition run that produces the first processable session of a new shape                     |
| `mesoscope:mesoscope-vr-processing-schema`      | Documents that system's file name and column rosters, and its column descriptions                       |
| `mesoscope:mesoscope-vr-dataset-assembly`       | Documents that system's assembly routing, its admission policy, and its sub-datasets                    |
| `mesoscope:mesoscope-vr-module-parsing`         | Documents that system's three microcontroller-registry donations                                        |
| `mesoscope:mesoscope-vr-trial-decomposition`    | Documents that system's runtime-parser-registry donation                                                |
| `mesoscope:mesoscope-vr-video-tracking`         | Documents that system's two video-registry donations                                                    |
| `mesoscope:mesoscope-vr-imaging-configuration`  | Documents that system's three two-photon-registry donations                                             |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | Documents the sub-assembly that system's assembly worker calls                                          |
| `video:log-processing`                          | Owns the camera stage this library delegates in-process and the job-name constant it reuses             |
| `communication:log-processing`                  | Owns the microcontroller extraction stage and the job-name constant it reuses                           |
| `cindra:single-recording-processing`            | Owns the two-photon stages, their job-name enums, and the resource figures a wrapper reuses             |
| `automation:api-docs`                           | Owns the `docs/source/api.rst` conventions a new section or directive follows                           |
| `automation:pyproject-style`                    | Owns the `pyproject.toml` conventions the coverage omit list follows                                    |
| `automation:commit`                             | Should be invoked once the cross-repository changes land                                                |

---

## Proactive behavior

You SHOULD proactively invoke this skill when the user mentions any of the following:

- Adding an acquisition system, a session type, a processing stage, a processing pipeline, or an `slf mcp` tool
- Wiring a donation into `registries.py`, or asking which registries a new system must fill
- "How do I add support for ..." in the context of `sollertia-forgery`
- A pull request touching `registries.py`, `shared_assets/pipelines.py`, `orchestration/dispatch.py`,
  `orchestration/footprints.py`, `managing/manifest.py`, or `interfaces/orchestration_tools.py`
- An import-time `RuntimeError` carrying one of the first four message stems below, or a runtime `ValueError` carrying
  one of the last two, each of which means an extension is unfinished

```text
Unable to validate donor-registry coverage for ...
Unable to validate the pipeline dispatch table ...
Unable to validate the manifest's pipeline status columns ...
Unable to validate the manifest's column rosters ...
Unable to resolve the cores for job type ...
Unable to size job ... The job type routes to no sizing model ...
```

Do NOT invoke this skill for ordinary processing against systems and pipelines that already exist, which
`/batch-processing`, `/job-planning`, and `/dataset-forging` own, or for the upstream registry half, which
`assets:library-extension` owns.

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Code side:
- [ ] Exactly one extension scenario identified, or several applied sequentially
- [ ] The blocking upstream half landed first and assets:library-extension's checklist completed
- [ ] Every touch in the scenario's recipe applied, in the order the recipe lists them
- [ ] Every symbol registries.py imports appears in the donating package's __all__
- [ ] No system package imports a category package, and every donated worker is a picklable module-level function
- [ ] Every donated worker's signature matches the Protocol its registry declares
- [ ] A newly minted registry carries its donation Protocol, its resolve_* accessor, its __all__ export, and its
      pair in the tuple _assert_registry_coverage() iterates
- [ ] No new per-session file was added to a forged dataset, or the change to reexported_assets and DatasetFiles was
      escalated to the human supervisor rather than worked around
- [ ] Every touch point under "What the checks do not catch" that this scenario reaches carries a test
- [ ] A new system or category package has a mirrored tests/ package, and a new system was added to the per-system
      assertions in tests/registry_coverage_test.py
- [ ] A newly minted registry's name was added to _DONOR_REGISTRY_NAMES in tests/registry_coverage_test.py
- [ ] docs/source/api.rst carries the section or autodata directive the recipe names
- [ ] A new *_tools.py module was added to [tool.coverage.run] omit in pyproject.toml

Code side, command-settled (run the three package imports, `slf --help`, `slf mcp`, and the three tox environments):
- [ ] python -c "import sollertia_forgery.registries" succeeds, which runs the donor-registry coverage check
- [ ] python -c "import sollertia_forgery.orchestration" succeeds, which runs the dispatch-table check
- [ ] python -c "import sollertia_forgery.managing" succeeds, which runs the manifest status-column check
- [ ] slf --help prints and slf mcp starts cleanly
- [ ] tox -e py314-test and tox -e coverage pass, and tox -e stubs regenerated the checked-in stubs

Skill side:
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
- [ ] Walked the cross-skill touch table and applied every update the scenario names
- [ ] Concrete per-system material recorded in that system's own companion plugin, not in a forging skill
- [ ] No other forging skill gained an acquisition-system-specific file, column, or session type
- [ ] Cross-references between the touched skills still resolve to a skill on the authoritative roster
- [ ] Every citation added to a touched skill names the asset or the section heading rather than a line

Downstream side:
- [ ] The acquisition runtime writes every artifact the new donations read, confirmed with
      experiment:library-extension by hand, since that library carries no import-time check
- [ ] A dependency's release landed before any stage that wraps one of its bindings
- [ ] Every handoff the recipe names is listed in the pull request description
- [ ] The sollertia-forgery version is bumped and its sollertia-shared-assets pin updated when a scenario
      changed the upstream contract
```
