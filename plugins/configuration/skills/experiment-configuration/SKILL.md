---
name: configure-experiment-configuration
description: >-
  Authors and modifies experiment configuration YAML files (MesoscopeExperimentConfiguration) and task
  templates for sollertia-shared-assets via the sl-configure MCP server. Covers task template authoring,
  trial structures, experiment state machines, and the relationship between templates and per-project
  experiment configurations. Use when designing a new experiment, customizing trial parameters, or
  reusing an existing template for a new project.
user-invocable: true
---

# Sollertia experiment configuration

Authors and modifies experiment configuration YAML files and reusable task templates for
`sollertia-shared-assets` using the `sl-configure mcp` MCP server.

---

## Scope

**Covers:**
- Authoring task templates (`TaskTemplate`) in the templates directory
- Authoring per-project experiment configurations (`MesoscopeExperimentConfiguration`)
- Trial structures (cues, segments, base trials, gas puff trials, water reward trials)
- Experiment state machines (`ExperimentState`, `populate_default_experiment_states`)
- Schema introspection for templates and experiment configurations

**Does not cover:**
- Authoring system-level configuration (see `/system-configuration`)
- Authoring session descriptors (see `/session-data`)
- Verifying that template values match the Unity prefab state (see the experiment plugin's
  `/configuration-verification`)
- Initial working directory and templates directory setup (see `/working-directory`)

---

## Templates vs experiment configurations

Sollertia separates **task structure** from **per-project configuration** so that the same experimental
paradigm can be reused across projects with project-specific parameter overrides.

| Concept                  | What it is                                                          | Where it lives                 |
|--------------------------|---------------------------------------------------------------------|--------------------------------|
| `TaskTemplate`           | Reusable VR environment + trial structure + cue catalog             | Task templates directory       |
| `MesoscopeExperimentConfiguration` | Project-specific instantiation of a template plus state machine | Per-project experiment YAML    |

A template defines **what is possible** (which trials exist, what cues are available, the VR layout). An
experiment configuration picks **which template to use** and parameterizes it (state durations, trial
weights, reward volumes, project-specific overrides).

---

## Trial structure primitives

Templates compose these primitives (see `sollertia_shared_assets.configuration` for the dataclasses):

| Primitive          | Purpose                                                                              |
|--------------------|--------------------------------------------------------------------------------------|
| `Cue`              | A visual / auditory / olfactory cue that can be triggered during a trial             |
| `Segment`          | A spatial region of the VR environment with associated cues and reward zones        |
| `BaseTrial`        | A generic trial type defined by an ordered sequence of segments                     |
| `GasPuffTrial`     | A trial that delivers an aversive gas puff stimulus                                 |
| `WaterRewardTrial` | A trial that delivers a water reward                                                |
| `TriggerType`      | Enum describing what event triggers a cue or stimulus                               |
| `VREnvironment`    | The VR scene the template runs in                                                   |
| `TrialStructure`   | The ordered sequence of trials and their weights                                    |

---

## MCP tool surface

| Tool                                                | Purpose                                                                  |
|-----------------------------------------------------|--------------------------------------------------------------------------|
| `discover_templates_tool`                           | Lists all task templates in the configured templates directory           |
| `describe_template_schema_tool`                     | Returns the field schema for `TaskTemplate`                              |
| `read_template_tool`                                | Reads an existing template by name                                       |
| `write_template_tool`                               | Writes a new template or overwrites an existing one                      |
| `discover_experiments_tool`                         | Lists experiment configurations under a project                          |
| `describe_experiment_configuration_schema_tool`     | Returns the field schema for the experiment configuration dataclass      |
| `read_experiment_configuration_tool`                | Reads a project's experiment configuration                               |
| `write_experiment_configuration_tool`               | Writes a new experiment configuration                                    |
| `create_experiment_config_tool`                     | Convenience: creates an experiment config from a template + parameters   |
| `list_supported_session_types_tool`                 | Returns the supported session type strings                               |

---

## Authoring a new template

Use this workflow when no existing template captures the trial structure the user needs.

