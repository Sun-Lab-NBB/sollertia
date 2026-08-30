# Extension recipes

Completes each `sollertia-shared-assets` extension scenario touch point by touch point. Every section names the README
section that owns the line-by-line code recipe, then carries the touches that recipe leaves implicit, the sibling-skill
updates, and the downstream-library coordination.

You MUST read the named README section before applying a scenario, because the lists below add to that recipe rather
than replacing it. The import-time checks that catch an unfinished recipe, their verbatim failure messages, and the
touch points no check covers are documented in [guardrails.md](guardrails.md).

| Scenario                               | README section                          |
|----------------------------------------|-----------------------------------------|
| New `SessionTypes` member              | "Adding New Session Types"              |
| New `AcquisitionSystems` member        | "Adding New Acquisition Systems"        |
| New runtime trial class                | "Adding a New Trial Class"              |
| New `TriggerType` member               | "Adding a New Trigger Type"             |
| New `ReadAssets` member                | "Adding a New Read Asset"               |
| New raw-tree directory                 | None, this file carries the only recipe |
| New processing pipeline, upstream half | None, this file carries the only recipe |
| New `CredentialsTypes` member          | None, this file carries the only recipe |

---

## Conventions every scenario shares

**Export every new class twice.** A dataclass an extension adds is exported from the `__init__.py` of the package that
defines it, which is `<system>/` for system classes and `data_classes/` for contract classes. It is then re-exported
from the top-level `src/sollertia_shared_assets/__init__.py` **and its `__all__`**. Every name in that `__all__` is a
backwards-compatibility commitment the library maintains until the user asks for a breaking change, so add a name there
deliberately and remove none as part of an extension.

**The enum value is the on-disk contract, and the member name is the diagnostic.** A member's string value is written
into `session_data.yaml` and matched back by `AcquisitionSystems(...)` and `SessionTypes(...)` at load. It is therefore
chosen once, and renaming it later is a data migration rather than a rename. Every import-time diagnostic reports the
member **name** instead, so the two are read in different places. The live conventions are lowercase single tokens for
`AcquisitionSystems` (`MESOSCOPE_VR = "mesoscope"`) and space-separated lowercase words for `SessionTypes`
(`"lick training"`, `"mesoscope experiment"`).

**Every session artifact is addressed through a session-record field.** A file or a directory that lives inside a
session is named by a member of `RawDataFiles`, `Directories`, or `ProcessingTrackers` in
`data_hierarchy/session_data.py`. It is then declared as a field on `RawData` or on `ProcessedData` in that same module,
and resolved from the member inside the owning dataclass's `build` classmethod. The Notes of `RawData` name `build` the
single source of truth for the enum-to-field mapping. Producers then write through the resolved field, the way
`snapshot_surgery_data` in `sollertia-experiment`'s `cross_system/data_preprocessing.py` writes the surgery snapshot
through `session_data.raw_data.surgery_metadata_path`.

An artifact that stops at its dataclass and its registry entry passes every import-time check and is served by the
generic MCP tools, yet has no session-resolved location. Its producer then invents a path literal, and the session
record stops describing the session. No import-time check reaches this rule, because `_assert_registry_coverage()`
compares registry keys against enum members and never reads `data_hierarchy/session_data.py`. The path-resolution tests
in `tests/data_hierarchy/session_data_test.py` are the only guardrail over the enum, field, and `build` touches.

**Verify by importing.** `python -c "import sollertia_shared_assets"` runs every import-time check, because the package
`__init__.py` imports `registries.py` directly. Run it after the code touches of any scenario, and use
[guardrails.md](guardrails.md) to map a failure message onto the touch that is missing.

---

## Adding a new `SessionTypes` member

README section: "Adding New Session Types". A session type is the high-level activity a session runs. Each type owns one
descriptor dataclass, persisted under the flat `session_descriptor.yaml` filename and dispatched through
`DESCRIPTOR_REGISTRY`, so a system that needs a different descriptor mints a new session type rather than a second
registry entry.

**Code touches:**

1. Append the member to `SessionTypes` in `enums.py`, following the space-separated value convention above.
2. Add the `<Type>Descriptor` dataclass, inheriting `YamlConfig`, to the `runtime_data.py` module of the subpackage of
   the system that runs the new type. It MUST declare `incomplete: bool = True`, which the session-inspection tooling
   reads to decide whether the session is eligible for unsupervised processing. Export it from `<system>/__init__.py`,
   then re-export it from the top-level `src/sollertia_shared_assets/__init__.py` and its `__all__`.
