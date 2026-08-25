---
name: library-extension
description: >-
  Owns the extension path of the sollertia-shared-assets registry system: adding an AcquisitionSystems,
  SessionTypes, ReadAssets, CredentialsTypes, or TriggerType member, a runtime trial class, or a new MCP tool
  module. Covers the per-scenario touch lists, the four import-time contract checks, the list_supported_* registry
  introspection family, and the sibling-skill updates. Use when implementing a new acquisition system against the
  Mesoscope-VR reference, adding any registry entry, or when an import-time RuntimeError names a registry.
user-invocable: false
---

# Sollertia library extension

Extends `sollertia-shared-assets` along its registry-backed extension points and keeps the sibling skills aligned with
the new state of the library. The library README owns the line-by-line code recipes, and this skill owns the model
those recipes assume, the guardrails that verify the result, and the cross-skill and repository-level touches the
README leaves implicit.

You MUST read this entire skill before extending the library, then read
[references/extension-recipes.md](references/extension-recipes.md) for the scenario you are applying and
[references/guardrails.md](references/guardrails.md) for the checks that verify it. You MUST run the verification
checklist before reporting an extension complete.

---

## Scope

**Covers:**
- Adding an `AcquisitionSystems` member, with its `<System>HardwareState`, `<System>ExperimentConfiguration`, and
  `<System>RawData` classes
- Adding a `SessionTypes` member, with its descriptor dataclass
- Adding a runtime trial class, with the trial-kind discriminator that makes it deserializable
- Adding a `TriggerType` member, with the per-system `from_task_template` branches that map it
- Adding a `ReadAssets` member, with its on-disk dataclass and `READ_ASSET_REGISTRY` entry
- Adding a `CredentialsTypes` member, with its `CREDENTIALS_FILE_REGISTRY` filename
- Adding an MCP tool module to the `slsa mcp` server
- The registry model, the four import-time contract checks, and the `list_supported_*` introspection family that
  confirms an extension landed
- The cross-skill touch list and the downstream coordination with sollertia-experiment and sollertia-forgery

**Does not cover:**
- The line-by-line code changes, which the `sollertia-shared-assets` README owns in its "Adding New Session Types",
  "Adding New Acquisition Systems", "Adding a New Trial Class", "Adding a New Trigger Type", and "Adding a New Read
  Asset" sections. Read the section the scenario table names before applying any recipe
- Per-asset CRUD against registered systems and session types (see `/session-data`, `/session-descriptors`,
  `/session-hardware-state`, `/experiment-configuration`, `/task-templates`, `/data-assets`, and `/datasets`)
- System-level acquisition runtime configuration, which lives in `sollertia-experiment` (see
  `experiment:acquisition-system-design`)
- Server-side processing configuration, which lives in `sollertia-forgery` (see `forging:server-configuration`)
- Unity prefab and scene authoring (see `unity:zone-prefabs`, `unity:task-prefabs`, and `unity:task-scenes`)

---

## Registry model

`sollertia-shared-assets` dispatches every system-specific and asset-specific behavior through six registries, defined
as fully populated literals in the top-level `registries.py` module. Each maps a member of one of the four enums in
the leaf `enums.py` module to the Python class, or the canonical filename, that the MCP tools use to parse, validate,
or build the corresponding asset. Two further structures live beside them in the same module. `SYSTEM_SESSION_TYPES`
is an association rather than a dispatch registry, recording which session types each acquisition system can run, and
`SESSION_TYPES_USING_VR_TASK` is an unkeyed gate. The enum members live in `enums.py`, the dispatched classes live in
the system subpackages and the `data_classes/` contract package, and `registries.py` imports both without
circularity, so a registry entry is added by editing `registries.py` directly.

Two governance tiers divide the eight structures. The **system tier** grows with every new acquisition system or
session type and is the extension point this skill orchestrates. The **contract tier**, `READ_ASSET_REGISTRY` and
`CREDENTIALS_FILE_REGISTRY`, is maintainer-curated, so adding an entry there is a platform-contract decision rather
than a routine extension.