### Step 1: Verify prerequisites

- MCP server connected (else `/configuration-mcp-environment-setup`)
- Working directory + templates directory configured (else `/working-directory`)

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
4. **Base trials** — assemble `BaseTrial` instances from segments. Add `GasPuffTrial` / `WaterRewardTrial`
   variants for trials that deliver stimuli or rewards.
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

---

## Authoring an experiment configuration

Use this workflow to create a per-project experiment configuration that instantiates an existing
template.

### Step 1: Verify the project exists

```text
discover_projects_tool()
```

If the project does not exist, use `create_project_tool(project="<name>")` first.

### Step 2: Pick a template

```text
discover_templates_tool()
```

Confirm with the user which template the experiment is based on.

### Step 3: Inspect the experiment configuration schema

```text
describe_experiment_configuration_schema_tool(acquisition_system="mesoscope")
```

### Step 4: Use create_experiment_config_tool for the standard path

For most cases, the convenience tool handles template instantiation, default state machine population,
and per-project parameter injection in one call:

```text
create_experiment_config_tool(
    project="<project>",
    experiment="<experiment-name>",
    template="<template-name>",
    ...
)
```

This calls `populate_default_experiment_states` internally to give every state a sensible default
duration and transition policy.

### Step 5: Customize state machine and parameters

Read the just-created configuration:

```text
read_experiment_configuration_tool(project="<project>", experiment="<experiment-name>")
```

Mutate the dictionary to override the fields the user wants to customize:

- **State durations** — `experiment_states[<i>].state_duration_s`
- **Trial weights** — `trial_structure.trial_weights`
- **Reward volumes** — `experiment_states[<i>].water_reward_volume_uL`
- **Cue parameters** — see template-specific fields

Then write it back:

```text
write_experiment_configuration_tool(
    project="<project>",
    experiment="<experiment-name>",
    configuration={ ... }
)
```

### Step 6: Verify

```text
read_experiment_configuration_tool(project="<project>", experiment="<experiment-name>")
```

Confirm the returned configuration matches your edits.

---

## Common patterns

| Goal                                       | Pattern                                                            |
|--------------------------------------------|--------------------------------------------------------------------|
| Reuse a template across projects           | `create_experiment_config_tool` for each project, override only the project-specific fields |
| Add a new trial type to an existing paradigm | Read template → add `BaseTrial`/`GasPuffTrial`/`WaterRewardTrial` → update `TrialStructure` weights → write template |
| Change reward volume per session block     | Edit `experiment_states[*].water_reward_volume_uL` (no template change needed) |
| Add a new state to the state machine       | Edit `experiment_states` list, then re-verify ordering and transitions |
| Migrate an experiment to a new template    | Read old config → use `create_experiment_config_tool` with the new template → port over the customizations manually |

---

## Verification checklist

```text
- [ ] /working-directory has been run and the templates directory is set
- [ ] discover_templates_tool was called before creating new templates (avoid duplicates)
- [ ] describe_*_schema_tool was used as the source of truth for field names
- [ ] write_template_tool succeeded without schema errors
- [ ] write_experiment_configuration_tool succeeded without schema errors
- [ ] read_*_tool returned the expected content after every write
- [ ] State machine transitions form a valid graph (no orphaned states)
- [ ] Reward volumes and state durations are within plausible biological ranges
- [ ] /configuration-verification was invoked if the template targets a modified Unity scene
```

---

## Related skills

| Skill                                          | Relationship                                                       |
|------------------------------------------------|--------------------------------------------------------------------|
| `/working-directory`                           | Required prerequisite — must be run first                          |
| `/configuration-mcp-environment-setup`         | Run first if the MCP server is not connected                       |
| `/system-configuration`                        | Sibling — system config is consumed by the same runtime            |
| experiment plugin `/configuration-verification`| Validates template values against the actual Unity prefab state    |
| experiment plugin `/experiment-design`         | Higher-level interactive workflow that uses this skill's tools     |
| experiment plugin `/pipeline`                  | Phase 4 of the experiment lifecycle is owned by this skill         |