3. In `registries.py`, import the descriptor from its system subpackage, register it under the new key in
   `DESCRIPTOR_REGISTRY`, and add the member to the `SYSTEM_SESSION_TYPES` frozenset of every acquisition system that
   can run it. A session type claimed by no system fails the import, and `SessionData.create()` rejects the type for any
   system whose set omits it.
4. Decide the required-asset policy. `SessionData.required_raw_assets` is data-driven rather than a per-type branch:
   every session requires `session_descriptor.yaml` and `system_configuration.yaml`, a session carrying an
   `experiment_name` also requires `experiment_configuration.yaml`, and a type listed in `SESSION_TYPES_USING_VR_TASK`
   also requires `vr_configuration.yaml`. Add the new type to that frozenset when it runs the corridor task, or extend
   `required_raw_assets` when it needs some other asset. Membership also gates session creation, because
   `SessionData.create()` rejects a session of a listed type created without an `experiment_name`. The VR task template
   is resolved from the experiment configuration's `unity_scene_name`, so a corridor-running training mode added to the
   frozenset is created with an experiment from that point on.
5. Cover the new type in `tests/data_hierarchy/session_data_test.py`, where `required_raw_assets` is unit-tested.
   Membership in `SESSION_TYPES_USING_VR_TASK` is not import-checked.

**Skill touches.** Visit every row and apply the named update. A row naming another skill's content points at the skill
that owns the concrete per-system material, so record the new material there.

