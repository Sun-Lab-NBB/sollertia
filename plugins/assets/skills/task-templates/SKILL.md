---
name: task-templates
description: >-
  Authors, modifies, and validates reusable TaskTemplate YAML files in the task templates directory
  via the slsa MCP server. Covers VR environment definition, cue catalogs, segment composition,
  trial primitives, trial structure, the schema and validation tools, and the trial/trigger-type
  enum introspection helpers. Use when designing a new behavioral task template, modifying an
  existing template, validating a template, or preparing templates for use by per-project
  experiment configurations.
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
- The template vocabulary (`Cue`, `Segment`, `TrialStructure`, `VREnvironment`, `TriggerType`)
- Discovering existing templates and reading their contents
- Schema introspection via `describe_template_schema_tool`
- Template validation via `validate_template_tool`
- Enumerating supported trial classes (`list_supported_trial_types_tool`)
- Enumerating supported trigger types (`list_supported_trigger_types_tool`)

**Does not cover:**
- Per-project `MesoscopeExperimentConfiguration` authoring (see `/experiment-configuration`)
- Setting the task templates directory path (see `/working-directory`)
- Generating a Unity task prefab from a template (see unity plugin's `/task-prefabs`)
- Verifying template values against the actual Unity prefab state (see unity plugin's
  `/task-prefabs` — `validate_prefab_against_template_tool`)

---

## What is a task template

A `TaskTemplate` is a **reusable** description of a behavioral paradigm: the VR environment, the cue
catalog, the segment layout, the available trial types, and the trial structure. Templates are
project-agnostic and acquisition-system-agnostic — the same template can back many
system-specific experiment configurations (currently only `MesoscopeExperimentConfiguration`, but
the `AcquisitionSystems` enum and factory registry are designed for additional systems) across many
projects.

A template defines **what is possible**. An experiment configuration picks a template and parameterizes
it (state durations, trial weights, reward volumes, project-specific overrides). The two are authored by
two different skills with two different ownership scopes.

---

## Template vocabulary

A `TaskTemplate` is composed of these classes (all defined in `sollertia_shared_assets.configuration`):

| Primitive        | Purpose                                                                                   |
|------------------|-------------------------------------------------------------------------------------------|
| `Cue`            | A visual cue (name, uint8 code, length, optional texture) referenced by segments          |
| `Segment`        | A spatial region: cue sequence + optional transition probabilities to other segments      |
| `TrialStructure` | Per-trial spatial config: segment, stimulus trigger zone, stimulus location, trigger type |
| `VREnvironment`  | VR corridor configuration: spacing, segments per corridor, padding prefab, units          |
| `TriggerType`    | Enum of stimulus trigger zone activators (`lick`, `occupancy`)                            |

`TaskTemplate.trial_structures` is a `dict[str, TrialStructure]` keyed by trial name. The template
itself does **not** carry trial weights, experiment-specific reward parameters, or trial-class
choices — those live on the system-specific experiment configuration (e.g.
`MesoscopeExperimentConfiguration`) and are owned by `/experiment-configuration`.
The concrete trial classes `WaterRewardTrial` and `GasPuffTrial` (both subclasses of the abstract
`BaseTrial`) are experiment-scope classes; they appear in this skill only when you call
`list_supported_trial_types_tool` to enumerate what an experiment configuration may instantiate from
a template's `TrialStructure` entries. `BaseTrial` itself is not returned by the tool — it is the
shared parent, not an instantiable trial type.

For canonical field definitions and valid values, call `describe_template_schema_tool` — do not rely on
handwritten documentation that may drift from the slsa source of truth. For the canonical trial-class
and trigger-type enum values, use `list_supported_trial_types_tool` and
`list_supported_trigger_types_tool` respectively.

---

## How a template drives Unity and the acquisition runtime

A template has two consumers: **Unity** generates the VR environment from it, and the
**acquisition runtime** decomposes the resulting cue sequence back into a trial timeline. The
schema is shaped by what those two consumers need. Unity-side prefab generation, scene loading,
and runtime corridor mechanics are owned by the unity plugin's `/task-prefabs` and `/scenes`
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

| Field                                      | Consumer-side role                                                                                                                                                                                               |
|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **`cues`**                                 | Unity bakes wall textures from each cue's `texture` asset; the uint8 `code` is the on-the-wire identifier the runtime uses for analysis.                                                                         |
| **`segments`**                             | Unity prefab building blocks. Each segment maps to a Unity prefab whose name matches `segment.name`.                                                                                                             |
| **`segments[i].transition_probabilities`** | Drive Unity's segment-sequence resolver at session init. Sampled to materialize the deterministic segment chain; null/empty falls back to uniform-random successor selection.                                    |
| **`vr_environment`**                       | Parameterizes corridor geometry: how many segments are visible at once, how parallel corridor instances are spaced, the centimeter↔Unity-unit conversion, the padding prefab. See `/task-prefabs` for specifics. |
| **`trial_structures`**                     | Spatial zone definitions per trial type — segment binding, stimulus trigger zone bounds, stimulus location, visible-boundary flag, and trigger type. The trigger type tells Unity which zone prefab to bake.     |
| **`cue_offset_cm`**                        | Shifts the cue sequence origin relative to the animal's per-corridor spawn point so cue alignment is preserved as the runtime advances through the corridor sequence.                                            |

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
- **Segments are reusable, prefab-aligned units** because Unity needs concrete prefabs to
  instantiate, and reusing the same prefab across trial types is how multi-segment paradigms
  share geometry.
- **Transition probabilities live on segments, not on trials** because the corridor topology is
  what generates the trial sequence. Putting the probabilities at the trial level would force
  the runtime to know about trials before the cue sequence is even emitted, which inverts the
  Unity → acquisition-runtime data flow.
- **`TrialStructure` is spatial-only** (no rewards, no puff durations, no occupancy thresholds)
  because those are project-level behavioral parameters that vary between teams using the same
  paradigm. They live on the experiment-config trial subclasses authored by
  `/experiment-configuration`.
- **`trigger_type` is on `TrialStructure`** because Unity must pick the zone prefab during
  template generation — long before any experiment-config trial subclass exists. The trigger
  type is therefore the template's contract with Unity; the per-trial subclass is the
  experiment config's contract with the runtime's stimulus delivery code.
- **`cue_offset_cm` is on the template (not the experiment config)** because the cue-origin
  position relative to the corridor's spawn point is an attribute of the corridor geometry
  itself; it must be identical across every Unity-spawned corridor instance, and projects
  reusing the same paradigm must not be able to redefine it.

---

## MCP tool surface

| Tool                                  | Purpose                                                                       |
|---------------------------------------|-------------------------------------------------------------------------------|
| `discover_templates_tool`             | Lists all task templates in the configured templates directory                |
| `read_template_tool`                  | Reads an existing template by name                                            |
| `write_template_tool`                 | Writes a new template or overwrites an existing one (exclusive to this skill) |
| `describe_template_schema_tool`       | Returns the field schema for `TaskTemplate` (exclusive to this skill)         |
| `validate_template_tool`              | Validates a template against its schema and cross-reference constraints       |
| `list_supported_trial_types_tool`     | Enumerates trial classes supported by experiment configurations (exclusive)   |
| `list_supported_trigger_types_tool`   | Enumerates the `TriggerType` enum values (exclusive)                          |

---

## Authoring workflow

### Step 1: Verify prerequisites

- The `sollertia-shared-assets` MCP server is connected (else hand off to `/assets-mcp-environment-setup`).
- The task templates directory is set (else hand off to `/working-directory`).

### Step 2: Discover existing templates

```text
discover_templates_tool()
```

If a similar template already exists, prefer reading it (`read_template_tool`) and modifying a copy.
Avoid creating near-duplicates.

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
   prefab name, cm-per-unity-unit conversion).
