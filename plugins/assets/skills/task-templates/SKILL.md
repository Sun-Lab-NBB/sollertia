---
name: task-templates
description: >-
  Authors, modifies, and validates reusable TaskTemplate YAMLs (VR environment, cue catalog,
  trial structures with per-trial cue sequences and zones) via the sollertia-shared-assets MCP
  server. Owns write_template_tool, validate_template_tool, and the schema / trial / trigger-type
  introspection helpers. Use when designing or modifying a task template or preparing it for
  per-project experiment configurations.
user-invocable: true
---

# Sollertia task templates

Authors and modifies reusable `TaskTemplate` YAML files for `sollertia-shared-assets` using the
`slsa mcp` MCP server. This skill is the **exclusive** owner of `write_template_tool`, the
template schema introspection tool, the template validator, and the trial / trigger-type enum
helpers — no other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Authoring `TaskTemplate` YAML files in the configured task templates directory
- The template vocabulary (`Cue`, `TrialStructure`, `VREnvironment`, `TriggerType`)
- Discovering existing templates and reading their contents
- Schema introspection via `describe_template_schema_tool`
- Template validation via `validate_template_tool`
- Enumerating supported trial classes (`list_supported_trial_types_tool`)
- Enumerating supported trigger types (`list_supported_trigger_types_tool`)

**Does not cover:**
- Per-project `MesoscopeExperimentConfiguration` authoring (see `/experiment-configuration`)
- Setting the task templates directory path (see `/working-directory`)
- Generating a Unity task from a template (see unity plugin's `/task-prefabs` —
  `create_task_tool` builds both the task prefab and the matching scene in one call)

---

## What is a task template

A `TaskTemplate` is a **reusable** description of a behavioral paradigm: the VR environment, the cue
catalog, and the trial structures. Each trial owns its own cue sequence, zone geometry, and trigger
type, and is materialized into a single segment prefab named `<template_name>_<trial_name>.prefab`
at generation time. Templates are project-agnostic and acquisition-system-agnostic — the same
template can back many system-specific experiment configurations (currently only
`MesoscopeExperimentConfiguration`, but the `AcquisitionSystems` enum and factory registry are
designed for additional systems) across many projects.

A template defines **what is possible**. An experiment configuration picks a template and parameterizes
it (state durations, trial weights, reward volumes, project-specific overrides). The two are authored by
two different skills with two different ownership scopes.

### Where the template name flows

The template basename anchors a three-tier Unity artifact chain — template → task prefab → scene —
all sharing the same name by convention:

```text
Configurations/<name>.yaml          template (authored here)
        │
        ▼  unity plugin /task-prefabs (create_task_tool)
InfiniteCorridorTask/Tasks/<name>.prefab    task prefab
        │
        ▼  unity plugin /task-scenes (create_task_tool)
Scenes/<name>.unity                 scene (instantiates the task prefab)
```

The canonical reference for this hierarchy is unity plugin's `/task-prefabs`. When you rename a
template, the regenerated task prefab and the next scene created from it inherit the new name; the
old `.prefab` and `.unity` files remain on disk until deleted via `delete_asset_tool`.

---

## Template vocabulary

A `TaskTemplate` is composed of these classes (all defined in `sollertia_shared_assets.configuration`):

| Primitive        | Purpose                                                                                                              |
|------------------|----------------------------------------------------------------------------------------------------------------------|
| `Cue`            | A visual cue (name, uint8 code, length, optional texture) referenced by trial cue sequences                          |
| `TrialStructure` | Per-trial spatial config: cue sequence, optional transitions, stimulus trigger zone, stimulus location, trigger type |
| `VREnvironment`  | VR corridor configuration: spacing, segments per corridor, padding prefab, units, cue offset                         |
| `TriggerType`    | Enum of stimulus trigger zone activators (`lick`, `occupancy`)                                                       |

`TaskTemplate.trial_structures` is a `dict[str, TrialStructure]` keyed by trial name. The template
itself does **not** carry trial weights, experiment-specific reward parameters, or trial-class
choices — those live on the system-specific experiment configuration (e.g.
`MesoscopeExperimentConfiguration`) and are owned by `/experiment-configuration`.
The standalone trial classes `WaterRewardTrial` and `GasPuffTrial` are experiment-scope classes;
they appear in this skill only when you call `list_supported_trial_types_tool` to enumerate what an
experiment configuration may instantiate to pair with a template's `TrialStructure` entries.

For canonical field definitions and valid values, call `describe_template_schema_tool` — do not rely on
handwritten documentation that may drift from the slsa source of truth. For the canonical trial-class
and trigger-type enum values, use `list_supported_trial_types_tool` and
`list_supported_trigger_types_tool` respectively.

---