| Skill                                   | What to update                                                                                                                                                                                                                |
|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`                         | The `SessionTypes` enumeration under "Session types", and the required-assets paragraph when the new type changes which snapshots a session must carry                                                                        |
| `/session-descriptors`                  | The generic `<session-type>_descriptor.yaml` placeholder shape and the path-resolution handoff. The skill stays system-agnostic and carries no filename roster                                                                |
| `mesoscope:mesoscope-vr-session-schema` | The new descriptor class, its field schema, and its persistent-cache filename, when Mesoscope-VR is the system that runs the new type                                                                                         |
| `mesoscope:mesoscope-vr-runtime`        | The runtime wiring of the new mode and the registry rules the skill restates from this recipe, when Mesoscope-VR is the system that runs the new type                                                                         |
| `/session-hardware-state`               | Nothing. The skill stays system-agnostic. Record which fields the new type populates, and whether it writes a `hardware_state.yaml`, in the owning system's schema skill                                                      |
| `/experiment-configuration`             | The statement of which sessions carry an experiment configuration. Any session created with an `experiment_name` is required to carry `experiment_configuration.yaml`, and only the VR-task snapshot is gated by session type |

**Downstream coordination:**

- `sollertia-experiment` creates sessions of the new type during acquisition. Hand off to
  `experiment:acquisition-system-runtime` and its per-system instance, and to `experiment:data-management` for the
  post-acquisition session lifecycle.
- `sollertia-forgery` decides whether sessions of the new type may join a forged dataset, and
  `forging:dataset-definition` owns that admission policy. The code touch is `MESOSCOPE_ADMISSION_PIPELINES` in
  `src/sollertia_forgery/mesoscope_vr/forging.py`, the per-system session-type mapping through which the admission
  registry dispatches for the reference system, reached through `_FORGING_ADMISSION_REGISTRY` in
  `src/sollertia_forgery/registries.py`. A session type absent from a system's mapping joins no dataset.
  `_MULTI_RECORDING_SESSION_TYPE_REGISTRY` in the same module is the second session-type-keyed structure, and it decides
  whether the new type's animals are tracked across recordings. Hand off to `forging:dataset-definition`.

---

## Adding a new `AcquisitionSystems` member

README section: "Adding New Acquisition Systems". A system contributes its own hardware-state snapshot,
experiment-configuration schema, and raw-data path builder, all inside one `<system>/` subpackage, and declares the
session types it can run. The system-level hardware and software configuration classes live in `sollertia-experiment`
rather than in this library.

**Code touches:**

1. Append the member to `AcquisitionSystems` in `enums.py`, following the single-token value convention above.
2. Create the `<system>/` subpackage as a sibling of `mesoscope_vr/`, holding three modules: `runtime_data.py` with
   `<System>HardwareState` and the system's per-session-type descriptors, `experiment_configuration.py` with
   `<System>ExperimentConfiguration` and the system's runtime trial classes, and `raw_data.py` with `<System>RawData`
   plus any `<System>RawDataFiles` and `<System>Directories` enums. Export every class from `<system>/__init__.py`, then
   re-export each of them from the top-level `src/sollertia_shared_assets/__init__.py` and its `__all__`.
3. In `registries.py`, import the three classes and register them in `HARDWARE_STATE_REGISTRY`,
   `EXPERIMENT_CONFIGURATION_REGISTRY`, and `SYSTEM_RAW_DATA_REGISTRY`, then add a `SYSTEM_SESSION_TYPES` entry mapping
   the new key to the frozenset of session types the system runs. Declare at least one type, since a key mapped to an
   empty frozenset fails the import exactly as a missing key does.
4. Give `<System>ExperimentConfiguration` the three contract fields `experiment_states`, `trial_structures`, and
   `unity_scene_name`, plus the `from_task_template` classmethod described below. No new MCP tool is needed:
   `create_experiment_from_vr_template_tool` dispatches through `EXPERIMENT_CONFIGURATION_REGISTRY`, and
   `write_experiment_configuration_tool` authors the system's full payload generically.
5. Apply the repo-level touches listed below, which no code recipe names.

### The `<System>RawData` build contract

Declare `<System>RawData` as `@dataclass(frozen=True, slots=True)`, matching `MesoscopeRawData`, and give it a
`build(cls, root: Path) -> <System>RawData` classmethod whose returned instance holds absolute paths anchored on `root`,
the session's `raw_data` directory. `SessionData` calls it to build the runtime-only `system_raw_data` attribute, so
registering the class is what wires the system into session loading.

The README shows `@dataclass(slots=True)` for this class, which is upstream drift from the only existing implementation.
Correct the README in the same pull request that adds the system.

`SYSTEM_RAW_DATA_REGISTRY` is annotated against the private `_SystemRawDataBuilder` Protocol, which is structural typing
with no runtime enforcement. No import-time check covers `build`, so a missing or misnamed classmethod surfaces as a
bare `AttributeError` inside `SessionData._build_sub_dataclasses` at session create or load time, rather than as a
guided `RuntimeError`. Verify the contract with a `SessionData.create()` smoke test for the new system.

### The `from_task_template` signature convention

`create_experiment_from_vr_template_tool` calls every registered builder with exactly three keyword arguments,
`template`, `unity_scene_name`, and `state_count`. The builder MUST therefore accept all three by keyword and MUST give
every other parameter a default, so the tool is able to omit the system-specific generation values. Declaring a contract
parameter `POSITIONAL_ONLY` fails the check, while a builder that accepts `**kwargs` satisfies the presence rule for a
contract parameter it does not name. A violation raises at import through `_assert_experiment_configuration_contract`,
which names the offending system and each gap.

`MesoscopeExperimentConfiguration.from_task_template` is the compliant shape to mirror. Its `template` and
`unity_scene_name` parameters carry no default, which is legal because contract parameters are exempt from the default
rule, and each of its system-specific `default_*` generation parameters carries one.

The builder maps the template's trial structures onto the system's runtime trial classes and seeds the default runtime
states. Nothing validates that `unity_scene_name` names a template that exists, and matching is the caller's
responsibility, so a mismatch surfaces later when `SessionData.create()` copies `<unity_scene_name>.yaml` out of the
task-templates directory.

### Repo-level touch points

| Touch point                 | What to add                                                                                                               | What catches an omission                                                                     |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| `docs/source/api.rst`       | A `<System> Assets` section carrying `.. automodule:: sollertia_shared_assets.<system>`, mirroring the Mesoscope-VR block | Nothing. `tox -e docs` passes and the subpackage is simply absent from the rendered API docs |
| `tests/<system>/`           | A test package mirroring the new subpackage                                                                               | `tox -e coverage`, whose gate is `fail_under = 100` and which omits only `interfaces/`       |
| The checked-in `.pyi` stubs | Regenerated stubs, produced by `tox -e stubs`, which depends on `tox -e lint`                                             | Nothing. A stale stub ships with the release                                                 |
| `pyproject.toml`            | A `sollertia-shared-assets` version bump                                                                                  | Nothing. The new system reaches the downstream libraries only through a released version     |

### Marketplace-level touch points

These four land in the `sollertia` plugin marketplace rather than in the library, and no code recipe names them. A
system whose extension stops at the code still runs, and stays invisible to every agent that would drive it.

| Touch point                           | What to add                                                                                   | What catches an omission                                                                                                                            |
|---------------------------------------|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `plugins/<system>/`                   | A companion plugin for the new system, carrying its own `.claude-plugin/plugin.json`          | Nothing. The directory belongs to no marketplace and installs nowhere                                                                               |
| `.claude-plugin/marketplace.json`     | A `plugins` entry naming the new plugin and pointing its `source` at `./plugins/<system>`     | Nothing. The plugin ships uninstallable                                                                                                             |
| `plugins/<system>/skills/`            | The three per-system schema skills to which the generic skills defer, named below the table   | Nothing. Every per-system pointer in the generic skills routes to a skill that does not exist                                                       |
| `experiment:acquisition-system-setup` | A row in its "Supported acquisition systems" table naming the new system and its schema skill | Nothing. `experiment:pipeline` routes to the running system's owning skill through that table, so operate-time routing never reaches the new system |

The three schema skills mirror the mesoscope trio, and copying that trio is the shortest route to them.
`mesoscope:mesoscope-vr-session-schema` carries the session-record schema, which is the descriptors and the
hardware-state snapshot. `mesoscope:mesoscope-vr-experiment-schema` carries the experiment-configuration schema, which
is the trial classes, the trial-kind discriminator, the trigger-to-trial mapping, and the builder defaults.
`mesoscope:mesoscope-vr-snapshots` carries the raw-data layout and the per-session position snapshots. Name the new
plugin after the system, and keep every concrete per-system value in it rather than in a core plugin skill.

**Skill touches.** Apply each update so the skill's framing and pointers cover the new member alongside the existing
one, enumerating both explicitly rather than lengthening a "currently only X" chain.

| Skill                       | What to update                                                                                                                                                                                          |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`             | The `instance.system_raw_data` bullet under "Path-resolution sub-dataclasses on `SessionData`", which gains the new `<System>RawData` field list                                                        |
| `/session-hardware-state`   | The per-system schema-skill pointers in the intro, the read and amend workflows, and the checklist. Add a parallel pointer to the new system's schema skill and a matching Related-skills row           |
| `/experiment-configuration` | The frontmatter description's per-system schema-skill pointer and the pointers in the body. The new system reuses `create_experiment_from_vr_template_tool` once its `from_task_template` builder lands |
| `/task-templates`           | The statement naming the experiment-configuration classes into which a template can be built                                                                                                            |
| `/datasets`                 | The acquisition-system vocabulary a dataset records, since `DatasetData` carries the acquisition system on which its sessions were acquired                                                             |