2. **Cue catalog** — define every `Cue` (name, uint8 code, length, optional texture filename).
3. **Segments** — define each `Segment` (cue sequence + optional transition probabilities that sum
   to 1.0).
4. **Trial structures** — populate the `trial_structures` dict, mapping each trial name to a
   `TrialStructure` (segment reference, stimulus trigger zone start/end, stimulus location, whether
   the collision boundary is visible, trigger type from `list_supported_trigger_types_tool`).
5. **Cue offset** — set `cue_offset_cm` (the animal's starting position offset relative to the cue
   sequence origin).

Trial weights, reward sizes, gas-puff durations, occupancy thresholds, experiment states, and the
choice of trial subclass (`WaterRewardTrial` vs `GasPuffTrial`) are **not** part of the template —
they are added per-experiment by `/experiment-configuration`.

### Step 5: Write, validate, and re-read

```text
write_template_tool(
    template_name="<template-name>",
    template_payload={ ... full nested dict ... },
    overwrite=False,
)
validate_template_tool(template_name="<template-name>")
read_template_tool(template_name="<template-name>")
```

The kwargs are `template_name` and `template_payload` (the latter accepts a JSON-friendly dict).
Pass `overwrite=True` only when intentionally replacing an existing template.

`validate_template_tool` loads the template through `TaskTemplate.from_yaml`, which triggers
`__post_init__` validation:

