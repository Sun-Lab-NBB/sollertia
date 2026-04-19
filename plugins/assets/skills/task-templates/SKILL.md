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
project-agnostic — the same template can back many `MesoscopeExperimentConfiguration` instances across
many projects.

A template defines **what is possible**. An experiment configuration picks a template and parameterizes
it (state durations, trial weights, reward volumes, project-specific overrides). The two are authored by
two different skills with two different ownership scopes.

---

## Template vocabulary

A `TaskTemplate` is composed of these classes (all defined in `sollertia_shared_assets.configuration`):

| Primitive        | Purpose                                                                                  |
|------------------|------------------------------------------------------------------------------------------|
| `Cue`            | A visual cue (name, uint8 code, length, optional texture) referenced by segments         |
| `Segment`        | A spatial region: cue sequence + optional transition probabilities to other segments     |
| `TrialStructure` | Per-trial spatial config: segment, stimulus trigger zone, stimulus location, trigger type |
| `VREnvironment`  | VR corridor configuration: spacing, segments per corridor, padding prefab, units         |
| `TriggerType`    | Enum of stimulus trigger zone activators (`lick`, `occupancy`)                           |

`TaskTemplate.trial_structures` is a `dict[str, TrialStructure]` keyed by trial name. The template
itself does **not** carry trial weights, experiment-specific reward parameters, or trial-class
choices — those live on `MesoscopeExperimentConfiguration` and are owned by `/experiment-configuration`.
The trial-class subclasses `BaseTrial`, `WaterRewardTrial`, and `GasPuffTrial` are experiment-scope
classes; they appear in this skill only when you call `list_supported_trial_types_tool` to enumerate
what an experiment configuration may instantiate from a template's `TrialStructure` entries.

For canonical field definitions and valid values, call `describe_template_schema_tool` — do not rely on
handwritten documentation that may drift from the slsa source of truth. For the canonical trial-class
and trigger-type enum values, use `list_supported_trial_types_tool` and
`list_supported_trigger_types_tool` respectively.

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
accepted by `trigger_type` and the exact class names used for `BaseTrial`, `GasPuffTrial`, and
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
`__post_init__` validation (schema compliance, cue-code uniqueness in [0, 255], non-empty segment
cue sequences, transition-probability sums equal to 1.0). It returns either a `summary`
(cue/segment/trial counts plus `cue_offset_cm`) or an `issues` list. Fix any reported issues and
re-write before handing off.

### Step 6: Hand off for Unity prefab generation and verification

When the template targets a Unity scene, hand off to the **unity plugin's `/task-prefabs`** to:
- Generate the concrete Unity task prefab (`generate_task_prefab_tool`).
- Validate segment prefab zone geometry against the template
  (`validate_prefab_against_template_tool`).

This replaces the older manual verification workflow.

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
| `/assets-mcp-environment-setup`               | Run first if the MCP server is not connected                    |
| `/experiment-configuration`            | Consumer — instantiates templates into per-project experiments  |
| unity plugin `/task-prefabs`           | Downstream — generates and validates the Unity prefab           |
| unity plugin `/scenes`                 | Downstream — places the generated prefab into a Unity scene     |