**Downstream coordination:**

- `sollertia-experiment` owns the system-level hardware and software configuration classes and the acquisition runtime.
  Hand off to `experiment:acquisition-system-design` for the configuration and binding-class design, and to
  `experiment:acquisition-system-runtime` for the runtime behavior.
- The "Author the system's MCP tool module" step of `experiment:acquisition-system-design`'s "Building a new acquisition
  system from scratch" workflow adds the per-system `interfaces/<system>_tools.py` module, without which the system is
  CLI-driveable but exposes no system-specific MCP surface. The "Author the system's dedicated companion plugin" step
  authors the system's agentic assets, a per-system instance skill and, when the system has non-trivial runtime modes, a
  per-system runtime skill. Both live in the system's companion plugin rather than in the experiment plugin, and both
  are required deliverables. The assets plugin's generic skills carry pointers that assume the per-system schema skills
  exist, so a system that stops at the code is driveable yet undocumented for every agent that would drive it.
- `sollertia-forgery` dispatches every per-system behavior through thirteen registries in
  `src/sollertia_forgery/registries.py`, and the new system needs an entry in each of them before its sessions are
  processed or forged. `forging:library-extension` carries the roster and the donation shape each one expects, and
  `_assert_registry_coverage()` in that module walks all thirteen in one coverage tuple and reports every system a
  registry omits. Two of the thirteen gate the dataset seam outright. `_FORGING_ASSEMBLY_REGISTRY` carries the system's
  `column_descriptions`, which the agnostic pipeline bakes into the dataset's `data_descriptions.feather` when the
  dataset is defined, and `_FORGING_ADMISSION_REGISTRY` carries the per-session-type pipeline requirements, so a session
  type absent from a system's mapping joins no dataset. Hand off to `forging:dataset-definition` for the admission
  policy and the column-description companion, and to `forging:data-processing-design` for the per-stage processing
  design behind the remaining entries.
