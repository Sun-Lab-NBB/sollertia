---
name: task-templates
description: >-
  Authors and modifies reusable TaskTemplate YAML files in the task templates directory via the
  sl-configure MCP server. Covers VR environment definition, cue catalogs, segment composition, trial
  primitives, trial structure, and the schema introspection tools. Use when designing a new behavioral
  task template, modifying an existing template, or preparing templates for use by per-project experiment
  configurations.
user-invocable: true
---

# Sollertia task templates

Authors and modifies reusable `TaskTemplate` YAML files for `sollertia-shared-assets` using the
`sl-configure mcp` MCP server. This skill is the **exclusive** owner of `write_template_tool` and the
template schema introspection tools — no other skill in the marketplace may call
`write_template_tool`.

---

## Scope

**Covers:**
- Authoring `TaskTemplate` YAML files in the configured task templates directory
- The trial primitive vocabulary (`Cue`, `Segment`, `BaseTrial`, `GasPuffTrial`, `WaterRewardTrial`,
  `TriggerType`, `VREnvironment`, `TrialStructure`)
- Discovering existing templates and reading their contents
- Schema introspection via `describe_template_schema_tool`

**Does not cover:**
- Per-project `MesoscopeExperimentConfiguration` authoring (see `/experiment-configuration`)
- Setting the task templates directory path (see `/working-directory`)
- Verifying template values against the actual Unity prefab state (see the experiment plugin's
  `/configuration-verification`)

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
handwritten documentation that may drift from the slsa source of truth.

---

## MCP tool surface

| Tool                            | Purpose                                                                       |
|---------------------------------|-------------------------------------------------------------------------------|
| `discover_templates_tool`       | Lists all task templates in the configured templates directory                |
| `read_template_tool`            | Reads an existing template by name                                            |
| `write_template_tool`           | Writes a new template or overwrites an existing one (exclusive to this skill) |
| `describe_template_schema_tool` | Returns the field schema for `TaskTemplate`                                   |

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

### Step 3: Inspect the schema

```text
describe_template_schema_tool()
```

Use the schema as the source of truth for field names and nesting.

### Step 4: Author the template

Build the template dictionary in this order:

1. **VR environment** — pick or define the `VREnvironment` (Unity scene name + dimensions).
2. **Cue catalog** — define every `Cue` the trials will reference.
3. **Segments** — define each `Segment` (spatial region + which cues fire there).
4. **Base trials** — assemble `BaseTrial` instances from segments. Add `GasPuffTrial` /
   `WaterRewardTrial` variants for trials that deliver stimuli or rewards.
5. **Trial structure** — order the trials and assign weights in `TrialStructure`.

### Step 5: Write and verify

```text
write_template_tool(name="<template-name>", template={ ... full nested dict ... })
read_template_tool(name="<template-name>")
```

### Step 6: Hand off for Unity verification

If the template targets a Unity scene that has been built or modified, hand off to the experiment
plugin's `/configuration-verification` skill to confirm the template values match the Unity prefab
state.

### Step 7: Hand off for experiment configuration

Once the template is written, projects that want to use it must author per-project
`MesoscopeExperimentConfiguration` files via `/experiment-configuration`. This skill is not responsible
for instantiating templates into experiment configurations.

---

## Common patterns

### Add a new trial type to an existing paradigm

1. Read the template with `read_template_tool`.
2. Add a `BaseTrial`, `GasPuffTrial`, or `WaterRewardTrial` entry to the trial list.
3. Update the `TrialStructure` weights to include the new trial.
4. Write the template back with `write_template_tool`.

### Migrate a template to a new VR scene

1. Read the template with `read_template_tool`.
2. Mutate the `VREnvironment` field to point at the new scene.
3. Hand off to `/configuration-verification` to re-verify segment zones against the new prefab state.

### Audit which projects use a template

1. List templates with `discover_templates_tool`.
2. Hand off to `/experiment-configuration` to enumerate consuming experiments per project.

---

## Verification checklist

```text
- [ ] /working-directory has been run and the templates directory is set
- [ ] discover_templates_tool was called before creating a new template (avoid duplicates)
- [ ] describe_template_schema_tool was used as the source of truth for field names
- [ ] write_template_tool succeeded without schema errors
- [ ] read_template_tool returned the expected content after the write
- [ ] If the template targets a modified Unity scene, /configuration-verification was invoked
```

---

## Related skills

| Skill                                           | Relationship                                                    |
|-------------------------------------------------|-----------------------------------------------------------------|
| `/working-directory`                            | Required prerequisite — owns the templates directory path       |
| `/mcp-environment-setup`                        | Run first if the MCP server is not connected                    |
| `/experiment-configuration`                     | Consumer — instantiates templates into per-project experiments  |
| experiment plugin `/configuration-verification` | Validates template values against the actual Unity prefab state |
