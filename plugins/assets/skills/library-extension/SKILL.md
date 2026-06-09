---
name: library-extension
description: >-
  Orchestrates cross-cutting changes to extend the sollertia-shared-assets library with a new
  acquisition system, session type, trial class, trigger type, or VR paradigm. Covers the
  registry touch list, import-time parity check, and sibling-skill enumeration updates. Use
  when adding a new AcquisitionSystems / SessionTypes / TriggerType member, runtime trial
  class, or extending the template vocabulary beyond the infinite corridor.
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
- Adding a new `TriggerType` member (and the trigger → trial-class pairing in
  `MesoscopeExperimentConfiguration.from_task_template`)
- Extending the template vocabulary beyond the **infinite corridor** VR paradigm
- Adding a new `ReadAssets` member (with its on-disk dataclass and `READ_ASSET_REGISTRY` entry) — the
  sollertia-shared-assets contract for an external asset the platform reads and caches on disk
- The cross-skill touch list — which other plugin skills carry hardcoded enumerations that drift
  the moment a new enum member is added
- Coordination with downstream libraries (sollertia-experiment, sollertia-forgery)

**Does not cover:**
- The line-by-line code changes themselves. Those are owned by the
  **`sollertia-shared-assets` README** sections "Adding New Session Types" and "Adding New
  Acquisition Systems" — this skill defers to the README as the authoritative recipe.
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

`sollertia-shared-assets` is structured around five **dispatch registries**. Each one is keyed by a
member of the `SessionTypes`, `AcquisitionSystems`, or `ReadAssets` enum, and each one resolves a
string identifier to a Python class that the MCP tools use to parse, validate, or build the
corresponding asset. A separate **association**,
`SYSTEM_SESSION_TYPES`, records which session types each acquisition system can run; it is keyed by
`AcquisitionSystems` but maps to a `frozenset` of `SessionTypes` rather than a dispatch class, and
`SessionData.create()` rejects any session-type / acquisition-system pairing it does not contain.

| Registry                            | File                                       | Keyed by             | Maps to                                        |
|-------------------------------------|--------------------------------------------|----------------------|------------------------------------------------|
| `DESCRIPTOR_REGISTRY`               | `interfaces/mcp_instance.py`               | `SessionTypes`       | Per-session-type descriptor dataclass          |
| `HARDWARE_STATE_REGISTRY`           | `interfaces/mcp_instance.py`               | `AcquisitionSystems` | Per-system hardware-state dataclass            |
| `EXPERIMENT_CONFIGURATION_REGISTRY` | `configuration/configuration_utilities.py` | `AcquisitionSystems` | Per-system experiment-configuration dataclass  |
| `SYSTEM_RAW_DATA_REGISTRY`          | `data_classes/session_data.py`             | `AcquisitionSystems` | Per-system raw-data sub-dataclass with `build` |
| `SYSTEM_SESSION_TYPES`              | `data_classes/session_data.py`             | `AcquisitionSystems` | `frozenset[SessionTypes]` the system can run   |
| `READ_ASSET_REGISTRY`               | `data_classes/read_assets.py`              | `ReadAssets`         | Per-read-asset on-disk dataclass               |

Every `<System>ExperimentConfiguration` shares one contract: an `experiment_states` field (a mapping
of `ExperimentState` — the experiment state machine; every experiment is a state machine, so this is
required) and a `trial_structures` field (the trials the experiment runs — required; the concrete
trial classes vary per system). Fields beyond the contract are system-specific. For example
`unity_scene_name` is added by systems that use Unity VR tasks.
Mesoscope-VR is one exemplar of the contract: its `WaterRewardTrial` / `GasPuffTrial` trial classes
and its `unity_scene_name` are exemplar-specific, and another system has its own trial classes and may
add different fields.

A system's trial classes are introspected from its experiment configuration's `trial_structures`
field via the `collect_field_dataclasses` helper in
`interfaces/mcp_instance.py`. `list_supported_trial_types_tool(acquisition_system)` resolves the
system's experiment-configuration class and derives the trial vocabulary from that field;
`MesoscopeExperimentConfiguration.from_task_template` maps each `TriggerType` to its runtime trial
class. The trial classes themselves are standalone dataclasses in
`configuration/experiment_configuration.py`. `VR_TEMPLATE_CONFIG_REGISTRY`
(`configuration/configuration_utilities.py`) maps each acquisition system that builds its configuration
from a Unity VR task template to that configuration class, and `create_experiment_from_vr_template_tool`
dispatches through it. This registry is optional and partial: only a system that uses Unity VR tasks
registers here, and it sits outside the import-time parity check.