- cue codes are unique
- cue codes are in `[0, 255]`
- cue names are unique
- segment `cue_sequence` entries are non-empty and reference valid cue names
- segment `transition_probabilities`, when provided, sum to 1.0
- each `TrialStructure.segment_name` references a valid segment
- each `TrialStructure.trigger_type` is a valid `TriggerType` value
- per-trial zone positions satisfy `start ≤ end`, `stimulus_location_cm ≥ start`, and all three are
  within the segment length

On success the tool returns a `summary` (cue/segment/trial counts plus `cue_offset_cm`); on failure
it returns an `issues` list. Fix any reported issues and re-write before handing off.

### Step 6: Hand off for Unity prefab generation and verification

When the template targets a Unity scene, hand off to the **unity plugin's `/task-prefabs`** to:
- Generate the concrete Unity task prefab (`generate_task_prefab_tool`).
- Validate segment prefab zone geometry against the template
  (`validate_prefab_against_template_tool`).

Programmatic validation is required — there is no manual fallback. If the Unity Editor, McpBridge,
or `slsa mcp` is unreachable, restore connectivity via `/unity-mcp-environment-setup` or
`/assets-mcp-environment-setup` before proceeding.

### Step 7: Hand off for experiment configuration

Once the template is written and validated, projects that want to use it must author per-project
`MesoscopeExperimentConfiguration` files via `/experiment-configuration`. This skill is not responsible
for instantiating templates into experiment configurations.

---

## Common patterns

### Add a new trial entry to an existing paradigm

1. Read the template with `read_template_tool`.
2. Confirm the trigger type via `list_supported_trigger_types_tool`.
3. Add a new `TrialStructure` entry to the `trial_structures` dict, keyed by trial name. Reference an
   existing segment in `segment_name`, set the stimulus zone bounds, and pick a `trigger_type`.
4. Write the template back with `write_template_tool` (use `overwrite=True`).
5. Re-run `validate_template_tool` to confirm the new entry passes cross-reference checks.
6. Hand off to `/experiment-configuration` if a per-project experiment needs to bind this trial to
   a `WaterRewardTrial` or `GasPuffTrial` subclass and assign weights.

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
- [ ] Payload was passed as template_payload and name as template_name (correct kwarg names)
- [ ] write_template_tool succeeded without schema errors
- [ ] validate_template_tool returned valid=True with no issues
- [ ] read_template_tool returned the expected content after the write
- [ ] If the template targets a Unity scene, unity plugin's /task-prefabs was invoked for prefab
      generation and validation
```

---

## Related skills

| Skill                                  | Relationship                                                    |
|----------------------------------------|-----------------------------------------------------------------|
| `/working-directory`                   | Required prerequisite — owns the templates directory path       |
| `/assets-mcp-environment-setup`        | Run first if the MCP server is not connected                    |
| `/experiment-configuration`            | Consumer — instantiates templates into per-project experiments  |
| unity plugin `/task-prefabs`           | Downstream — generates and validates the Unity prefab           |
| unity plugin `/scenes`                 | Downstream — places the generated prefab into a Unity scene     |