- `sollertia-virtual-reality` may need new scene scaffolding when the new system uses Unity.

---

## Adding a new runtime trial class

README section: "Adding a New Trial Class". Trial classes are acquisition-system-specific and live next to that system's
experiment configuration. The spatial layout of a trial, its cues and zones, lives on the matching `TrialStructure` in
the paired Unity task template.

**Code touches:**

1. Define the class in the owning system's `<system>/experiment_configuration.py` as a standalone
   `@dataclass(frozen=True, slots=True)`, prefixing its name with the system name. It carries its runtime parameters
   (rewards, durations, thresholds) plus the trial-kind discriminator, and declares no spatial fields.
2. Wire the trial-kind discriminator, which is four separate edits in the same module:
   1. Add a member to the system's `TrialKind` enum.
   2. Declare `trial_kind: TrialKind = TrialKind.<NEW>` on the new class, with the matching `__post_init__` rejection of
      every other member.
   3. Add the `(TrialKind.<NEW>, <NewTrial>)` pair to the module's `_TRIAL_CLASSES` tuple, which drives
      `_restore_trial_kind` and `_unique_trial_fields`.
   4. Add the class to the `isinstance` acceptance tuple in `<System>ExperimentConfiguration.__post_init__`.
3. Export the class from `<system>/__init__.py`, then re-export it from the top-level
   `src/sollertia_shared_assets/__init__.py` and its `__all__`.
4. Add the class to the `trial_structures` type-union annotation of every `<System>ExperimentConfiguration` that uses
   it, for example `dict[str, <ExistingTrial> | <NewTrial>]`. `list_supported_trial_types_tool` derives a system's trial
   vocabulary from that annotation, so a class absent from the union never surfaces in the tooling.
5. Map a `TriggerType` member to the class inside that configuration's `from_task_template`. A trigger that no branch
   handles raises, so every trigger the template can carry on this system needs a branch.
6. Cover the new class in the experiment-configuration tests.

Skipping any of the four discriminator edits ships a trial class that serializes but cannot be deserialized, and every
configuration containing it raises at load. No import-time check covers step 2, step 4, or step 5, so the tests are the
only guardrail. Mesoscope-VR's concrete discriminator members, trial classes, and field schemas are documented in
`mesoscope:mesoscope-vr-experiment-schema`.

**Skill touches:**

