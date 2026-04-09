---
name: configure-experiment-configuration
description: >-
  Authors and modifies per-project MesoscopeExperimentConfiguration YAML files for sollertia-shared-assets
  via the sl-configure MCP server. Owns the experiment configuration write tool, the
  create_experiment_config_tool convenience helper, and schema introspection. Use when designing a new
  experiment configuration for a project, customizing trial parameters, or instantiating an existing
  task template into a new experiment.
user-invocable: true
---

# Sollertia experiment configuration

Authors and modifies per-project `MesoscopeExperimentConfiguration` YAML files for
`sollertia-shared-assets` using the `sl-configure mcp` MCP server. This skill is the **exclusive** owner
of `write_experiment_configuration_tool`, `create_experiment_config_tool`, and
`describe_experiment_configuration_schema_tool` — no other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Authoring per-project experiment configurations (`MesoscopeExperimentConfiguration`)
- Experiment state machines (`ExperimentState`, `populate_default_experiment_states`)
- Schema introspection for experiment configurations
- Reading the frozen experiment configuration captured at session start
  (`read_session_experiment_configuration_tool`)
- Instantiating an existing task template into a new experiment configuration

**Does not cover:**
- Authoring task templates themselves (see `/task-templates`)
- Authoring system-level configuration (see `/system-configuration`)
- Authoring server configuration (see `/server-configuration`)
- Creating projects (see `/project-hierarchy`)
- Verifying that template values match the Unity prefab state (see the experiment plugin's
  `/configuration-verification`)
- Initial working directory setup (see `/working-directory`)

---

## Templates vs experiment configurations

Sollertia separates **task structure** from **per-project configuration**. The same task template can
back many experiment configurations across many projects.

| Concept                            | What it is                                                 | Owning skill                |
|------------------------------------|------------------------------------------------------------|-----------------------------|
| `TaskTemplate`                     | Reusable VR environment + trial structure + cue catalog    | `/task-templates`           |
| `MesoscopeExperimentConfiguration` | Project-specific instantiation of a template + state machine | this skill                |

A template defines **what is possible**. An experiment configuration picks **which template to use**
and parameterizes it (state durations, trial weights, reward volumes, project-specific overrides). The
two are owned by two different skills.

---

## MCP tool surface

| Tool                                                | Purpose                                                                  |
|-----------------------------------------------------|--------------------------------------------------------------------------|
| `discover_experiments_tool`                         | Lists experiment configurations under a project                          |
| `describe_experiment_configuration_schema_tool`     | Returns the field schema for the experiment configuration dataclass      |
| `read_experiment_configuration_tool`                | Reads a project's experiment configuration                               |
| `write_experiment_configuration_tool`               | Writes a new experiment configuration (exclusive to this skill)          |
| `create_experiment_config_tool`                     | Convenience: creates an experiment config from a template + parameters (exclusive to this skill) |
| `read_session_experiment_configuration_tool`        | Reads the frozen experiment configuration captured at session start      |

---

## Authoring an experiment configuration

### Step 1: Verify prerequisites

- MCP server connected (else `/mcp-environment-setup`).
- Working directory set (else `/working-directory`).
- The target project exists. If it does not, hand off to `/project-hierarchy` to create it. This skill
  must not call `create_project_tool` directly.
- The target task template exists. If it does not, hand off to `/task-templates` to author it. This
  skill must not call `write_template_tool` directly.

### Step 2: Discover existing experiments under the project

```text
discover_experiments_tool(project="<project>")
```

If a similar experiment already exists, prefer reading it (`read_experiment_configuration_tool`) and
modifying a copy.

### Step 3: Inspect the experiment configuration schema

```text
describe_experiment_configuration_schema_tool(acquisition_system="mesoscope")
```

Use the schema as the source of truth for field names and nesting.

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

## Reading the frozen configuration from a session

After a session has run, the experiment configuration that was active at session start is captured as a
frozen YAML inside the session directory. To read it:

```text
read_session_experiment_configuration_tool(session_path="<absolute>")
```

This is a read-only operation. Do not attempt to write to the frozen file — modifying historical session
metadata is the responsibility of `/session-snapshots` (which deals with hardware snapshots, not the
experiment config). If a frozen experiment config needs to be amended for some reason, that is currently
not supported by the slsa MCP layer.

---

## Common patterns

| Goal                                       | Pattern                                                            |
|--------------------------------------------|--------------------------------------------------------------------|
| Reuse a template across projects           | `create_experiment_config_tool` for each project, override only the project-specific fields |
| Change reward volume per session block     | Edit `experiment_states[*].water_reward_volume_uL` (no template change needed) |
| Add a new state to the state machine       | Edit `experiment_states` list, then re-verify ordering and transitions |
| Migrate an experiment to a new template    | Read old config → hand off to `/task-templates` if the new template doesn't exist → call `create_experiment_config_tool` with the new template → port over customizations manually |
| Add a new trial type                       | Hand off to `/task-templates` to modify the template — don't author trial structure here |

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Target project exists (handed off to /project-hierarchy if missing)
- [ ] Target template exists (handed off to /task-templates if missing)
- [ ] describe_experiment_configuration_schema_tool was used as the source of truth for field names
- [ ] write_experiment_configuration_tool succeeded without schema errors
- [ ] read_experiment_configuration_tool returned the expected configuration after the write
- [ ] State machine transitions form a valid graph (no orphaned states)
- [ ] Reward volumes and state durations are within plausible biological ranges
- [ ] Did not call create_project_tool or write_template_tool from this skill
```

---

## Related skills

| Skill                                          | Relationship                                                       |
|------------------------------------------------|--------------------------------------------------------------------|
| `/working-directory`                           | Required prerequisite — must be run first                          |
| `/mcp-environment-setup`                       | Run first if the MCP server is not connected                       |
| `/task-templates`                              | Required upstream — owns template authoring                        |
| `/project-hierarchy`                           | Required upstream — owns project creation                          |
| `/system-configuration`                        | Sibling — system config is consumed by the same runtime            |
| experiment plugin `/configuration-verification`| Validates template values against the Unity prefab state           |
| experiment plugin `/pipeline`                  | Phase 4 of the experiment lifecycle is owned by this skill         |
