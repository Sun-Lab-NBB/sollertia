---
name: library-extension
description: >-
  Orchestrates cross-cutting changes to extend the sollertia-shared-assets library with a new
  acquisition system, session type, trial class, or trigger type. Covers the
  registry touch list, import-time parity check, and sibling-skill enumeration updates. Use
  when adding a new AcquisitionSystems / SessionTypes / TriggerType member or runtime trial
  class.
user-invocable: false
---

# Sollertia library extension

Orchestrates extending the `sollertia-shared-assets` library along its registry-backed
extensibility points and keeping the assets-plugin skills aligned with the new state of the
library. The library defines the canonical recipes in its README; this skill exists to surface
the **cross-skill update list** that the README does not enforce, to point at the import-time
parity check that does enforce the code-side coverage, and to coordinate the downstream-library
hand-offs.

You MUST read this entire skill before extending the library, and you MUST run through the
verification checklist before reporting an extension complete.

---

## Scope

**Covers:**
- Adding a new `AcquisitionSystems` member (with its `<System>HardwareState`,
  `<System>ExperimentConfiguration`, and `<System>RawData`)
- Adding a new `SessionTypes` member (with its descriptor dataclass)
- Adding a new runtime trial class
- Adding a new `TriggerType` member (and, for each system that supports it, the trigger →
  trial-class pairing in that system's `from_task_template`; a system may leave a member unmapped)
- Adding a new `ReadAssets` member (with its on-disk dataclass and `READ_ASSET_REGISTRY` entry) — the
  sollertia-shared-assets contract for an external asset the platform reads and caches on disk
- The cross-skill touch list — which other plugin skills carry hardcoded enumerations that drift
  the moment a new enum member is added
- Coordination with downstream libraries (sollertia-experiment, sollertia-forgery)

**Does not cover:**
- The line-by-line code changes themselves. Those are owned by the
  **`sollertia-shared-assets` README** sections "Adding New Session Types", "Adding New
  Acquisition Systems", and "Adding a New Read Asset" — this skill defers to the README as the
  authoritative recipe.
- Authoring per-asset CRUD on existing systems and session types (see
  `/session-data`, `/session-descriptors`, `/session-hardware-state`,
  `/experiment-configuration`, `/task-templates`)
- System-level acquisition runtime configuration (lives in `sollertia-experiment` — see the
  experiment plugin's `/acquisition-system-design`)
- Server-side processing configuration (lives in `sollertia-forgery` — see the forging plugin's
  `/server-configuration`)
- Unity prefab and scene authoring (see the unity plugin's `/task-prefabs` and `/task-scenes`)

---

## Library extension model

`sollertia-shared-assets` is structured around six **dispatch registries**, all defined — fully
populated — in the top-level `registries.py` module and keyed by members of the `SessionTypes`,
`AcquisitionSystems`, `ReadAssets`, and `CredentialsTypes` enums (all in the top-level leaf module
`enums.py`). Each registry resolves a string identifier to the Python class (or canonical filename)
that the MCP tools use to parse, validate, or build the corresponding asset. A separate
**association**, `SYSTEM_SESSION_TYPES`, records which session types each acquisition system can
run; it is keyed by `AcquisitionSystems` but maps to a `frozenset` of `SessionTypes` rather than a
dispatch class, and `SessionData.create()` rejects any session-type / acquisition-system pairing it
does not contain. The system-facing registries are the extension point this skill orchestrates;
`READ_ASSET_REGISTRY` and `CREDENTIALS_FILE_REGISTRY` are maintainer-curated contract registries —
adding an entry there is a platform-contract decision, not a routine extension.

| Registry                            | Keyed by             | Maps to                                        |
|-------------------------------------|----------------------|------------------------------------------------|
| `DESCRIPTOR_REGISTRY`               | `SessionTypes`       | Per-session-type descriptor dataclass          |
| `HARDWARE_STATE_REGISTRY`           | `AcquisitionSystems` | Per-system hardware-state dataclass            |
| `EXPERIMENT_CONFIGURATION_REGISTRY` | `AcquisitionSystems` | Per-system experiment-configuration dataclass  |
| `SYSTEM_RAW_DATA_REGISTRY`          | `AcquisitionSystems` | Per-system raw-data sub-dataclass with `build` |
| `SYSTEM_SESSION_TYPES`              | `AcquisitionSystems` | `frozenset[SessionTypes]` the system can run   |
| `READ_ASSET_REGISTRY`               | `ReadAssets`         | Per-read-asset on-disk dataclass               |
| `CREDENTIALS_FILE_REGISTRY`         | `CredentialsTypes`   | Canonical credentials filename per category    |

These registries, the `SYSTEM_SESSION_TYPES` association, the `SESSION_TYPES_USING_VR_TASK` gate,
`resolve_read_asset`, and the import-time checks that guard them are all defined — as plain,
fully populated literals — in the top-level `registries.py` module. The enum members live in the
leaf module `enums.py`; the dispatched classes live in the system subpackages (e.g.
`mesoscope_vr/`) and the `data_classes/` contract package, which `registries.py` imports without
circularity. When you add a registry entry, edit `registries.py` directly.

Every `<System>ExperimentConfiguration` shares one contract: an `experiment_states` field (a mapping
of `ExperimentState` — the experiment state machine; every experiment is a state machine, so this is
required), a `trial_structures` field (the trials the experiment runs — required; the concrete
trial classes vary per system), a `unity_scene_name` field (the corridor scene the system presents),
and a `from_task_template` builder. Fields beyond the contract are system-specific.
Mesoscope-VR is one exemplar of the contract: its `MesoscopeWaterRewardTrial` / `MesoscopeGasPuffTrial`
trial classes are exemplar-specific, and another system has its own trial classes and may add
different fields.

A system's trial classes are introspected from its experiment configuration's `trial_structures`
field via the `collect_field_dataclasses` helper in
`interfaces/mcp_instance.py`. `list_supported_trial_types_tool(acquisition_system)` resolves the
system's experiment-configuration class and derives the trial vocabulary from that field;
`MesoscopeExperimentConfiguration.from_task_template` maps the subset of `TriggerType` members it
supports to runtime trial classes (it maps `INTERACTION` → `MesoscopeWaterRewardTrial` and
`OCCUPANCY_DISARM` → `MesoscopeGasPuffTrial`; `COLLISION`, `OCCUPANCY_ARM`, and `OCCUPANCY_TRIGGER`
are intentionally unmapped and raise a clear "not mapped to a runtime trial class" error if a
Mesoscope-VR config uses them).
The trial classes themselves are standalone dataclasses defined in the owning system's subpackage
(Mesoscope-VR's live in `mesoscope_vr/experiment_configuration.py`, next to
`MesoscopeExperimentConfiguration`). `from_task_template` is a mandatory contract method on
every `<System>ExperimentConfiguration`, enforced at import by `_assert_experiment_configuration_contract`
(`registries.py`), and `create_experiment_from_vr_template_tool` dispatches through
`EXPERIMENT_CONFIGURATION_REGISTRY` to the resolved system's builder.

### The parity check

`registries.py` (the top-level registry hub) runs `_assert_registry_coverage()` at import time. The function
walks `(DESCRIPTOR_REGISTRY, HARDWARE_STATE_REGISTRY, EXPERIMENT_CONFIGURATION_REGISTRY,
SYSTEM_RAW_DATA_REGISTRY, READ_ASSET_REGISTRY, CREDENTIALS_FILE_REGISTRY)` and raises `RuntimeError` if any enum
member is missing its dispatch class. It additionally checks `SYSTEM_SESSION_TYPES`: every acquisition system
must declare at least one session type, and every session type must be claimed by at least one
system. The hub also runs two contract checks at import: `_assert_descriptor_contract()` (every registered descriptor
must declare the `incomplete` field the inspection tooling reads) and `_assert_experiment_configuration_contract()`
(every registered `<System>ExperimentConfiguration` must declare the contract fields `experiment_states`,
`trial_structures`, and `unity_scene_name` plus the `from_task_template` builder). Because the package `__init__.py`
imports `registries.py` directly, all of these run on a bare `import sollertia_shared_assets` (not
only when `slsa mcp` starts), so an incomplete extension fails fast and names the offending registry; it cannot
silently slip through.

---

## Authoritative code recipes

The `sollertia-shared-assets` README owns the canonical, line-numbered recipes. **Do not retype
them here** — read them, then come back for the cross-skill update map below.

| Scenario                        | README section                   |
|---------------------------------|----------------------------------|
| New `SessionTypes` member       | "Adding New Session Types"       |
| New `AcquisitionSystems` member | "Adding New Acquisition Systems" |
| New runtime trial class         | "Adding a New Trial Class"       |
| New `TriggerType` member        | "Adding a New Trigger Type"      |
| New read asset                  | "Adding a New Read Asset"        |

The README carries a recipe for every scenario in the table above. The **per-scenario touch lists**
below add the cross-skill updates that accompany each recipe.

---

## Per-scenario touch lists

Each scenario lists the code touches (defer to the README where one exists), the **skill
touches** (which is what this skill uniquely owns), and the downstream-library coordination.

### Adding a new `SessionTypes` member

**Code touches** — follow the README's "Adding New Session Types" recipe:
1. Append the member to `SessionTypes` (`enums.py`), and add it to the `SYSTEM_SESSION_TYPES`
   frozenset of every acquisition system that can run it (`registries.py`). The parity check fails
   if the new type is claimed by no system, and `SessionData.create()` rejects it for any system
   whose set omits it.
2. Add the `<Type>Descriptor` dataclass in the `runtime_data.py` module of the subpackage of the system that runs
   the new type (the Mesoscope-VR descriptors live in `mesoscope_vr/runtime_data.py`); export it from the
   subpackage's `__init__.py`. The descriptor MUST declare an `incomplete: bool = True` field — the
   session-inspection tooling reads it, and `_assert_descriptor_contract()` fails the import if it is missing.
3. Register the descriptor in `DESCRIPTOR_REGISTRY` (`registries.py`) — import the class from its system
   subpackage there.
4. The required-asset policy lives in `SessionData.required_raw_assets` (`data_hierarchy/session_data.py`), not in a
   per-session-type branch. It is data-driven: `experiment_configuration.yaml` is required whenever the session has
   an `experiment_name`, and `vr_configuration.yaml` is required for any session type listed in the
   `SESSION_TYPES_USING_VR_TASK` frozenset (`registries.py`). If the new type runs the corridor task, add it to that
   frozenset; if it needs some other extra raw asset, extend `required_raw_assets`.
5. Run the test suite — `_assert_registry_coverage()` catches a forgotten descriptor entry or an unclaimed session
   type, and `_assert_descriptor_contract()` fails the import if the descriptor omits `incomplete`. A missing
   `SESSION_TYPES_USING_VR_TASK` entry is not import-checked, so cover the new type in
   `tests/data_hierarchy/session_data_test.py`, where `required_raw_assets` is unit-tested.

**Skill touches** — update each of the following so its hardcoded enumeration matches the new
member:

| Skill                       | What to update                                                                                                                                                                                       |
|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`             | The `SessionTypes` enumeration sentence under "Session types"; the required-assets paragraph if the new type changes which per-session snapshots are required                                        |
| `/session-descriptors`      | The "Session types and descriptor classes" mapping table                                                                                                                                             |
| `/session-hardware-state`   | The "Per-session-type field population" table — add a row for the new type even if it produces no hardware-state file (record the absence explicitly so callers do not infer "missing data")         |
| `/experiment-configuration` | Mention the new session type only if the experiment-configuration flow accepts it (it currently does not — only `mesoscope experiment` consumes the experiment configuration snapshot)               |

**Downstream coordination:**
- `sollertia-experiment` actually creates sessions of the new type during acquisition. Hand off to
  the experiment plugin's `/acquisition-system-runtime` (and its `/mesoscope-vr-runtime` instance)
  for the session-running runtime, and `/data-management` for the post-acquisition session lifecycle.
- `sollertia-forgery` per-session pipelines (behavior, manifest, dataset forging) may need to
  decide whether the new type is eligible. Hand off to the forging plugin's `/behavior-input-format`,
  `/project-manifest`, and `/dataset-forging-input-format`.

### Adding a new `AcquisitionSystems` member

**Code touches** — follow the README's "Adding New Acquisition Systems" recipe:
1. Append the member to `AcquisitionSystems` (`enums.py`).
2. Create a new `<system>/` subpackage (a sibling of `mesoscope_vr/`) holding all the system's dataclasses:
   `<system>/runtime_data.py` holds `<System>HardwareState` plus the system's per-session-type descriptors (mirror
   `mesoscope_vr/runtime_data.py`); `<system>/raw_data.py` holds `<System>RawData` with its `build` classmethod and
   any `<System>RawDataFiles` / `<System>Directories` enums (mirror `mesoscope_vr/raw_data.py`); and
   `<system>/experiment_configuration.py` holds `<System>ExperimentConfiguration` (mirror
   `mesoscope_vr/experiment_configuration.py`). Export every class from the subpackage's `__init__.py`.
3. In `registries.py`, import the new classes from the system subpackage and register them in
   `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`, and `SYSTEM_RAW_DATA_REGISTRY`, and add a
   `SYSTEM_SESSION_TYPES` entry mapping the new system to the `frozenset` of `SessionTypes` it can run (declare at
   least one, or the parity check fails).
4. Add the `from_task_template` classmethod to its `<System>ExperimentConfiguration` (mirror
   `MesoscopeExperimentConfiguration.from_task_template`) alongside the contract fields `experiment_states`,
   `trial_structures`, and `unity_scene_name`. The shared `create_experiment_from_vr_template_tool` dispatches
   through `EXPERIMENT_CONFIGURATION_REGISTRY` to the system's `from_task_template`, so no new tool is needed;
   `write_experiment_configuration_tool` already authors the system's full payload generically.
5. Run the test suite — `_assert_registry_coverage()` catches the dispatch registries and the
   `SYSTEM_SESSION_TYPES` pairing, and `_assert_experiment_configuration_contract()` fails the import if the new
   `<System>ExperimentConfiguration` omits a contract field or the `from_task_template` builder.

**Skill touches** — update each of the following so its hardcoded "currently only Mesoscope-VR"
framing reflects the new member:

| Skill                       | What to update                                                                                                                                                                                                                                                                                                           |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`             | The `instance.system_raw_data` bullet under "Path-resolution sub-dataclasses on `SessionData`" — add the new `<System>RawData` field list; the `Mesoscope-VR` mention in "Does not cover" if the experiment plugin's `/mesoscope-vr-snapshots` becomes one of several owners of system-specific snapshots                |
| `/session-hardware-state`   | The frontmatter description, the "currently the only concrete subclass is `MesoscopeHardwareState`" prose, and the "Per-session-type field population (Mesoscope-VR example)" framing — clone the table format for the new system                                                                                        |
| `/experiment-configuration` | The frontmatter description, the "currently only `MesoscopeExperimentConfiguration`" prose, and any per-trial-class assumptions specific to the Mesoscope-VR rig. The new system reuses `create_experiment_from_vr_template_tool` once its `<System>ExperimentConfiguration` implements the `from_task_template` builder |
| `/task-templates`           | The "currently only `MesoscopeExperimentConfiguration`" mention                                                                                                                                                                                                                                                          |

**Downstream coordination:**
- `sollertia-experiment` owns the system-level hardware/software configuration classes and the
  acquisition runtime for the new system. Hand off to the experiment plugin's
  `/acquisition-system-design` for the configuration and binding-class design, and
  `/acquisition-system-runtime` for the runtime behavior; the runtime work itself is out of scope
  for this skill.
- A new acquisition system **optionally benefits from dedicated agentic assets** in the experiment
  plugin: a per-system instance skill (modeled on `experiment:mesoscope-vr`) and, when the system
  has non-trivial runtime modes, a per-system runtime skill (modeled on
  `experiment:mesoscope-vr-runtime`). These are authored through `/acquisition-system-design`'s
  "Building a new acquisition system from scratch" workflow (steps 9–10). They are not required for
  the system to run, but omitting them leaves the system driveable yet undocumented for agents.
- `sollertia-forgery` may need new behavior-processing or video-processing branches per system.
- `sollertia-unity-tasks` may need new scene scaffolding if the new system uses Unity.

### Adding a new runtime trial class

**Code touches** — follow the README's "Adding a New Trial Class" recipe:
1. Define the new class in the owning system's `<system>/experiment_configuration.py` as a standalone
   `@dataclass(frozen=True, slots=True)`, prefixing its name with the system name (mirror
   `MesoscopeWaterRewardTrial` / `MesoscopeGasPuffTrial` for shape and naming). The new class
   carries **only** runtime parameters (rewards, durations, thresholds) — no spatial fields. Those
   live on the matching `TrialStructure` inside the paired task template.
2. Export it from the system subpackage's `__init__.py` and the top-level `__init__.py`.
3. Add it to the `trial_structures` union annotation of each `<System>ExperimentConfiguration` that
   uses it (e.g. `dict[str, MesoscopeWaterRewardTrial | MesoscopeGasPuffTrial | <NewTrial>]`).
4. Update that config class's `from_task_template` trigger → trial mapping so the matching
   `TriggerType` instantiates the new class (which may itself be new — see "Adding a new
   `TriggerType` member").

**Skill touches:**

| Skill                                    | What to update                                                                                                                                                                                                                                                                              |
|------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/experiment-configuration`              | The "Templates vs experiment configurations" framing, the trigger → trial-class pairing convention, the `trial_structures` schema description, and the "Common patterns" table                                                                                                              |
| `/task-templates`                        | The trial-class enumeration in the template vocabulary section                                                                                                                                                                                                                              |
| experiment plugin `/vr-driver-interface` | The `DecomposedTrials.trigger_types` semantics table (e.g. `INTERACTION` = reward, `OCCUPANCY_DISARM` = aversive; the `COLLISION` / `OCCUPANCY_ARM` / `OCCUPANCY_TRIGGER` members exist in the enum but Mesoscope-VR leaves them unmapped) and the orchestrator's per-trigger dispatch note |

### Adding a new `TriggerType` member

This skill owns the **Python registry slice** of the cross-cutting recipe. The full extension is
split three ways and each skill owns its slice — apply all three:

| Slice                                                                                             | Owning skill                             |
|---------------------------------------------------------------------------------------------------|------------------------------------------|
| Python `TriggerType` enum + per-supporting-system `from_task_template` branch (this skill, below) | `/library-extension` (this skill)        |
| Hand-authored zone prefab manufacturing                                                           | unity plugin `/zone-prefabs` (Steps 1–6) |
| `CreateTask` pipeline edits + `DeleteProtectedPaths`                                              | unity plugin `/task-generator`           |

The platform `TriggerType` enum carries the full taxonomy — currently five members: `INTERACTION`,
`COLLISION`, `OCCUPANCY_DISARM`, `OCCUPANCY_ARM`, and `OCCUPANCY_TRIGGER` (the C# `ConfigLoader`
accepts all five literals). **System support is a per-system subset**: each acquisition system's
`from_task_template` maps only the members it supports, and may leave the rest unmapped. A new
`TriggerType` member therefore does **not** require a `from_task_template` branch in every system. A 
system that does not support it simply omits the branch, and a config that uses the unmapped member
raises a clear "not mapped to a runtime trial class" error. The Mesoscope-VR system maps `INTERACTION`
(→ `MesoscopeWaterRewardTrial`) and `OCCUPANCY_DISARM` (→ `MesoscopeGasPuffTrial`), and does not map
`COLLISION`, `OCCUPANCY_ARM`, or `OCCUPANCY_TRIGGER`.

**Code touches** — follow the README's "Adding a New Trigger Type" recipe for the Python slice owned here:
1. Append the member to `TriggerType` in `configuration/vr_configuration.py`.
2. For **each system that supports the new member**, update that system's `from_task_template` to add
   the matching `elif trial_structure.trigger_type == TriggerType.<NEW>:` branch, instantiating the
   corresponding runtime trial class (which may itself be new — see "Adding a new runtime trial
   class"). A system that does not support the member adds no branch; its `from_task_template` then
   raises on that member, which is the intended "unsupported on this system" behavior.

**Skill touches** owned here:

| Skill                       | What to update                                                    |
|-----------------------------|-------------------------------------------------------------------|
| `/task-templates`           | The `TriggerType` enumeration sentence and the "primitives" table |
| `/experiment-configuration` | The trigger → trial-class pairing convention                      |

### Adding a new read asset

A **read asset** is metadata the platform reads from an external, human-maintained source (e.g., the
surgery log Google Sheet) and caches on disk as a typed dataclass. The architecture decision is that
every read asset is translated by the acquisition library into a standardized on-disk dataclass, so
downstream consumers (sollertia-forgery) read that dataclass and never touch the external source — the
dataclass is storage-agnostic, and the acquisition library translates any source into it. `SurgeryData`
(the `surgery_data` read asset) is the current example. This applies only to assets the platform
**reads**; assets it only **writes** to an external source (e.g., the water-restriction log) need no
dataclass and no registry entry. Read assets are a maintainer-curated contract surface: each entry is
a durable translation contract, and adding one is a platform-contract decision by the
sollertia-shared-assets maintainers rather than a routine extension.

**Code touches** — follow the README's "Adding a New Read Asset" recipe:
1. Add the `<Asset>Data` dataclass in a new module under `data_classes/` (the contract package; mirror
   `data_classes/surgery_data.py`); export it from `data_classes/__init__.py`. Contract modules export
   plain dataclasses and never consume the dispatch registries.
2. Append the member to `ReadAssets` (`enums.py`).
3. Register the dataclass in `READ_ASSET_REGISTRY` (`registries.py`) under the new key.
4. Run the test suite — `_assert_registry_coverage()` catches a forgotten registry entry at import,
   naming the missing member.

**Skill touches:**

| Skill                                         | What to update                                                                                                                                          |
|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/data-assets`                                | No new skill needed — the generic data-asset tools serve the new asset automatically once registered; add it to that skill's worked examples if notable |
| experiment plugin `/google-sheets-processing` | Add the new asset's reader/translation and the registered dataclass it produces                                                                         |

**Downstream coordination:**
- `sollertia-experiment` owns the reader that translates the external source into the new dataclass and
  caches it on disk. Hand off to the experiment plugin's `/google-sheets-processing`. This skill owns the
  sollertia-shared-assets dataclass and registry entry; `/google-sheets-processing` owns the reader.
- `sollertia-forgery` consumes the cached on-disk dataclass during dataset assembly.

---

## Workflow

### Step 1: Identify the extension scenario

Pick exactly one row from the per-scenario touch lists above. If the change spans multiple
scenarios (e.g., a new acquisition system **and** a new session type), apply the touch lists
sequentially. Do not interleave them — the parity check will reject a half-wired extension on
the next import attempt and the resulting error message points at one registry at a time.

### Step 2: Apply the code touches

Defer to the README sections cited above for `SessionTypes`, `AcquisitionSystems`, and read-asset
extensions. For the trial-class and trigger-type scenarios, follow the touch lists in this skill.
Run the test suite after each scenario's code touches land — `_assert_registry_coverage()` catches
the dispatch-registry side, but required-asset branches need explicit test coverage.

### Step 3: Apply the skill touches

For each entry in the relevant touch list, open the named SKILL.md and update the specified
content. Do **not** rewrite framing that says "currently only X" into "currently only X and Y" —
prefer enumerating the new member alongside the existing one explicitly so the prose stays
honest. The `assets:experiment-configuration`, `assets:session-data`,
`assets:session-descriptors`, `assets:session-hardware-state`, and `assets:task-templates`
skills are the most likely targets; check the others only if your scenario crosses their scope.

### Step 4: Coordinate with downstream libraries

Each scenario lists the downstream libraries that need parallel changes. Do not attempt those
changes from this skill — hand off to the matching environment-setup, configuration, or
processing skill in the affected plugin. List the hand-offs in your PR description so the
reviewer can confirm the cross-repo coordination happened.

### Step 5: Verify

Run the verification checklist below. The library's import-time parity assertion is the safety
net for the dispatch registries; the manual checks cover the trigger → trial mapping, the
required-asset branches, and the skill content.

---

## Pitfalls

| Pitfall                                                      | Why it bites                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|--------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Adding an enum member without registering its dispatch class | `_assert_registry_coverage()` raises at import time, so `slsa mcp` fails to start until every registry is wired. Run `python -c 'import sollertia_shared_assets'` after the code change.                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Forgetting `from_task_template` wiring                       | `from_task_template` is a contract method on every `<System>ExperimentConfiguration`; `_assert_experiment_configuration_contract()` raises at import if it (or a contract field) is missing, so there is no registry to forget. Within `from_task_template`, every `TriggerType` a template carries **on a system that supports it** needs a branch — an unmapped trigger raises rather than being silently dropped. Leaving a member unmapped is a legitimate per-system choice (a system supports only the subset it implements); the raise is then the intended "unsupported on this system" signal, not a wiring bug. |
| Forgetting the descriptor `incomplete` field                 | A new `<Type>Descriptor` that omits `incomplete: bool = True` fails the import via `_assert_descriptor_contract()` (the inspection tooling reads this field). Declare it on every new descriptor.                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Forgetting the required-asset entry                          | The required-asset policy is `SessionData.required_raw_assets`; a session type that runs the corridor task but is missing from `SESSION_TYPES_USING_VR_TASK` is NOT import-checked and will pass `inspect_sessions_tool` even when its VR snapshot is missing. Cover the new type in `tests/data_hierarchy/session_data_test.py`.                                                                                                                                                                                                                                                                                         |
| Updating an existing descriptor schema in place              | Existing on-disk YAMLs can fail to load. Treat schema changes to existing dataclasses as a separate migration concern, coordinated through the library's semantic versioning — not as part of the extension flow this skill covers.                                                                                                                                                                                                                                                                                                                                                                                       |

---

## Related skills

| Skill                                           | Relationship                                                                                                                                            |
|-------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`                 | Run if the parity check fails at import time — the failure manifests as an MCP startup error                                                            |
| `/working-directory`                            | Required prerequisite — bootstraps the working directory consumed by every extension touch-point                                                        |
| `/session-data`                                 | Receives skill touch-ups for new `SessionTypes` and new `AcquisitionSystems`                                                                            |
| `/session-descriptors`                          | Receives skill touch-ups for new `SessionTypes`                                                                                                         |
| `/session-hardware-state`                       | Receives skill touch-ups for new `SessionTypes` and new `AcquisitionSystems`                                                                            |
| `/experiment-configuration`                     | Receives skill touch-ups for new `AcquisitionSystems`, runtime trial classes, and `TriggerType` members                                                 |
| `/task-templates`                               | Receives skill touch-ups for new `TriggerType` and runtime trial classes                                                                                |
| experiment plugin `/acquisition-system-design`  | Owns the runtime-side configuration and binding-class design for any new acquisition system; authors the system's dedicated agentic assets (steps 9–10) |
| experiment plugin `/acquisition-system-runtime` | Owns the runtime that creates and runs sessions of any new session type during acquisition                                                              |
| experiment plugin `/data-management`            | Manages the post-acquisition lifecycle (preprocess, migrate, delete) for sessions of any type                                                           |
| experiment plugin `/google-sheets-processing`   | Owns the reader that translates an external source into a read asset's on-disk dataclass                                                                |
| forging plugin `/behavior-input-format`         | Decides eligibility of new session types for behavior processing                                                                                        |
| forging plugin `/project-manifest`              | Tabulates new session types in the project manifest                                                                                                     |
| forging plugin `/dataset-forging-input-format`  | Decides eligibility of new session types for dataset forging                                                                                            |
| unity plugin `/task-prefabs`                    | Generates Unity prefabs for new `TriggerType` members                                                                                                   |
| unity plugin `/task-scenes`                     | Authors Unity scenes for new acquisition systems                                                                                                        |
| `/commit`                                       | Should be invoked after the cross-cutting changes land                                                                                                  |
| experiment plugin `/vr-driver-interface`        | Consumes the `TriggerType` enum via `DecomposedTrials.trigger_types`                                                                                    |

---

## Verification checklist

```text
Code side:
- [ ] Identified exactly one extension scenario (or applied multiple sequentially)
- [ ] Followed the README recipe for SessionTypes / AcquisitionSystems extensions
- [ ] `python -c 'import sollertia_shared_assets'` succeeds without RuntimeError from the import-time checks in
      `registries.py` (`_assert_registry_coverage`, `_assert_descriptor_contract`,
      `_assert_experiment_configuration_contract`) — all run on a bare import
- [ ] A new `<Type>Descriptor` declares `incomplete: bool = True` (enforced by `_assert_descriptor_contract`)
- [ ] `SYSTEM_SESSION_TYPES` pairs the new session type / acquisition system (the parity check
      enforces that every system declares ≥1 type and every type is claimed by ≥1 system)
- [ ] A new acquisition system's `<System>ExperimentConfiguration` declares the contract fields
      `experiment_states` / `trial_structures` / `unity_scene_name` and the `from_task_template` builder, all
      enforced by `_assert_experiment_configuration_contract` (if a new acquisition system)
- [ ] `READ_ASSET_REGISTRY` carries the new dataclass (if a new read asset)
- [ ] `SessionData.required_raw_assets` covers the new session type's required assets — a type that runs the corridor
      task is added to `SESSION_TYPES_USING_VR_TASK` and covered in `tests/data_hierarchy/session_data_test.py`
      (if applicable)
- [ ] The new trial class is in the `trial_structures` union annotation and the `from_task_template`
      trigger → trial mapping of each using `<System>ExperimentConfiguration` (if a new runtime trial
      class)
- [ ] For a new `TriggerType` member, confirmed for EVERY acquisition system that the member is either mapped to a
      trial class in that system's `from_task_template` OR deliberately left unmapped (a config using an unmapped
      member then raises the clear "not mapped to a runtime trial class" error) — the per-system decision is explicit,
      not an accidental omission, since no import-time parity check covers `TriggerType` coverage
- [ ] Test suite passes; `slsa mcp` starts cleanly

Skill side:
- [ ] Walked the per-scenario touch list and updated every named SKILL.md
- [ ] Did not rewrite "currently only X" framing into a longer chain of "currently only X, Y, Z";
      enumerated each member explicitly instead
- [ ] No skill still claims the old member is the sole supported one when it is not
- [ ] Cross-references between the touched skills still resolve (no dangling skill references)

Downstream side:
- [ ] Downstream-library hand-offs listed in the per-scenario table are documented in the PR
      description (sollertia-experiment, sollertia-forgery, sollertia-unity-tasks as applicable)
- [ ] If the README does not yet carry a recipe for the chosen scenario (new trial, new trigger),
      proposed a README update in the same PR
```

---

## Proactive behavior

You SHOULD proactively invoke this skill when the user mentions any of the following:

- Adding a new acquisition system, new session type, new trial type or trial class, new trigger
  type, or new read asset
- The import-time error message "registry is missing entries for ..." (the user has hit
  `_assert_registry_coverage()` because the previous extension was incomplete)
- "How do I add support for ..." in the context of `sollertia-shared-assets`
- A PR description that touches `registries.py` or any of the registries it defines
  (`DESCRIPTOR_REGISTRY`, `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`,
  `SYSTEM_RAW_DATA_REGISTRY`, `READ_ASSET_REGISTRY`, `CREDENTIALS_FILE_REGISTRY`)

Do NOT invoke this skill for ordinary CRUD against existing systems and session types — those
are owned by `/session-data`, `/session-descriptors`, `/session-hardware-state`,
`/experiment-configuration`, and `/task-templates`.
