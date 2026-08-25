# Extension recipes

Completes each `sollertia-shared-assets` extension scenario touch point by touch point. Every section names the README
section that owns the line-by-line code recipe, then carries the touches that recipe leaves implicit, the sibling-skill
updates, and the downstream-library coordination.

You MUST read the named README section before applying a scenario, because the lists below add to that recipe rather
than replacing it. The import-time checks that catch an unfinished recipe, their verbatim failure messages, and the
touch points no check covers are documented in [guardrails.md](guardrails.md).

| Scenario                        | README section                          |
|---------------------------------|-----------------------------------------|
| New `SessionTypes` member       | "Adding New Session Types"              |
| New `AcquisitionSystems` member | "Adding New Acquisition Systems"        |
| New runtime trial class         | "Adding a New Trial Class"              |
| New `TriggerType` member        | "Adding a New Trigger Type"             |
| New `ReadAssets` member         | "Adding a New Read Asset"               |
| New `CredentialsTypes` member   | None, this file carries the only recipe |

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
   `required_raw_assets` when it needs some other asset.
5. Cover the new type in `tests/data_hierarchy/session_data_test.py`, where `required_raw_assets` is unit-tested.
   Membership in `SESSION_TYPES_USING_VR_TASK` is not import-checked.

**Skill touches.** Visit every row and apply the named update. A row naming another skill's content points at the skill
that owns the concrete per-system material, so record the new material there.

| Skill                                   | What to update                                                                                                                                                                                                                |
|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`                         | The `SessionTypes` enumeration under "Session types", and the required-assets paragraph when the new type changes which snapshots a session must carry                                                                        |
| `/session-descriptors`                  | The generic `<session-type>_descriptor.yaml` placeholder shape and the path-resolution handoff. The skill stays system-agnostic and carries no filename roster                                                                |
| `mesoscope:mesoscope-vr-session-schema` | The new descriptor class, its field schema, and its persistent-cache filename, when Mesoscope-VR is the system that runs the new type                                                                                         |
| `/session-hardware-state`               | Nothing. The skill stays system-agnostic. Record which fields the new type populates, and whether it writes a `hardware_state.yaml`, in the owning system's schema skill                                                      |
| `/experiment-configuration`             | The statement of which sessions carry an experiment configuration. Any session created with an `experiment_name` is required to carry `experiment_configuration.yaml`, and only the VR-task snapshot is gated by session type |

**Downstream coordination:**

- `sollertia-experiment` creates sessions of the new type during acquisition. Hand off to
  `experiment:acquisition-system-runtime` and its per-system instance, and to `experiment:data-management` for the
  post-acquisition session lifecycle.
- `sollertia-forgery` decides whether the new type is eligible for each per-session pipeline. Hand off to
  `forging:behavior-input-format`, `forging:project-manifest`, and `forging:dataset-forging-input-format`.

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

**Skill touches.** Apply each update so the skill's framing and pointers cover the new member alongside the existing
one, enumerating both explicitly rather than lengthening a "currently only X" chain.

| Skill                       | What to update                                                                                                                                                                                          |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`             | The `instance.system_raw_data` bullet under "Path-resolution sub-dataclasses on `SessionData`", which gains the new `<System>RawData` field list                                                        |
| `/session-hardware-state`   | The per-system schema-skill pointers in the intro, the read and amend workflows, and the checklist. Add a parallel pointer to the new system's schema skill and a matching Related-skills row           |
| `/experiment-configuration` | The frontmatter description's per-system schema-skill pointer and the pointers in the body. The new system reuses `create_experiment_from_vr_template_tool` once its `from_task_template` builder lands |
| `/task-templates` | The statement naming the experiment-configuration classes into which a template can be built |
| `/datasets` | The acquisition-system vocabulary a dataset records, since `DatasetData` carries the acquisition system on which its sessions were acquired |

**Downstream coordination:**

- `sollertia-experiment` owns the system-level hardware and software configuration classes and the acquisition runtime.
  Hand off to `experiment:acquisition-system-design` for the configuration and binding-class design, and to
  `experiment:acquisition-system-runtime` for the runtime behavior.
- Step 9 of `experiment:acquisition-system-design`'s "Building a new acquisition system from scratch" workflow adds the
  per-system `interfaces/<system>_tools.py` MCP tool module, without which the system is CLI-driveable but exposes no
  system-specific MCP surface. Steps 10 and 11 of the same workflow author the system's dedicated agentic assets, a
  per-system instance skill and, when the system has non-trivial runtime modes, a per-system runtime skill. Those two
  are optional for the system to run, and omitting them leaves the system driveable yet undocumented for agents.
- `sollertia-forgery` may need new behavior-processing or video-processing branches per system.
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

| Skill                                      | What to update                                                                                                                                                                  |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/experiment-configuration`                | The "Templates vs. experiment configurations" framing, the trigger to trial-class pairing convention, the `trial_structures` schema description, and the "Common patterns" table |
| `/task-templates`                          | The trial-class enumeration in the template vocabulary section                                                                                                                  |
| `mesoscope:mesoscope-vr-experiment-schema` | The trial-class field roster and the trigger-to-trial mapping table, whenever Mesoscope-VR is the system that gains the class                                                   |
| `experiment:vr-driver-interface`           | How the orchestrator dispatches per-trigger outcomes through the driver's `Stimulus` events, joined by `DecomposedTrials.trial_names`                                           |

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
2. For each system that supports the new member, add the matching branch to that system's `from_task_template`,
   instantiating the runtime trial class to which the trigger resolves. A system that leaves the member unmapped raises
   the "not mapped to a runtime trial class" error for it, which is the intended unsupported-on-this-system signal
   rather than a wiring bug, so record the per-system decision explicitly.
3. Cover the new branch in the experiment-configuration tests. No import-time check covers trigger coverage.

**Skill touches:**

| Skill                                      | What to update                                                                              |
|--------------------------------------------|---------------------------------------------------------------------------------------------|
| `/task-templates`                          | The `TriggerType` enumeration sentence and the primitives table                             |
| `/experiment-configuration`                | The trigger to trial-class pairing convention                                               |
| `mesoscope:mesoscope-vr-experiment-schema` | The trigger-to-trial mapping table, whenever Mesoscope-VR gains a branch for the new member |

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
4. Run `python -c "import sollertia_shared_assets"`. A forgotten registry entry is named at import time.

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
  disk. This recipe owns the dataclass and the registry entry only.
- `sollertia-forgery` consumes the cached on-disk dataclass during dataset assembly.

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
