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
- The trial primitive vocabulary (`Cue`, `Segment`, `BaseTrial`, `GasPuffTrial`, `WaterRewardTrial`,
  `TriggerType`, `VREnvironment`, `TrialStructure`)
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

## Trial primitive vocabulary

| Primitive          | Purpose                                                                  |
|--------------------|--------------------------------------------------------------------------|
| `Cue`              | A visual / auditory / olfactory cue that can be triggered during a trial |
| `Segment`          | A spatial region of the VR environment with associated cues and rewards  |
| `BaseTrial`        | A generic trial type defined by an ordered sequence of segments          |
| `GasPuffTrial`     | A trial that delivers an aversive gas puff stimulus                      |
| `WaterRewardTrial` | A trial that delivers a water reward                                     |
| `TriggerType`      | Enum describing what event triggers a cue or stimulus                    |
| `VREnvironment`    | The VR scene the template runs in                                        |
| `TrialStructure`   | The ordered sequence of trials and their weights                         |

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

- The `sollertia-shared-assets` MCP server is connected (else hand off to `/mcp-environment-setup`).
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

1. **VR environment** — pick or define the `VREnvironment` (Unity scene name + dimensions).
2. **Cue catalog** — define every `Cue` the trials will reference.
3. **Segments** — define each `Segment` (spatial region + which cues fire there).
4. **Base trials** — assemble `BaseTrial` instances from segments. Add `GasPuffTrial` /
   `WaterRewardTrial` variants for trials that deliver stimuli or rewards.
5. **Trial structure** — order the trials and assign weights in `TrialStructure`.

### Step 5: Write, validate, and re-read

```text
write_template_tool(name="<template-name>", template={ ... full nested dict ... })
validate_template_tool(name="<template-name>")
read_template_tool(name="<template-name>")
```

`validate_template_tool` checks schema compliance, cue-code uniqueness, segment-to-cue references,
trial-to-segment references, trigger-type validity, and transition probability sums. If it returns
validation errors, fix the template and re-write before handing off.

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

### Add a new trial type to an existing paradigm

1. Read the template with `read_template_tool`.
2. Confirm the trial class name via `list_supported_trial_types_tool` and the trigger type via
   `list_supported_trigger_types_tool`.
3. Add a `BaseTrial`, `GasPuffTrial`, or `WaterRewardTrial` entry to the trial list.
4. Update the `TrialStructure` weights to include the new trial.
5. Write the template back with `write_template_tool`.
6. Re-run `validate_template_tool` to confirm the new trial passes cross-reference checks.

### Migrate a template to a new VR scene

1. Read the template with `read_template_tool`.
2. Mutate the `VREnvironment` field to point at the new scene.
3. `validate_template_tool` to catch structural regressions.
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
- [ ] write_template_tool succeeded without schema errors
- [ ] validate_template_tool returned zero errors
- [ ] read_template_tool returned the expected content after the write
- [ ] If the template targets a Unity scene, unity plugin's /task-prefabs was invoked for prefab
      generation and validation
```

---

## Related skills

| Skill                                  | Relationship                                                    |
|----------------------------------------|-----------------------------------------------------------------|
| `/working-directory`                   | Required prerequisite — owns the templates directory path       |
| `/mcp-environment-setup`               | Run first if the MCP server is not connected                    |
| `/experiment-configuration`            | Consumer — instantiates templates into per-project experiments  |
| unity plugin `/task-prefabs`           | Downstream — generates and validates the Unity prefab           |
| unity plugin `/scenes`                 | Downstream — places the generated prefab into a Unity scene     |