### The parity check

`interfaces/mcp_instance.py` runs `_assert_registry_coverage()` at import time. The function
walks `(DESCRIPTOR_REGISTRY, HARDWARE_STATE_REGISTRY, EXPERIMENT_CONFIGURATION_REGISTRY,
SYSTEM_RAW_DATA_REGISTRY, READ_ASSET_REGISTRY)` and raises `RuntimeError` if any enum member is
missing its dispatch class. It additionally checks `SYSTEM_SESSION_TYPES`: every acquisition system
must declare at least one session type, and every session type must be claimed by at least one
system. Importing the package (or starting `slsa mcp`) fails fast and names the offending registry,
so an incomplete extension cannot silently slip through.

---

## Authoritative code recipes

The `sollertia-shared-assets` README owns the canonical, line-numbered recipes. **Do not retype
them here** — read them, then come back for the cross-skill update map below.

| Scenario                        | README section                       |
|---------------------------------|--------------------------------------|
| New `SessionTypes` member       | "Adding New Session Types"           |
| New `AcquisitionSystems` member | "Adding New Acquisition Systems"     |
| New read asset                  | "Adding a New Read Asset"            |

For new trial classes, new trigger types, and new VR paradigms, the README does not currently
carry a step-by-step recipe. Use the **per-scenario touch lists** below as the working spec, then
propose a README update in the same PR so the recipe lands next to the existing two.

---

## Per-scenario touch lists

Each scenario lists the code touches (defer to the README where one exists), the **skill
touches** (which is what this skill uniquely owns), and the downstream-library coordination.

### Adding a new `SessionTypes` member

**Code touches** — follow the README's "Adding New Session Types" recipe:
1. Append the member to `SessionTypes` (`data_classes/session_data.py`), and add it to the
   `SYSTEM_SESSION_TYPES` frozenset of every acquisition system that can run it (same file). The
   parity check fails if the new type is claimed by no system, and `SessionData.create()` rejects
   it for any system whose set omits it.
2. Add the `<Type>Descriptor` dataclass in the runtime-data module of the system that runs the new type (the
   Mesoscope-VR descriptors live in `data_classes/mesoscope_runtime_data.py`); export it from
   `data_classes/__init__.py`.
3. Register the descriptor in `DESCRIPTOR_REGISTRY` (`interfaces/mcp_instance.py`).
4. If the new type has required raw assets beyond `session_descriptor.yaml` and
   `system_configuration.yaml`, extend `_required_asset_inventory` in `interfaces/data_tools.py`.
   The existing `MESOSCOPE_EXPERIMENT` branch requires BOTH `experiment_configuration.yaml`
   AND `vr_configuration.yaml`; any new session type that uses Unity VR tasks should mirror both
   appends.
5. Run the test suite — `_assert_registry_coverage()` catches a forgotten descriptor entry or an
   unclaimed session type, but a forgotten required-asset branch will not surface until
   `inspect_sessions_tool` reports the wrong `issues` list.

**Skill touches** — update each of the following so its hardcoded enumeration matches the new
member:

| Skill                       | What to update                                                                                                                                                                                       |
|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`             | The `SessionTypes` enumeration sentence under "SessionTypes"; the required-assets paragraph that singles out `mesoscope experiment` if the new type also needs the experiment configuration snapshot |
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
1. Append the member to `AcquisitionSystems` (`configuration/configuration_utilities.py`).
2. Add `<System>HardwareState` in a new `data_classes/<system>_runtime_data.py` module (the Mesoscope-VR classes
   live in `data_classes/mesoscope_runtime_data.py`; the same module later holds the system's descriptors), and add
   `<System>RawData` in `data_classes/session_data.py`; export them all from `data_classes/__init__.py`.
3. Add `<System>ExperimentConfiguration` (a new module under `configuration/`); export it from
   `configuration/__init__.py`.
4. Register the dataclasses in `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`,
   and `SYSTEM_RAW_DATA_REGISTRY`, and add a `SYSTEM_SESSION_TYPES` entry mapping the new system to
   the `frozenset` of `SessionTypes` it can run (declare at least one, or the parity check fails).
5. Give the new system a creation path for its experiment configuration. If it runs Unity VR tasks,
   add a `from_task_template` classmethod to its `<System>ExperimentConfiguration` (mirror
   `MesoscopeExperimentConfiguration.from_task_template`) and register the configuration class under the
   system's `AcquisitionSystems` key in `VR_TEMPLATE_CONFIG_REGISTRY`
   (`configuration/configuration_utilities.py`). The shared `create_experiment_from_vr_template_tool`
   dispatches through that registry to the registered class's `from_task_template`, so no new tool is
   needed. If it builds its configuration from different inputs, add a dedicated creation tool plus a
   matching creation classmethod on its config dataclass. Either way,
   `write_experiment_configuration_tool` already authors the system's full payload generically.
6. Run the test suite — `_assert_registry_coverage()` catches the dispatch registries and the
   `SYSTEM_SESSION_TYPES` pairing. `VR_TEMPLATE_CONFIG_REGISTRY` is optional and partial, so the parity
   check leaves it alone; a system that uses Unity VR tasks creates its configuration through
   `create_experiment_from_vr_template_tool` once it implements `from_task_template` and registers its
   configuration class in `VR_TEMPLATE_CONFIG_REGISTRY`.

**Skill touches** — update each of the following so its hardcoded "currently only Mesoscope-VR"
framing reflects the new member:

| Skill                       | What to update                                                                                                                                                                                                                                                                                                                                                                                                                             |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/session-data`             | The `instance.system_raw_data` bullet under "Path-resolution sub-dataclasses on `SessionData`" — add the new `<System>RawData` field list; the `Mesoscope-VR` mention in "Does not cover" if the experiment plugin's `/mesoscope-vr-snapshots` becomes one of several owners of system-specific snapshots                                                                                                                                  |
| `/session-hardware-state`   | The frontmatter description, the "currently the only concrete subclass is `MesoscopeHardwareState`" prose, and the "Per-session-type field population (Mesoscope-VR example)" framing — clone the table format for the new system                                                                                                                                                                                                          |
| `/experiment-configuration` | The frontmatter description, the "currently only `MesoscopeExperimentConfiguration`" prose, and any per-trial-class assumptions specific to the Mesoscope-VR rig. A system that uses Unity VR tasks reuses `create_experiment_from_vr_template_tool` (note it gains a `from_task_template` classmethod and a `VR_TEMPLATE_CONFIG_REGISTRY` entry); a system with different inputs gains its own creation tool — call out which one applies |
| `/task-templates`           | The "currently only `MesoscopeExperimentConfiguration`" mention                                                                                                                                                                                                                                                                                                                                                                            |

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

**Code touches:**
1. Define the new class in `configuration/experiment_configuration.py` as a standalone
   `@dataclass(frozen=True, slots=True)` (mirror `WaterRewardTrial` / `GasPuffTrial` for shape). The new class
   carries **only** runtime parameters (rewards, durations, thresholds) — no spatial fields. Those
   live on the matching `TrialStructure` inside the paired task template.
2. Export it from `configuration/__init__.py`.
3. Add it to the `trial_structures` union annotation of each `<System>ExperimentConfiguration` that
   uses it (e.g. `dict[str, WaterRewardTrial | GasPuffTrial | <NewTrial>]`).