| Skill                                      | What to update                                                                                                                                                                   |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/experiment-configuration`                | The "Templates vs. experiment configurations" framing, the trigger to trial-class pairing convention, the `trial_structures` schema description, and the "Common patterns" table |
| `/task-templates`                          | The trial-class enumeration in the template vocabulary section                                                                                                                   |
| `mesoscope:mesoscope-vr-experiment-schema` | The trial-class field roster and the trigger-to-trial mapping table, whenever Mesoscope-VR is the system that gains the class                                                    |
| `experiment:vr-driver-interface`           | How the orchestrator dispatches per-trigger outcomes through the driver's `Stimulus` events, joined by `DecomposedTrials.trial_names`                                            |

---

## Adding a new `TriggerType` member

README section: "Adding a New Trigger Type". The enum is platform-wide, and each acquisition system maps only the subset
of trigger types it implements to its runtime trial classes, so a new member does not require a branch in every system.

The full extension is split four ways, and each skill owns its slice:

| Slice                                                                                   | Owning skill                       |
|-----------------------------------------------------------------------------------------|------------------------------------|
| The Python `TriggerType` enum and the per-supporting-system `from_task_template` branch | `/library-extension` (this recipe) |
| Zone prefab manufacturing (`clone_zone_prefab_tool`)                                    | `unity:zone-prefabs`               |
| `CreateTask` pipeline edits and `DeleteProtectedPaths`                                  | `unity:task-generator`             |
| EditMode and PlayMode test updates                                                      | `unity:unity-tests`                |

**Code touches:**

1. Append the member to `TriggerType` in `configuration/vr_configuration.py`, which is where the enum lives rather than
   in the leaf `enums.py` module.
2. Decide whether the new member reads a dwell time. An occupancy-style member is added to the `occupancy_types` tuple
   in `TrialStructure.__post_init__` in the same module, which holds `OCCUPANCY_DISARM`, `OCCUPANCY_ARM`, and
   `OCCUPANCY_TRIGGER` today and gates whether `occupancy_duration_ms` is required. An occupancy-style member left out
   of that tuple ships with no dwell-time validation, so a template that omits the duration loads cleanly and the trial
   reaches the acquisition runtime with `occupancy_duration_ms=None`.
3. Classify the new member for geometry validation in `TaskTemplate._validate_zone_positions`, also in the same module.
   That method sets `validates_zone = trigger_value != TriggerType.COLLISION.value` and
   `validates_boundary = trigger_value != TriggerType.OCCUPANCY_TRIGGER.value`, so every member other than those two
   validates the trigger zone, the stimulus boundary, and their relative ordering. A collision-style member left out
   of that classification raises spurious geometry errors on every legitimate template that uses it.
4. For each system that supports the new member, add the matching branch to that system's `from_task_template`,
   instantiating the runtime trial class to which the trigger resolves. A system that leaves the member unmapped raises
   the "not mapped to a runtime trial class" error for it, which is the intended unsupported-on-this-system signal
   rather than a wiring bug, so record the per-system decision explicitly.
5. Cover the new branch and both per-member classifications in the configuration tests. No import-time check covers
   trigger coverage.

Steps 2 and 3 are silent at import time. The enum member itself imports cleanly whether or not either branch mentions
it, and neither branch is named by any of the checks in [guardrails.md](guardrails.md), so the tests in
`tests/configuration/vr_configuration_test.py` are the only guardrail over them.

**Skill touches:**

| Skill                                      | What to update                                                                                                                                                                                                                                                                                                                          |
|--------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/task-templates`                          | The `TriggerType` enumeration sentence and the primitives table in SKILL.md, plus `references/field-semantics.md`. That reference hardcodes the trigger-mode count in its intro, its section heading, and three sentences of that section, counts the occupancy modes separately, and carries a firing-rule table with one row per mode |
| `/experiment-configuration`                | The trigger to trial-class pairing convention                                                                                                                                                                                                                                                                                           |
| `mesoscope:mesoscope-vr-experiment-schema` | The trigger-to-trial mapping table, whenever Mesoscope-VR gains a branch for the new member                                                                                                                                                                                                                                             |

---

## Adding a new read asset

README section: "Adding a New Read Asset". A read asset is metadata the platform reads from an external,
human-maintained source, such as a surgery log, and caches on disk as a typed dataclass, so downstream consumers read
the dataclass and never the source. An asset the platform only writes to an external source has no on-disk
representation to standardize, so it needs no dataclass and no registry entry and stays owned by the writing library.
`READ_ASSET_REGISTRY` is a maintainer-curated contract registry: each entry is a durable translation contract, and
adding one is a platform-contract decision rather than a routine extension.

**Code touches:**

1. Add the `<Asset>Data` dataclass in a new module under `data_classes/`, inheriting `YamlConfig`. Contract modules
   export plain dataclasses and never consume the dispatch registries. Export it from `data_classes/__init__.py`, then
   re-export it from the top-level `src/sollertia_shared_assets/__init__.py` and its `__all__`.
2. Append the member to `ReadAssets` in `enums.py`.
3. Register the dataclass under the new key in `READ_ASSET_REGISTRY` in `registries.py`.
4. Add a `RawDataFiles` member in `data_hierarchy/session_data.py` holding the canonical filename the cached asset
   takes at the root of a session's `raw_data` directory, following `SURGERY_METADATA = "surgery_metadata.yaml"`.
5. Declare the matching `<asset>_path: Path` field on `RawData` in the same module, following `surgery_metadata_path`.
6. Resolve that field from the new member inside `RawData.build`, following
   `surgery_metadata_path=root.joinpath(RawDataFiles.SURGERY_METADATA)`.
7. Extend `test_session_data_raw_data_file_paths` in `tests/data_hierarchy/session_data_test.py` with the new field.
8. Run `python -c "import sollertia_shared_assets"`. A forgotten registry entry is named at import time.

Touches 4 through 6 are what give the asset a session-resolved home, per the session-record convention above. An asset
cached per animal rather than per session resolves through `AnimalData` in `data_hierarchy/project_hierarchy.py`, whose
`persistent_data_path` property addresses the animal's cross-session directory, so record which of the two locations
the asset takes before applying touch 4.