## How a template drives Unity and the acquisition runtime

A template has two consumers: **Unity** generates the VR environment from it, and the
**acquisition runtime** decomposes the resulting cue sequence back into a trial timeline. The
schema is shaped by what those two consumers need. Unity-side prefab generation, scene loading,
and runtime corridor mechanics are owned by the unity plugin's `/task-prefabs` and `/task-scenes`
skills; trial decomposition and runtime trial state are owned by the experiment library and
documented at the conceptual level in `/experiment-configuration`. The overview below is the
slsa-side conceptual model — defer to those skills for implementation specifics.

### Supported VR paradigm: the infinite corridor

The platform currently supports a single VR topology — the **infinite corridor**. The animal is
head-fixed on a treadmill and runs forward through a one-dimensional corridor whose wall
geometry is built from segment prefabs. The corridor has no end: the entire cue sequence the
animal will see is **pre-resolved at session init** from the template, and the animal then
traverses a deterministic chain of segments without branching, finish lines, or on-the-fly
geometry generation. Every concept in this skill (cues, segments, transition probabilities,
trial structures, `cue_offset_cm`, `segments_per_corridor`) is built around this paradigm.
Future support for additional VR paradigms would require new template classes and corresponding
Unity scaffolding; until then, treat "VR template" and "infinite corridor template" as
synonymous.

### How template fields are used

| Field                                 | Consumer-side role                                                                                                                                                                                                                                                                                               |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **`cues`**                            | Unity bakes wall textures from each cue's `texture` asset; the uint8 `code` is the on-the-wire identifier the runtime uses for analysis.                                                                                                                                                                         |
| **`vr_environment`**                  | Parameterizes corridor geometry: how many segments are visible at once, how parallel corridor instances are spaced, the centimeter↔Unity-unit conversion, the padding prefab, and the cue offset that shifts the cue sequence origin relative to each corridor's spawn point. See `/task-prefabs` for specifics. |
| **`trial_structures`**                | Spatial config per trial type — cue sequence, stimulus trigger zone bounds, stimulus location, visible-boundary flag, trigger type, and optional transitions. The trigger type tells Unity which zone prefab to bake.                                                                                            |
| **`trial_structures[].cue_sequence`** | Drives Unity's segment-prefab geometry: each trial generates a single segment prefab whose cue ordering matches this sequence. Cue prefab lengths sum to the segment length used by zone validation.                                                                                                             |
| **`trial_structures[].transitions`**  | Drives Unity's segment-sequence resolver at session init. Sampled to materialize the deterministic trial chain; null/empty falls back to uniform-random successor selection.                                                                                                                                     |

After Unity emits the materialized cue sequence at session start, the acquisition runtime
**decomposes it back into a trial timeline** by motif-matching each `TrialStructure`'s cue
sequence against the materialized sequence. The trial timeline is what downstream analysis
joins against licks, rewards, and gas puffs. Adding a new trial type to a paradigm therefore
means (a) adding a `TrialStructure` to the template here, and (b) binding it to a concrete
trial subclass in the per-project experiment configuration via `/experiment-configuration`.

### Why the template is shaped this way

- **Cues are a flat catalog with unique uint8 codes** because the analysis pipeline indexes
  wall encounters by code and needs deterministic, low-cost packing. The 0–255 range caps the
  vocabulary at 256 cues per template, which has been more than sufficient in practice.
- **Cue identity `(name, length_cm)` must resolve to one texture across every template** in the
  configured templates directory. Unity stores cue prefabs and materials filesystem-keyed as
  `Cue_<name>_<length>cm.prefab` / `.mat`, and the generator reuses an existing cue prefab whenever
  it finds one on disk. Two templates that declare the same cue identity with different textures
  would therefore silently corrupt each other depending on generation order. The Unity-side
  `CreateFromTemplate` preflight scans every template under `Assets/InfiniteCorridorTask/Configurations/`
  before any mutation and aborts the generation request with the offending template pair(s) listed
  in the error. When authoring a new cue or modifying an existing one, either reuse the same texture
  for the same `(name, length_cm)` everywhere or rename / re-length the cue so it occupies a distinct
  filesystem slot.
- **Each trial structure embeds its own cue sequence** because the Unity task generator derives
  segment prefab geometry directly from the trial's cue sequence — there is no separate segment
  catalog. The segment prefab name is `<template_name>_<trial_name>`, so every trial structure
  yields a distinct prefab keyed by the trial key (no geometric coincidence between two trials can
  cause them to collapse into a single prefab).
- **Transitions are a named dict (`{trial_name: probability}`)** because
  the trial-to-trial topology is the source of truth for corridor sequencing. Names make the
  topology order-independent and self-documenting; omitted keys carry implicit zero probability,
  so the dict need only enumerate reachable trials.