| Structure                           | Tier     | Keyed by             | Maps to                                        |
|-------------------------------------|----------|----------------------|------------------------------------------------|
| `DESCRIPTOR_REGISTRY`               | System   | `SessionTypes`       | Per-session-type descriptor dataclass          |
| `HARDWARE_STATE_REGISTRY`           | System   | `AcquisitionSystems` | Per-system hardware-state dataclass            |
| `EXPERIMENT_CONFIGURATION_REGISTRY` | System   | `AcquisitionSystems` | Per-system experiment-configuration dataclass  |
| `SYSTEM_RAW_DATA_REGISTRY`          | System   | `AcquisitionSystems` | Per-system raw-data sub-dataclass with `build` |
| `SYSTEM_SESSION_TYPES`              | System   | `AcquisitionSystems` | `frozenset[SessionTypes]` the system can run   |
| `SESSION_TYPES_USING_VR_TASK`       | System   | Not keyed            | `frozenset[SessionTypes]` that run the VR task |
| `READ_ASSET_REGISTRY`               | Contract | `ReadAssets`         | Per-read-asset on-disk dataclass               |
| `CREDENTIALS_FILE_REGISTRY`         | Contract | `CredentialsTypes`   | Canonical credentials filename per category    |

Two properties of that table decide how it is used:

- `SYSTEM_RAW_DATA_REGISTRY` is the only one of the eight that is not re-exported from the package root, so import it
  as `from sollertia_shared_assets.registries import SYSTEM_RAW_DATA_REGISTRY`. The other six registries and
  `SESSION_TYPES_USING_VR_TASK` are importable from `sollertia_shared_assets` directly.
- `SESSION_TYPES_USING_VR_TASK` is the seventh system-tier touch point and the one system-tier touch point no
  import-time check covers. Extend it whenever a new session type runs the corridor task, because it decides whether
  `SessionData.required_raw_assets` demands the `vr_configuration.yaml` snapshot.

Every `<System>ExperimentConfiguration` shares one contract: an `experiment_states` field holding the experiment state
machine, a `trial_structures` field holding the trials the experiment runs, a `unity_scene_name` field naming the
corridor scene the system presents, and a `from_task_template` builder. Fields beyond the contract are
system-specific, and the concrete trial classes vary per system. `list_supported_trial_types_tool(acquisition_system)`
derives a system's trial vocabulary from the `trial_structures` annotation through the `collect_field_dataclasses`
helper, so a trial class absent from that annotation never surfaces in the tooling.

Each system's `from_task_template` maps only the subset of `TriggerType` members it implements to runtime trial
classes and leaves the rest unmapped. A configuration that uses an unmapped member raises the "not mapped to a runtime
trial class" error, which is the intended unsupported-on-this-system signal rather than a wiring bug, so a new
`TriggerType` member does not require a branch in every system. `mesoscope:mesoscope-vr-experiment-schema` documents
the mapping the current reference system implements.

---

## Import-time guardrails

`registries.py` runs three assertions at the bottom of its module body, and the third folds in a fourth check:

| Check                                         | What it enforces                                                                                                                                                                                                         |
|-----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `_assert_registry_coverage()`                 | Every member of the six keyed enums has a dispatch entry, every acquisition system declares at least one session type, and every session type is claimed by at least one system                                          |
| `_assert_descriptor_contract()`               | Every registered descriptor declares a field named `incomplete`                                                                                                                                                          |
| `_assert_experiment_configuration_contract()` | Every registered `<System>ExperimentConfiguration` declares `experiment_states`, `trial_structures`, and `unity_scene_name`, and provides a callable `from_task_template`                                                |
| `_experiment_builder_signature_gaps()`        | `template`, `unity_scene_name`, and `state_count` are reachable by keyword on `from_task_template`, which rules out `POSITIONAL_ONLY` unless the builder accepts `**kwargs`, and every other parameter carries a default |

The package `__init__.py` imports `registries.py` directly, so all four run on a bare `import sollertia_shared_assets`
rather than only when `slsa mcp` starts. An extension that misses an entry in one of the six dispatch registries
therefore fails fast and names the offending registry and member, which is the one class of omission that cannot slip
through silently. [references/guardrails.md](references/guardrails.md) maps each verbatim `RuntimeError` message onto
the touch it names.

### What the checks do not catch

Six touch points pass a bare import and fail later, so tests rather than a guardrail cover each one: the four
trial-kind discriminator edits, the `trial_structures` type union, the trigger-to-trial mapping, the
`_SystemRawDataBuilder.build` contract, `SESSION_TYPES_USING_VR_TASK` membership, and a stale registry key left behind
by a removed enum member, which passes because the coverage check computes only `expected - actual`.
[references/guardrails.md](references/guardrails.md) carries how each omission surfaces and where to cover it.