### Resolving the new asset from Python

`resolve_read_asset(read_asset: str | ReadAssets) -> type[YamlConfig]` is the code-path counterpart of
`read_data_asset_tool` and becomes usable for the new asset the moment its registry entry lands. It accepts a
`ReadAssets` member or its raw string value, returns the registered dataclass, and raises `ValueError` with:

```text
Unable to resolve the read-asset format '<value>'. Expected one of the supported ReadAssets members: <valid>.
```

It exists for `sollertia-experiment` and `sollertia-forgery` and has no in-repo caller, because the MCP tools resolve
through their own `_resolve_read_asset_class`, which returns an error envelope with different wording instead of
raising. `assets:data-assets` documents the MCP path.

**Skill touches:**

| Skill                                 | What to update                                                                                                                                             |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/data-assets`                        | Nothing structural. The generic data-asset tools serve the new asset automatically once it is registered. Add it to the worked examples when it is notable |
| `/datasets`                           | The read-asset artifacts a dataset inventory reports per animal, when the new asset is cached at the animal level                                          |
| `experiment:google-sheets-processing` | The reader that translates the external source and the registered dataclass it produces                                                                    |

**Downstream coordination:**

- `sollertia-experiment` owns the reader that translates the external source into the new dataclass and caches it on
  disk. This recipe owns the dataclass, the registry entry, and the asset's session-resolved location only.
- `sollertia-forgery` consumes the cached on-disk dataclass during dataset assembly.

---

## Adding a raw-tree directory for a new artifact

The library README carries no section for this scenario, so this recipe is the only one. It applies when an acquisition
runtime, or an external tool bound into one, writes an artifact into the session's `raw_data` tree and no existing
directory field is the right home for it. `experiment:external-tool-bindings` routes here from the artifact-home step of
its binding workflow.

**Code touches:**

1. Add the member to `Directories` in `data_hierarchy/session_data.py`, following `BEHAVIOR_DATA = "behavior_data"`.
   The enum names the subdirectories of both session trees, so the member's docstring states under which of `raw_data`
   and `processed_data` the directory sits, the way every existing member's docstring opens.
2. Declare the matching `<name>_path: Path` field on `RawData` in the same module, following `behavior_data_path`.
3. Resolve that field from the new member inside `RawData.build`, following
   `behavior_data_path=root.joinpath(Directories.BEHAVIOR_DATA)`.
4. Extend `test_session_data_raw_data_directory_paths` in `tests/data_hierarchy/session_data_test.py` with the new
   field.

`build` resolves paths and creates nothing, and `SessionData.create()` calls `ensure_directory_exists` for the
`raw_data` root alone, so the producer that writes into the new directory creates it. `required_raw_assets` is not
touched, because it lists the files a session must contain rather than its directories.

**Skill touches:**

| Skill           | What to update                                                                                                                                                |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data` | The `instance.raw_data` field list under "Path-resolution sub-dataclasses on `SessionData`", and the raw-tree branch of the session directory tree it renders |

**Downstream coordination:**

- `sollertia-experiment` owns the producer that writes the artifact. Hand off to `experiment:external-tool-bindings`
  for a bound external tool, and to `experiment:acquisition-system-runtime` when the acquisition runtime writes it
  directly.
- `sollertia-forgery` is involved only when a processing stage reads the artifact back, which
  `forging:library-extension` owns.

---

## Adding the session-record surfaces of a new processing pipeline

The library README carries no section for this scenario, so this recipe is the only one. A new `sollertia-forgery`
processing pipeline writes into a directory under the session's `processed_data` tree and records its outcome in a
tracker beside that output, and both are addressed through `ProcessedData`. `forging:library-extension` names this half
blocking and lands it here, because `_SESSION_TRACKER_LOCATIONS` in that library's `shared_assets/pipelines.py` resolves
each per-session pipeline's tracker through a `session.raw_data` or `session.processed_data` accessor that has to exist
first. The Notes of `ProcessedData` state that future processing tools applying across acquisition systems get added
directly as new fields there, so this recipe is the intended growth path for the dataclass rather than a widening of a
frozen contract.

**Code touches:**