- **`TrialStructure` is spatial-only** (no rewards, no puff durations, no occupancy thresholds)
  because those are project-level behavioral parameters that vary between teams using the same
  paradigm. They live on the experiment-config trial classes (`WaterRewardTrial`, `GasPuffTrial`)
  authored by `/experiment-configuration` and are joined to the spatial structure by trial name.
- **`trigger_type` is on `TrialStructure`** because Unity must pick the zone prefab during
  template generation — long before any experiment-config trial is instantiated. The trigger
  type is therefore the template's contract with Unity; the matching experiment-config trial
  class is the experiment config's contract with the runtime's stimulus delivery code.
- **`cue_offset_cm` lives on `vr_environment`** because the cue-origin shift is an attribute of
  the corridor geometry itself — every Unity-spawned corridor instance sees the same value. It
  also controls the per-segment ResetZone placement: the ResetZone sits at segment-local
  `z = cue_offset_cm / cm_per_unity_unit` so the animal's spawn point (world z = 0) falls inside
  the zone on every lap restart.

---

## MCP tool surface

| Tool                                  | Purpose                                                                       |
|---------------------------------------|-------------------------------------------------------------------------------|
| `discover_templates_tool`             | Lists all task templates in the configured templates directory                |
| `read_template_tool`                  | Reads an existing template at an explicit path                                |
| `write_template_tool`                 | Writes a new template or overwrites an existing one (exclusive to this skill) |
| `describe_template_schema_tool`       | Returns the field schema for `TaskTemplate` (exclusive to this skill)         |
| `validate_template_tool`              | Validates a template against its schema and cross-reference constraints       |
| `list_supported_trial_types_tool`     | Enumerates trial classes supported by experiment configurations (exclusive)   |
| `list_supported_trigger_types_tool`   | Enumerates the `TriggerType` enum values (exclusive)                          |

`read_template_tool`, `write_template_tool`, and `validate_template_tool` take an explicit
`file_path` — path resolution is the caller's responsibility. The canonical home for templates is
the directory set via `/working-directory`'s `set_task_templates_directory_tool`, and
`discover_templates_tool()` returns each template's absolute path, which is what you pass to the
other three tools.

---

## Authoring workflow

### Step 1: Verify prerequisites

- The `sollertia-shared-assets` MCP server is connected (else hand off to `/assets-mcp-environment-setup`).
- The task templates directory is set (else hand off to `/working-directory`).

### Step 2: Discover existing templates

```text
discover_templates_tool()
```

If a similar template already exists, prefer reading it (`read_template_tool(file_path=...)`) and
modifying a copy. Avoid creating near-duplicates. The response includes each template's absolute
`path` — capture it to pass into `read_template_tool` / `write_template_tool` /
`validate_template_tool`.

### Step 3: Inspect the schema and enums

```text
describe_template_schema_tool()
list_supported_trial_types_tool()
list_supported_trigger_types_tool()
```

Use the schema and enum lists as the source of truth — the enum tools pin down the exact strings
accepted by `trigger_type` and the exact class names used for the `GasPuffTrial` and
`WaterRewardTrial` variants. This avoids silent typos that slip past YAML syntax but fail at runtime.

### Step 4: Author the template

Build the template dictionary in this order:

1. **VR environment** — define the `VREnvironment` (corridor spacing, segments per corridor, padding
   prefab name, cm-per-unity-unit conversion, cue offset).
2. **Cue catalog** — define every `Cue` (name, uint8 code, length, optional texture filename).
3. **Trial structures** — populate the `trial_structures` dict, mapping each trial name to a
   `TrialStructure` (cue sequence, stimulus trigger zone start/end, stimulus location, whether the
   collision boundary is visible, trigger type from `list_supported_trigger_types_tool`, and an
   optional `transitions` dict mapping target trial names to probabilities summing to 1.0).

Trial weights, reward sizes, gas-puff durations, occupancy thresholds, experiment states, and the
choice of trial class (`WaterRewardTrial` vs `GasPuffTrial`) are **not** part of the template —
they are added per-experiment by `/experiment-configuration`.

### Step 5: Write, validate, and re-read

Use `read_task_templates_directory_tool` (from `/working-directory`) or `discover_templates_tool`
to learn the canonical templates directory, then construct the destination file path as
`<templates-directory>/<template-name>.yaml`:

```text
write_template_tool(
    file_path="<templates-directory>/<template-name>.yaml",
    template_payload={ ... full nested dict ... },
    overwrite=False,
)
validate_template_tool(file_path="<templates-directory>/<template-name>.yaml")
read_template_tool(file_path="<templates-directory>/<template-name>.yaml")
```

The kwargs are `file_path` and `template_payload` (the latter accepts a JSON-friendly dict).
Pass `overwrite=True` only when intentionally replacing an existing template.