---

## Registry introspection

Seven MCP tools report the live contents of the enums and registries, which makes them the runtime confirmation that
an extension landed. Each is a comprehension over the live vocabulary, so a new enum member appears the moment its
registry entry lands and no MCP tool and no CLI option is added or edited for it. The
`slsa configure credentials --category` choice list auto-extends the same way. Every response carries the envelope
documented under `## Response contract` in `assets:assets-mcp-environment-setup`.

| Tool                                                         | Keying enum                | Source structure                                     | Payload key                                   |
|--------------------------------------------------------------|----------------------------|------------------------------------------------------|-----------------------------------------------|
| `list_supported_acquisition_systems_tool()`                  | `AcquisitionSystems`       | The enum itself                                      | `acquisition_systems`, `{value, name}` pairs  |
| `list_supported_session_types_tool(acquisition_system=None)` | `SessionTypes`             | `SYSTEM_SESSION_TYPES` filter, `DESCRIPTOR_REGISTRY` | `session_types`, each with `descriptor_class` |
| `list_session_type_support_tool()`                           | `AcquisitionSystems`       | `SYSTEM_SESSION_TYPES`                               | `session_type_support`                        |
| `list_supported_data_assets_tool()`                          | `ReadAssets`               | `READ_ASSET_REGISTRY`                                | `data_assets`, each with `data_asset_class`   |
| `list_supported_credentials_tool()`                          | `CredentialsTypes`         | `CREDENTIALS_FILE_REGISTRY`                          | `credentials`, each with `file_name`          |
| `list_supported_trial_types_tool(acquisition_system)`        | None, argument is required | `EXPERIMENT_CONFIGURATION_REGISTRY`                  | `trial_types`                                 |
| `list_supported_trigger_types_tool()`                        | `TriggerType`              | The enum itself                                      | `trigger_types`                               |

Per-domain usage guidance stays with the skills that own each domain. This skill owns the family as the
extension-verification surface.

### Extending the MCP tool surface

`mcp_server.py` globs `*_tools.py` inside `src/sollertia_shared_assets/interfaces/` in `sorted()` order at import and
imports every match, so tools register purely as an import side effect. To add a tool module, drop `<name>_tools.py`
into that directory, import `mcp` from `.mcp_instance`, and decorate each function with `@mcp.tool()`, returning
`dict[str, Any]` built by `ok_response` or `error_response`. The module registers with zero edits to `mcp_server.py`.
`interfaces/` is the one package the coverage gate omits, so a tool module ships without a mirrored test package.
`assets:assets-mcp-environment-setup` documents the server the new module joins.

---

## Extension scenarios

Pick exactly one row, read the README section it names, then apply the recipe it points at. The reference adds the
cross-skill and repository-level touches on top of each README recipe, so it completes the recipe rather than
replacing it.