1. Add the member naming the pipeline's output directory to `Directories` in `data_hierarchy/session_data.py`,
   following `RUNTIME_DATA = "runtime_data"`, with a docstring stating that it sits under `processed_data`.
2. Add the member holding the tracker filename to `ProcessingTrackers` in the same module, following
   `RUNTIME = "runtime_processing_tracker.yaml"`. Choose the member **name** to mirror the downstream
   `ProcessingPipelines` member, whose own Notes in `sollertia-forgery`'s `shared_assets/pipelines.py` state that the
   two names mirror each other.
3. Declare the field pair on `ProcessedData`, one `<name>_data_path: Path` for the output directory and one
   `<name>_tracker_path: Path` for the tracker, following `runtime_data_path` and `runtime_tracker_path`.
4. Resolve both inside `ProcessedData.build`, binding the directory to a local variable and joining the tracker filename
   onto it, following `runtime_data_path = root.joinpath(Directories.RUNTIME_DATA)` and
   `runtime_tracker_path=runtime_data_path.joinpath(ProcessingTrackers.RUNTIME)`. The tracker is resolved under the
   pipeline's own output directory rather than under the `processed_data` root.
5. Add the new member to the `descriptions` dictionary inside `list_processing_trackers_tool` in
   `interfaces/data_tools.py`. Its entry comprehension iterates every `ProcessingTrackers` member and indexes that
   dictionary, so a member the dictionary omits raises `KeyError` on the first call of the tool.
6. Extend `test_session_data_processed_data_directory_paths` and `test_session_data_processing_tracker_paths` in
   `tests/data_hierarchy/session_data_test.py` with the new directory and tracker.

A tracker written outside a session takes touches 2 and 5 alone, plus a value assertion in
`test_processing_trackers_enum_is_string_enum` in `tests/data_hierarchy/session_data_test.py`, because no fixed
per-session path addresses it and neither test named in touch 6 can assert on it. Both tests resolve a
`session.processed_data.<field>` attribute, which such a tracker never gains. `ProcessingTrackers.FORGING` lives at the
forged dataset root and `ProcessingTrackers.MANIFEST` at the project root, and neither carries a `ProcessedData` field.

**Skill touches:**

| Skill           | What to update                                                                                                                                                                |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data` | The `instance.processed_data` field list and its asset count, the count of `ProcessingTrackers` members dispatched onto a sub-dataclass field, and the session directory tree |

**Downstream coordination:**

- `sollertia-forgery` owns the pipeline itself, its category package, its dispatch entry, and its admission wiring. Hand
  off to `forging:library-extension`, whose processing-pipeline recipe lists this half as blocking and lands first.
- `sollertia-experiment` is involved only when the pipeline reads an artifact acquisition does not yet write, which
  `experiment:external-tool-bindings` and `experiment:acquisition-system-runtime` own between them.

---

## Adding a new credentials category

The library README carries no section for this scenario, so this recipe is the only one. The import-time failure message
for a forgotten registry entry routes the reader to the "Adding New Session Types", "Adding New Acquisition Systems",
and "Adding a New Read Asset" README sections, none of which covers credentials.

`CREDENTIALS_FILE_REGISTRY` is the second maintainer-curated contract registry and the only registry whose value is a
canonical filename string rather than a class.

**Code touches:**

1. Append the member to `CredentialsTypes` in `enums.py`.
2. Add the `CredentialsTypes.<NEW>: "<name>.<ext>"` entry to `CREDENTIALS_FILE_REGISTRY` in `registries.py`. The
   filename is the canonical name the credentials file takes inside the working directory's `credentials` subdirectory,
   and `set_credentials` rejects a source file whose extension differs from it, so choose the extension the external
   service actually issues.
3. Nothing else. `resolve_credentials_file`, `set_credentials`, `get_credentials`, and
   `list_supported_credentials_tool`, along with the `slsa configure credentials --category` choice list, all derive
   their vocabulary from the enum and the registry, so no tool, CLI option, or choice list is edited.

**Skill touches:**

| Skill                | What to update                                                                                                         |
|----------------------|------------------------------------------------------------------------------------------------------------------------|
| `/working-directory` | The credentials-category roster it documents alongside `list_supported_credentials_tool` and the credentials directory |

**Downstream coordination:**

- `sollertia-experiment` owns the client that consumes the credentials file for the external service. Hand off to
  `experiment:google-sheets-processing`, which is the current worked example of an external-service integration.