`validate_template_tool` loads the template through `TaskTemplate.from_yaml`, which triggers
`__post_init__` validation:

- cue codes are unique
- cue codes are in `[0, 255]`
- cue names are unique
- each trial name matches `^[A-Za-z0-9_]+$` (used verbatim in the Unity-side
  `<template>_<trial>.prefab` segment filename)
- each trial `cue_sequence` is non-empty and references valid cue names
- each trial `transitions`, when provided, sums to 1.0 and references valid trial names
- each `TrialStructure.trigger_type` is a valid `TriggerType` value
- per-trial zone positions satisfy `start ≤ end`, `stimulus_location_cm ≥ start`, and all three are
  within the trial's segment length

On success the tool returns a `summary` (cue/trial counts plus `cue_offset_cm`); on failure it
returns an `issues` list. Fix any reported issues and re-write before handing off.

### Step 6: Hand off for Unity task creation

When the template targets a Unity scene, hand off to the **unity plugin's `/task-prefabs`** and
run `create_task_tool(template_name="<name>")`. The tool builds the task prefab and the matching
scene in one call from the same template basename — both artifacts then live at
`Assets/InfiniteCorridorTask/Tasks/<name>.prefab` and `Assets/Scenes/<name>.unity`. If the Unity
Editor, McpBridge, or `slsa mcp` is unreachable, restore connectivity via
`/unity-mcp-environment-setup` or `/assets-mcp-environment-setup` before proceeding.

### Step 7: Hand off for experiment configuration

Once the template is written and validated, projects that want to use it must author per-project
`MesoscopeExperimentConfiguration` files via `/experiment-configuration`. This skill is not responsible
for instantiating templates into experiment configurations.

---

## Common patterns

### Add a new trial entry to an existing paradigm

1. Read the template with `read_template_tool`.
2. Confirm the trigger type via `list_supported_trigger_types_tool`.
3. Add a new `TrialStructure` entry to the `trial_structures` dict, keyed by trial name. Specify the
   `cue_sequence`, the stimulus zone bounds, the `trigger_type`, and (if other trials should route
   to this one or vice versa) update the relevant `transitions` dicts.
4. Write the template back with `write_template_tool` (use `overwrite=True`).
5. Re-run `validate_template_tool` to confirm the new entry passes cross-reference checks.
6. Hand off to `/experiment-configuration` if a per-project experiment needs to pair this trial with
   a `WaterRewardTrial` or `GasPuffTrial` runtime class and assign weights.

### Migrate a template to a new VR scene

1. Read the template with `read_template_tool`.
2. Mutate the `vr_environment` fields (corridor spacing, segments per corridor, padding prefab,
   cm-per-unity-unit) to match the new scene's geometry. The Unity scene name itself is bound by
   per-project experiment configurations, not by the template.
3. Run `validate_template_tool` to catch structural regressions.
4. Hand off to unity plugin's `/task-prefabs` to regenerate the prefab and re-verify segment zones
   against the new prefab state.

### Audit which projects use a template

1. List templates with `discover_templates_tool`.
2. Hand off to `/experiment-configuration` to enumerate consuming experiments per project.

---

## Verification checklist

```text
- [ ] /working-directory has been run and the templates directory is set
- [ ] discover_templates_tool was called before creating a new template (avoid duplicates)
- [ ] describe_template_schema_tool was used as the source of truth for field names
- [ ] list_supported_trial_types_tool / list_supported_trigger_types_tool consulted for enum values
- [ ] file_path was constructed as <templates-directory>/<template-name>.yaml (absolute path)
- [ ] Payload was passed as template_payload (the correct kwarg name)
- [ ] write_template_tool succeeded without schema errors
- [ ] validate_template_tool returned valid=True with no issues
- [ ] read_template_tool returned the expected content after the write
- [ ] If the template targets a Unity scene, unity plugin's /task-prefabs was invoked for prefab
      generation and validation
```

---

## Related skills

| Skill                                  | Relationship                                                                                                |
|----------------------------------------|-------------------------------------------------------------------------------------------------------------|
| `/working-directory`                   | Required prerequisite — owns the templates directory path                                                   |
| `/assets-mcp-environment-setup`        | Run first if the MCP server is not connected                                                                |
| `/experiment-configuration`            | Consumer — instantiates templates into per-project experiments                                              |
| `/library-extension`                   | Cross-cutting recipe to add a new `TriggerType`, runtime trial class, or VR paradigm beyond the corridor    |
| unity plugin `/task-prefabs`           | Downstream — generates and validates the Unity prefab                                                       |
| unity plugin `/task-scenes`                 | Downstream — places the generated prefab into a Unity scene                                                 |