4. Update that config class's `from_task_template` trigger → trial mapping so the matching
   `TriggerType` instantiates the new class (which may itself be new — see "Adding a new
   `TriggerType` member").

**Skill touches:**

| Skill                                    | What to update                                                                                                                                                                 |
|------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/experiment-configuration`              | The "Templates vs experiment configurations" framing, the trigger → trial-class pairing convention, the `trial_structures` schema description, and the "Common patterns" table |
| `/task-templates`                        | The trial-class enumeration in the template vocabulary section                                                                                                                 |
| experiment plugin `/vr-driver-interface` | The `DecomposedTrials.trigger_types` semantics table (e.g. `LICK` = reward, `OCCUPANCY` = aversive) and the orchestrator's per-trigger dispatch note                           |

### Adding a new `TriggerType` member

This skill owns the **Python registry slice** of the cross-cutting recipe. The full extension is
split three ways and each skill owns its slice — apply all three:

| Slice                                                                       | Owning skill                          |
|-----------------------------------------------------------------------------|---------------------------------------|
| Python `TriggerType` enum + `from_task_template` branch (this skill, below) | `/library-extension` (this skill)     |
| Hand-authored zone prefab manufacturing                                     | unity plugin `/zone-prefabs` (Step 7) |
| `CreateTask` pipeline edits + `DeleteProtectedPaths`                        | unity plugin `/task-generator`        |

**Code touches** owned here:
1. Append the member to `TriggerType` in `configuration/vr_configuration.py`.
2. Update `MesoscopeExperimentConfiguration.from_task_template` in
   `configuration/mesoscope_configuration.py` to add the matching `elif trial_structure.trigger_type
   == TriggerType.<NEW>:` branch, instantiating the corresponding runtime trial class (which may
   itself be new — see "Adding a new runtime trial class").

**Skill touches** owned here:

| Skill                       | What to update                                                          |
|-----------------------------|-------------------------------------------------------------------------|
| `/task-templates`           | The `TriggerType` enumeration sentence and the "primitives" table       |
| `/experiment-configuration` | The trigger → trial-class pairing convention                            |

### Extending the VR paradigm beyond the infinite corridor

The platform currently supports a single VR topology — the **infinite corridor**. Extending
beyond it is a code-driven change: it requires new template classes, new `VREnvironment` fields
(or a sibling environment class), new Unity scaffolding, and new runtime mechanics. The skill
touches and downstream coordination below are the working spec for it.

**Skill touches:**

| Skill                       | What to update                                                                                                                       |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| `/task-templates`           | The "Supported VR paradigm: the infinite corridor" framing and the "How a template drives Unity and the acquisition runtime" section |
| `/experiment-configuration` | The trial-decomposition description (motif matching against a deterministic cue sequence is corridor-specific)                       |

Defer the runtime and Unity work to the experiment plugin and the unity plugin respectively;
this skill only surfaces the sollertia-shared-assets schema and skill touches.

### Adding a new read asset

A **read asset** is metadata the platform reads from an external, human-maintained source (e.g., the
surgery log Google Sheet) and caches on disk as a typed dataclass. The architecture decision is that
every read asset is translated by the acquisition library into a standardized on-disk dataclass, so
downstream consumers (sollertia-forgery) read that dataclass and never touch the external source — the
dataclass is storage-agnostic, and the acquisition library translates any source into it. `SurgeryData`
(the `surgery_data` read asset) is the current example. This applies only to assets the platform
**reads**; assets it only **writes** to an external source (e.g., the water-restriction log) need no
dataclass and no registry entry.

**Code touches** — follow the README's "Adding a New Read Asset" recipe:
1. Add the `<Asset>Data` dataclass inheriting `YamlConfig` (mirror `data_classes/surgery_data.py`);
   export it from `data_classes/__init__.py`.
2. Append the member to `ReadAssets` (`data_classes/read_assets.py`).
3. Register the dataclass in `READ_ASSET_REGISTRY` (same file) under the new key.
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

Defer to the README sections cited above for `SessionTypes` and `AcquisitionSystems` extensions.
For the other three scenarios, follow the touch lists in this skill. Run the test suite after
each scenario's code touches land — `_assert_registry_coverage()` catches the dispatch-registry
side, but required-asset branches need explicit test coverage.

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

| Pitfall                                                              | Why it bites                                                                                                                                                                                                                                                                                       |
|----------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Adding an enum member without registering its dispatch class         | `_assert_registry_coverage()` raises at import time, so `slsa mcp` fails to start until every registry is wired. Run `python -c 'import sollertia_shared_assets'` after the code change.                                                                                                           |
| Forgetting `from_task_template`                                      | A system that uses Unity VR tasks creates its config through `create_experiment_from_vr_template_tool` once it implements `from_task_template` and registers its configuration class in `VR_TEMPLATE_CONFIG_REGISTRY`. That registry is optional and partial, so the parity check leaves it alone. |
| Forgetting `_required_asset_inventory`                               | A new session type that needs an extra raw asset will pass `inspect_sessions_tool` even when that asset is missing on disk. Caught by tests, not by the parity check.                                                                                                                              |
| Updating the descriptor schema without bumping the dataclass version | Existing on-disk YAMLs can fail to load. Treat schema changes to existing dataclasses as a separate migration concern — not as part of the extension flow this skill covers.                                                                                                                       |
| Treating a new VR paradigm as a registry-backed extension            | VR topologies are extended through new template classes and Unity scaffolding, so the skill touches and downstream coordination are larger than for the registry-backed scenarios; budget accordingly.                                                                                             |

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
| `/task-templates`                               | Receives skill touch-ups for new `TriggerType`, runtime trial classes, and VR paradigm extensions                                                       |
| experiment plugin `/acquisition-system-design`  | Owns the runtime-side configuration and binding-class design for any new acquisition system; authors the system's dedicated agentic assets (steps 9–10) |
| experiment plugin `/acquisition-system-runtime` | Owns the runtime that creates and runs sessions of any new session type during acquisition                                                              |
| experiment plugin `/data-management`            | Manages the post-acquisition lifecycle (preprocess, migrate, delete) for sessions of any type                                                           |
| experiment plugin `/google-sheets-processing`   | Owns the reader that translates an external source into a read asset's on-disk dataclass                                                                |
| forging plugin `/behavior-input-format`         | Decides eligibility of new session types for behavior processing                                                                                        |
| forging plugin `/project-manifest`              | Tabulates new session types in the project manifest                                                                                                     |
| forging plugin `/dataset-forging-input-format`  | Decides eligibility of new session types for dataset forging                                                                                            |
| unity plugin `/task-prefabs`                    | Generates Unity prefabs for new `TriggerType` members or VR paradigms                                                                                   |
| unity plugin `/task-scenes`                     | Authors Unity scenes for new acquisition systems or VR paradigms                                                                                        |
| `/commit`                                       | Should be invoked after the cross-cutting changes land                                                                                                  |
| experiment plugin `/vr-driver-interface`        | Consumes the `TriggerType` enum via `DecomposedTrials.trigger_types`                                                                                    |

---

## Verification checklist

```text
Code side:
- [ ] Identified exactly one extension scenario (or applied multiple sequentially)
- [ ] Followed the README recipe for SessionTypes / AcquisitionSystems extensions
- [ ] `python -c 'import sollertia_shared_assets'` succeeds without RuntimeError from
      `_assert_registry_coverage()`
- [ ] `SYSTEM_SESSION_TYPES` pairs the new session type / acquisition system (the parity check
      enforces that every system declares ≥1 type and every type is claimed by ≥1 system)
- [ ] A new acquisition system that uses Unity VR tasks implements `from_task_template` on its
      `<System>ExperimentConfiguration` and registers that class in `VR_TEMPLATE_CONFIG_REGISTRY`, or a
      system with different inputs has a dedicated creation tool + classmethod (if a new acquisition
      system)
- [ ] `READ_ASSET_REGISTRY` carries the new dataclass (if a new read asset)
- [ ] `_required_asset_inventory` covers the new session type's required assets (if applicable)
- [ ] The new trial class is in the `trial_structures` union annotation and the `from_task_template`
      trigger → trial mapping of each using `<System>ExperimentConfiguration` (if a new runtime trial
      class)
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
- [ ] If the README does not yet carry a recipe for the chosen scenario (new trial, new trigger,
      new VR paradigm), proposed a README update in the same PR
```

---

## Proactive behavior

You SHOULD proactively invoke this skill when the user mentions any of the following:

- Adding a new acquisition system, new session type, new trial type or trial class, new trigger
  type, new read asset, or extending the VR paradigm
- The import-time error message "registry is missing entries for ..." (the user has hit
  `_assert_registry_coverage()` because the previous extension was incomplete)
- "How do I add support for ..." in the context of `sollertia-shared-assets`
- A PR description that touches one of the registries (`DESCRIPTOR_REGISTRY`,
  `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`, `SYSTEM_RAW_DATA_REGISTRY`,
  `READ_ASSET_REGISTRY`)

Do NOT invoke this skill for ordinary CRUD against existing systems and session types — those
are owned by `/session-data`, `/session-descriptors`, `/session-hardware-state`,
`/experiment-configuration`, and `/task-templates`.
