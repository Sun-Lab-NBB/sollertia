---
name: experiment-configuration
description: >-
  Authors and modifies per-project MesoscopeExperimentConfiguration YAML files for sollertia-shared-assets
  via the slsa MCP server. Owns the experiment configuration write tool, the
  create_experiment_config_tool convenience helper, and schema introspection. Use when designing a new
  experiment configuration for a project, customizing trial parameters, or instantiating an existing
  task template into a new experiment.
user-invocable: true
---

# Sollertia experiment configuration

Authors and modifies per-project `MesoscopeExperimentConfiguration` YAML files for
`sollertia-shared-assets` using the `slsa mcp` MCP server. This skill is the **exclusive** owner of:

- `write_experiment_configuration_tool`
- `create_experiment_config_tool`
- `describe_experiment_configuration_schema_tool`
- `validate_experiment_configuration_tool`

No other skill in the marketplace may call these tools.

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
- Verifying that template values match the Unity prefab state (see unity plugin's `/task-prefabs`)
- Initial working directory setup (see `/working-directory`)

---

## Templates vs experiment configurations

Sollertia separates **task structure** from **per-project configuration**. The same task template can
back many experiment configurations across many projects.

| Concept                            | What it is                                            | Owning skill      |
|------------------------------------|-------------------------------------------------------|-------------------|
| `TaskTemplate`                     | Reusable VR environment, trial structure, cue catalog | `/task-templates` |
| `MesoscopeExperimentConfiguration` | Per-project template instantiation + state machine    | this skill        |

A template defines **what is possible**. An experiment configuration picks **which template to use**
and parameterizes it (state durations, trial weights, reward volumes, project-specific overrides). The
two are owned by two different skills.

---

## MCP tool surface

| Tool                                            | Purpose                                                         |
|-------------------------------------------------|-----------------------------------------------------------------|
| `discover_experiments_tool`                     | Lists experiment configurations under a project                 |
| `describe_experiment_configuration_schema_tool` | Returns the field schema for the experiment dataclass           |
| `read_experiment_configuration_tool`            | Reads a project's experiment configuration                      |
| `write_experiment_configuration_tool`           | Writes a new experiment configuration (exclusive to this skill) |
| `create_experiment_config_tool`                 | Creates a config from a template + parameters (exclusive)       |
| `validate_experiment_configuration_tool`        | Validates an experiment configuration YAML (exclusive)          |
| `read_session_experiment_configuration_tool`    | Reads the frozen experiment configuration from a session        |

---

## Authoring an experiment configuration

### Step 1: Verify prerequisites

- MCP server connected (else `/assets-mcp-environment-setup`).
- Working directory set (else `/working-directory`).
- The target project exists. If it does not, hand off to `/project-hierarchy` to create it. This skill
  must not call `create_project_tool` directly.
- The target task template exists. If it does not, hand off to `/task-templates` to author it. This
  skill must not call `write_template_tool` directly.

### Step 2: Discover existing experiments under the project

```text
discover_experiments_tool(
    root_directory="<absolute path to data root>",
    project="<project>",
)
```

`root_directory` is required for every read/write/validate/create call — the slsa working directory
no longer implicitly resolves the data root after the asset redistribution. If a similar experiment
already exists, prefer reading it (`read_experiment_configuration_tool`) and modifying a copy.

### Step 3: Inspect the experiment configuration schema

```text
describe_experiment_configuration_schema_tool(acquisition_system="mesoscope")
```

Use the schema as the source of truth for field names and nesting.

### Step 4: Use create_experiment_config_tool for the standard path

For most cases, the convenience tool handles template loading, default state-machine population, and
per-project file placement in one call:

```text
create_experiment_config_tool(
    project="<project>",
    experiment="<experiment-name>",
    template="<template-name>",
    root_directory="<absolute path to data root>",
    state_count=1,
    overwrite=False,
)
```

This loads the named template via `TaskTemplate.from_yaml`, builds the configuration with
`create_experiment_configuration` (system fixed to `MESOSCOPE_VR`, `unity_scene_name` defaulted to
the template name), then calls `populate_default_experiment_states` with `state_count` to seed the
`experiment_states` dict with default-valued runtime states.

### Step 5: Customize state machine and trial parameters

Read the just-created configuration:

```text
read_experiment_configuration_tool(
    project="<project>",
    experiment="<experiment-name>",
    root_directory="<absolute>",
)
```

Mutate the payload to override the fields the user wants to customize. The
`MesoscopeExperimentConfiguration` schema is:

- `cues: list[Cue]`
- `segments: list[Segment]`
- `trial_structures: dict[str, WaterRewardTrial | GasPuffTrial]` — a per-trial dict; each entry is
  either a `WaterRewardTrial` (with `reward_size_ul`, `reward_tone_duration_ms`) or a `GasPuffTrial`
  (with `puff_duration_ms`, `occupancy_duration_ms`), each carrying inherited spatial fields.
- `experiment_states: dict[str, ExperimentState]` — a dict, **not a list**. Access by string key
  (e.g. `experiment_states["state_0"].state_duration_s`), not by integer index. `ExperimentState`
  fields include `experiment_state_code`, `system_state_code`, `state_duration_s`, `supports_trials`,
  and the reinforcing/aversive guidance counters.
- `vr_environment: VREnvironment`
- `unity_scene_name: str`
- `cue_offset_cm: float`

There is no `trial_weights` field and no `water_reward_volume_uL` field — water-reward sizing lives
on `WaterRewardTrial.reward_size_ul`, and the relative frequency of trial types is determined by
segment transition probabilities in the template (not by per-trial weights here).

Then write it back:

```text
write_experiment_configuration_tool(
    project="<project>",
    experiment="<experiment-name>",
    configuration_payload={ ... },
    root_directory="<absolute>",
    overwrite=True,
)
```

The kwarg is `configuration_payload` (not `configuration`).

### Step 6: Validate and verify

```text
validate_experiment_configuration_tool(
    project="<project>",
    experiment="<experiment-name>",
    root_directory="<absolute>",
)
read_experiment_configuration_tool(
    project="<project>",
    experiment="<experiment-name>",
    root_directory="<absolute>",
)
```

`validate_experiment_configuration_tool` loads the YAML through
`MesoscopeExperimentConfiguration.from_yaml`, which triggers `__post_init__` validation: cue codes
unique, cue names unique, segment cue sequences reference valid cues, trial structures reference
valid segments. On success it returns a `summary` (cue/segment/trial/state counts plus
`unity_scene_name`); on failure it returns an `issues` list. Fix any reported issues and re-write.

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

| Goal                                   | Pattern                                                                                |
|----------------------------------------|----------------------------------------------------------------------------------------|
| Reuse a template across projects       | Call `create_experiment_config_tool` per project, then override per-project fields     |
| Change reward volume for a trial type  | Edit `trial_structures["<trial>"].reward_size_ul` for `WaterRewardTrial` entries       |
| Change gas-puff duration               | Edit `trial_structures["<trial>"].puff_duration_ms` for `GasPuffTrial` entries         |
| Adjust a state's duration              | Edit `experiment_states["<state-key>"].state_duration_s` (state machine is a dict)     |
| Add a new state to the state machine   | Add a new key to the `experiment_states` dict, then re-validate                        |
| Add a new spatial trial entry          | Hand off to `/task-templates` — trial-structure spatial config lives in the template   |

### Migrating an experiment to a new template

1. Read the old configuration with `read_experiment_configuration_tool` (pass `root_directory`).
2. If the new template does not exist, hand off to `/task-templates` to author it.
3. Call `create_experiment_config_tool` with the new template name and `root_directory`.
4. Port the customizations (state durations, per-trial reward sizes, puff and occupancy durations,
   guidance counters) over manually.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Target project exists (handed off to /project-hierarchy if missing)
- [ ] Target template exists (handed off to /task-templates if missing)
- [ ] describe_experiment_configuration_schema_tool was used as the source of truth for field names
- [ ] root_directory was passed to every read/write/validate/create call
- [ ] Payload was passed as configuration_payload (the correct kwarg name)
- [ ] write_experiment_configuration_tool succeeded without schema errors
- [ ] validate_experiment_configuration_tool returned valid=True with no issues
- [ ] read_experiment_configuration_tool returned the expected configuration after the write
- [ ] experiment_states was treated as a dict (string keys), not a list (integer indices)
- [ ] No reference to a non-existent trial_weights or water_reward_volume_uL field
- [ ] Reward sizes (reward_size_ul) and state durations are within plausible biological ranges
- [ ] Did not call create_project_tool or write_template_tool from this skill
```

---

## Related skills

| Skill                                          | Relationship                                                       |
|------------------------------------------------|--------------------------------------------------------------------|
| `/working-directory`                           | Required prerequisite — must be run first                          |
| `/assets-mcp-environment-setup`                       | Run first if the MCP server is not connected                       |
| `/task-templates`                              | Required upstream — owns template authoring                        |
| `/project-hierarchy`                           | Required upstream — owns project creation                          |
| experiment plugin `/system-configuration`      | Owns MesoscopeSystemConfiguration (moved out of this plugin)       |
| unity plugin `/task-prefabs`                   | Validates template values against the Unity prefab state           |
| experiment plugin `/experiment-pipeline`       | Phase 4 of the experiment lifecycle is owned by this skill         |