| Scenario                        | README section                          | Recipe                                                                                       |
|---------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------|
| New `SessionTypes` member       | "Adding New Session Types"              | [Session type](references/extension-recipes.md#adding-a-new-sessiontypes-member)             |
| New `AcquisitionSystems` member | "Adding New Acquisition Systems"        | [Acquisition system](references/extension-recipes.md#adding-a-new-acquisitionsystems-member) |
| New runtime trial class         | "Adding a New Trial Class"              | [Trial class](references/extension-recipes.md#adding-a-new-runtime-trial-class)              |
| New `TriggerType` member        | "Adding a New Trigger Type"             | [Trigger type](references/extension-recipes.md#adding-a-new-triggertype-member)              |
| New `ReadAssets` member         | "Adding a New Read Asset"               | [Read asset](references/extension-recipes.md#adding-a-new-read-asset)                        |
| New `CredentialsTypes` member   | None, the recipe carries the whole flow | [Credentials category](references/extension-recipes.md#adding-a-new-credentials-category)    |

### Reading the Mesoscope-VR reference

Mesoscope-VR is the only registered acquisition system today and the worked example a new system mirrors. Read the
four skills that document it in this order, against the three modules of the `<system>/` subpackage you are authoring:

| Read                                       | What it documents                                                                                                                                 | Module you are authoring                                      |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| `mesoscope:mesoscope-vr-session-schema`    | The reference hardware-state snapshot and its per-session-type descriptors                                                                        | `<system>/runtime_data.py`                                    |
| `mesoscope:mesoscope-vr-experiment-schema` | The reference trial classes, its `TrialKind` discriminator and `_TRIAL_CLASSES` tuple, its trigger-to-trial mapping, and its `from_task_template` | `<system>/experiment_configuration.py`                        |
| `mesoscope:mesoscope-vr-snapshots`         | The reference raw-data layout the system's path builder produces                                                                                  | `<system>/raw_data.py`                                        |
| `mesoscope:mesoscope-vr`                   | The sollertia-experiment side of the reference system, where its runtime, configuration, and CLI live                                             | None here. Hand off to `experiment:acquisition-system-design` |

Mirror the shape and leave the values behind. Every concrete Mesoscope-VR field name, enum member, and filename stays
in the mesoscope plugin, and a new system's equivalents are recorded in that system's own schema skill.

---

## Cross-skill touch table

Each recipe names the skills its scenario touches. This table is the inverse view, which is what a review of a
finished extension checks against.

| Skill                                                             | Touched by                                                 | What changes                                                                                                                                                                                              |
|-------------------------------------------------------------------|------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`                                                   | New session type, new acquisition system                   | The `SessionTypes` enumeration, the required-assets paragraph, and the `instance.system_raw_data` field list                                                                                              |
| `/session-descriptors`                                            | New session type                                           | The generic `<session-type>_descriptor.yaml` placeholder shape and the path-resolution handoff. The skill carries no per-system filename roster                                                           |
| `/session-hardware-state`                                         | New acquisition system                                     | The per-system schema-skill pointers and the matching Related-skills row. The skill stays system-agnostic across new session types                                                                        |
| `/experiment-configuration`                                       | New session type, acquisition system, trial class, trigger | The per-system schema-skill pointers, the trigger to trial-class pairing convention, and the rule that any session created with an `experiment_name` is required to carry `experiment_configuration.yaml` |
| `/task-templates`                                                 | New trial class, new trigger type, new acquisition system  | The `TriggerType` enumeration and primitives table, the trial-class enumeration, and the experiment-configuration classes a template builds into                                                          |
| `/data-assets`                                                    | New read asset                                             | Nothing structural. Add the asset to the worked examples when it is notable                                                                                                                               |
| `/datasets`                                                       | New read asset, new acquisition system                     | The acquisition-system vocabulary a dataset records and the read-asset artifacts an inventory reports per animal                                                                                          |
| `/working-directory`                                              | New credentials category                                   | The credentials-category roster documented alongside `list_supported_credentials_tool`                                                                                                                    |
| `mesoscope:mesoscope-vr-session-schema`                           | New session type run by Mesoscope-VR                       | The new descriptor class, its field schema, and its persistent-cache filename                                                                                                                             |
| `mesoscope:mesoscope-vr-experiment-schema`                        | New trial class or trigger branch on Mesoscope-VR          | The trial-class field roster and the trigger-to-trial mapping table                                                                                                                                       |
| `experiment:vr-driver-interface`                                  | New trial class                                            | How the orchestrator dispatches per-trigger outcomes through the driver's `Stimulus` events                                                                                                               |
| `experiment:google-sheets-processing`                             | New read asset, new credentials category                   | The reader that translates the external source, and the client for a new external service                                                                                                                 |
| `unity:zone-prefabs`, `unity:task-generator`, `unity:unity-tests` | New trigger type                                           | The three Unity slices of the trigger-type recipe, each owned by the skill named                                                                                                                          |

---

## Workflow

### Step 1: Identify the extension scenario

Pick exactly one row of the scenario table. A change that spans several scenarios, such as a new acquisition system
that also introduces a new session type, applies their recipes sequentially rather than interleaved, because the
import-time checks report one structure at a time.

### Step 2: Apply the code touches

Read the README section the scenario table names for **every** scenario, then apply the touch list in
[references/extension-recipes.md](references/extension-recipes.md), which adds the cross-skill and repository-level
updates on top of each README recipe. The credentials scenario has no README section, so its recipe carries the whole
flow. Run `python -c "import sollertia_shared_assets"` once the code touches land, then run the test suite, because
the import-time checks cover the dispatch registries alone and every touch point under "What the checks do not catch"
needs explicit test coverage.

### Step 3: Apply the skill touches

Work through every row the recipe names, cross-checked against the cross-skill touch table above. A row naming
content to update points at a SKILL.md edited directly. A row naming where concrete per-system material belongs points
at the owning system's schema skill, so record the new material there. Enumerate the new member alongside the existing
one explicitly rather than rewriting "currently only X" into a longer chain, and leave the skills the recipe omits
alone.

### Step 4: Coordinate with downstream libraries

Each recipe lists the downstream libraries that need parallel changes. Hand those off to the matching skill in the
affected plugin rather than attempting them here, and list the hand-offs in the pull request description so the
reviewer is able to confirm the cross-repository coordination happened.

### Step 5: Verify

Run the verification checklist below. The import-time checks are the safety net for the dispatch registries, the
`list_supported_*` family confirms the new member reached the tooling, and the manual items cover everything neither
one reaches.

---

## Pitfalls

| Pitfall                                                | Why it bites                                                                                                                                                                                                          |
|--------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trusting the import to prove the extension is complete | The checks cover the six dispatch registries and two contract shapes. Six further touch points fail only at load, at session creation, or silently, so run the test suite and the matching `list_supported_*` call    |
| Changing an existing dataclass schema in place         | On-disk YAML documents written by earlier releases can fail to load. Treat a schema change to a registered dataclass as a migration coordinated through the library's semantic versioning rather than as an extension |
| Reading an unmapped trigger as a wiring bug            | A system maps only the trigger subset it implements, so an unmapped member is a deliberate per-system choice. Record the decision explicitly rather than adding a branch to silence the raise                         |

---

## Related skills

The `automation:commit` entry below resolves through the ataraxis marketplace. Every other entry resolves inside the
sollertia marketplace.

| Skill                                      | Relationship                                                                                                                                                                                                                           |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`            | Owns the MCP response contract every introspection tool returns, and diagnoses the server startup failure an incomplete extension causes                                                                                               |
| `/working-directory`                       | Bootstraps the working directory and the credentials directory every extension touch point consumes                                                                                                                                    |
| `/session-data`                            | Owns the session record a new session type or acquisition system widens                                                                                                                                                                |
| `/session-descriptors`                     | Owns the generic descriptor placeholder shape a new session type extends                                                                                                                                                               |
| `/session-hardware-state`                  | Owns the generic hardware-state surface a new acquisition system extends                                                                                                                                                               |
| `/experiment-configuration`                | Owns authoring against a registered `<System>ExperimentConfiguration` once the extension lands                                                                                                                                         |
| `/task-templates`                          | Owns the task template a `from_task_template` builder consumes                                                                                                                                                                         |
| `/data-assets`                             | Owns the read and amend surface a new read asset gains automatically                                                                                                                                                                   |
| `/datasets`                                | Owns the dataset vocabulary a new acquisition system or read asset widens                                                                                                                                                              |
| `mesoscope:mesoscope-vr-session-schema`    | The reference descriptor and hardware-state schema a new `<system>/runtime_data.py` mirrors                                                                                                                                            |
| `mesoscope:mesoscope-vr-experiment-schema` | The reference trial classes, discriminator, and trigger mapping a new `<system>/experiment_configuration.py` mirrors                                                                                                                   |
| `mesoscope:mesoscope-vr-snapshots`         | The reference raw-data layout a new `<system>/raw_data.py` mirrors                                                                                                                                                                     |
| `mesoscope:mesoscope-vr`                   | The sollertia-experiment side of the reference acquisition system                                                                                                                                                                      |
| `experiment:system-design-pipeline`        | Orchestrates the four-layer build of a new acquisition system and designates this skill as Phase 1. Return there once the shared-assets contract lands                                                                                 |
| `experiment:acquisition-system-design`     | Owns the runtime-side configuration and binding-class design. Step 9 of its from-scratch workflow adds the per-system `interfaces/<system>_tools.py` MCP tool module, and steps 10 and 11 author the system's dedicated agentic assets |
| `experiment:acquisition-system-runtime`    | Owns the runtime that creates and runs sessions of a new session type during acquisition                                                                                                                                               |
| `experiment:data-management`               | Manages the post-acquisition lifecycle of sessions of any type                                                                                                                                                                         |
| `experiment:google-sheets-processing`      | Owns the reader that translates an external source into a read asset's on-disk dataclass, and the client for a new credentials category                                                                                                |
| `experiment:vr-driver-interface`           | Decomposes the VR cue sequence into `DecomposedTrials`, which a new trial class joins through `trial_names`                                                                                                                            |
| `forging:behavior-input-format`            | Decides eligibility of a new session type for behavior processing                                                                                                                                                                      |
| `forging:project-manifest`                 | Tabulates a new session type in the project manifest                                                                                                                                                                                   |
| `forging:dataset-forging-input-format`     | Decides eligibility of a new session type for dataset forging                                                                                                                                                                          |
| `unity:zone-prefabs`                       | Manufactures the trigger zone prefab a new `TriggerType` member needs through `clone_zone_prefab_tool`                                                                                                                                 |
| `unity:task-generator`                     | Owns the `CreateTask` pipeline edits and `DeleteProtectedPaths` a new `TriggerType` member needs                                                                                                                                       |
| `unity:unity-tests`                        | Owns the EditMode and PlayMode fixtures a new `TriggerType` member changes                                                                                                                                                             |
| `unity:task-prefabs`                       | Builds or removes the task prefabs and the scene for a template through `create_task_tool` and `delete_task_tool`                                                                                                                      |
| `unity:task-scenes`                        | Lists, opens, and inspects the scenes generated per task template                                                                                                                                                                      |
| `automation:commit`                        | Should be invoked once the cross-cutting changes land                                                                                                                                                                                  |

---

## Proactive behavior

You SHOULD proactively invoke this skill when the user mentions any of the following:

- Adding a new acquisition system, session type, trial type or trial class, trigger type, read asset, or credentials
  category
- Adding an MCP tool module to the `slsa mcp` server
- "How do I add support for ..." in the context of `sollertia-shared-assets`
- A pull request that touches `registries.py` or `enums.py`, or any structure the registry model table names
- An import-time `RuntimeError` carrying one of the five message stems below, which means an earlier extension is
  incomplete

```text
<REGISTRY> is missing entries for ...
SYSTEM_SESSION_TYPES is missing entries for ...
SYSTEM_SESSION_TYPES does not claim ...
DESCRIPTOR_REGISTRY descriptors are missing the required 'incomplete' field for ...
EXPERIMENT_CONFIGURATION_REGISTRY classes do not satisfy the experiment-configuration contract for ...
```

Do NOT invoke this skill for ordinary CRUD against registered systems and session types, which `/session-data`,
`/session-descriptors`, `/session-hardware-state`, `/experiment-configuration`, `/task-templates`, `/data-assets`, and
`/datasets` own.

---

## Verification checklist

```text
Code side:
- [ ] Identified exactly one extension scenario, or applied several sequentially
- [ ] Read the README section the scenario table names, then applied the matching recipe in
      references/extension-recipes.md
- [ ] Every new class is exported from its own package __init__.py and re-exported from the top-level __init__.py
      and its __all__
- [ ] `python -c "import sollertia_shared_assets"` succeeds, which runs all four import-time checks
- [ ] The new member appears in the matching `list_supported_*` call
- [ ] Every touch point under "What the checks do not catch" that this scenario reaches carries a test
- [ ] A new acquisition system passes a `SessionData.create()` smoke test and gains a `tests/<system>/` package, a
      `docs/source/api.rst` section, and regenerated `.pyi` stubs
- [ ] The `sollertia-shared-assets` version is bumped (if a new acquisition system)
- [ ] Test suite passes and `slsa mcp` starts cleanly

Skill side:
- [ ] Walked the cross-skill touch table and applied every update the scenario names
- [ ] Enumerated the new member alongside the existing one instead of lengthening a "currently only X" chain
- [ ] Recorded concrete per-system material in the owning system's schema skill rather than in a generic skill
- [ ] Cross-references between the touched skills still resolve

Downstream side:
- [ ] The downstream hand-offs the recipe names are listed in the pull request description
- [ ] The README recipe for the chosen scenario still describes the code after the change, and any drift it picked
      up is corrected in the same pull request
- [ ] A new acquisition system is handed to `experiment:system-design-pipeline` only once
      `python -c "import sollertia_shared_assets"` succeeds AND the system appears in
      `list_supported_acquisition_systems_tool`
```
