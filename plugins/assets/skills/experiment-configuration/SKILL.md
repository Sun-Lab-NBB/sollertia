---
name: experiment-configuration
description: >-
  Authors and modifies per-project, system-specific experiment configuration YAML files for
  sollertia-shared-assets via the slsa MCP server. Currently the only concrete subclass is
  MesoscopeExperimentConfiguration, but the factory registry is designed for additional
  acquisition systems. Owns the experiment configuration write tool, the
  create_experiment_config_tool convenience helper, and schema introspection. Use when designing a new
  experiment configuration for a project, customizing trial parameters, or instantiating an existing
  task template into a new experiment.
user-invocable: true
---

# Sollertia experiment configuration

Authors and modifies per-project, system-specific experiment configuration YAML files for
`sollertia-shared-assets` using the `slsa mcp` MCP server. The only concrete subclass that exists
today is `MesoscopeExperimentConfiguration`, but the system factory registry
(`_experiment_config_factory_registry`) and the `AcquisitionSystems` enum are deliberately
extensible — additional systems may be added in the future, at which point this skill will own
their experiment configurations as well. This skill is the **exclusive** owner of:

- `write_experiment_configuration_tool`
- `create_experiment_config_tool`
- `describe_experiment_configuration_schema_tool`
- `validate_experiment_configuration_tool`

No other skill in the marketplace may call these tools.

---

## Scope

**Covers:**
- Authoring per-project experiment configurations (currently only `MesoscopeExperimentConfiguration`)
- Experiment state machines (`ExperimentState`, `populate_default_experiment_states`)
- Schema introspection for experiment configurations
- Reading the frozen experiment configuration captured at session start
  (`read_session_experiment_configuration_tool`)
- Instantiating an existing task template into a new experiment configuration

**Does not cover:**
- Authoring task templates themselves (see `/task-templates`)
- Authoring system-level configuration (see experiment plugin's `/system-configuration`)
- Authoring server configuration (see forging plugin's `/server-configuration`)
- Creating projects (see `/project-hierarchy`)
- Verifying that template values match the Unity prefab state (see unity plugin's `/task-prefabs`)
- Initial working directory setup (see `/working-directory`)

---

## Templates vs experiment configurations

Sollertia separates **task structure** from **per-project configuration**. Templates are
acquisition-system-agnostic; the same template can back many experiment configurations across many
projects, and across additional acquisition systems if and when their own factories are registered
in `_experiment_config_factory_registry`.

| Concept                              | What it is                                              | Owning skill      |
|--------------------------------------|---------------------------------------------------------|-------------------|
| `TaskTemplate`                       | Reusable VR environment, trial structure, cue catalog   | `/task-templates` |
| System-specific experiment config    | Per-project template instantiation + state machine      | this skill        |
| `MesoscopeExperimentConfiguration`   | Currently the only concrete experiment-config subclass  | this skill        |

A template defines **what is possible**. An experiment configuration picks **which template to use**
and parameterizes it (state durations, trial weights, reward volumes, project-specific overrides). The
two are owned by two different skills.

---

## How experiment configurations are executed at runtime

An experiment configuration is the contract between the agent-authored definition of what should
happen during a session and the acquisition runtime that actually conducts that session. The
description below is intentionally conceptual; for the precise call sequence and tool surface
of any specific consumer, defer to the runtime package's own documentation.

### The state machine

`experiment_states` is a **`dict[str, ExperimentState]`** iterated in **insertion order**. That
ordering is the sequence in which states fire; the string keys (`state_1`, `state_2`, …) give
each state a stable, human-readable identifier for logs and analysis without forcing positional
indexing. Each state holds for `state_duration_s` seconds, then control falls through to the next
state. The session ends when the last state's timer expires.

The dict-of-named-states shape (rather than a list) lets you add, rename, or reorder states by
editing keys and re-emitting the YAML, without renumbering downstream references. The keys
appear verbatim in the log stream that the analysis pipeline aligns trials against.

### Experiment states vs system states

Each `ExperimentState` carries two distinct codes:

- **`experiment_state_code`** — the logical phase label (e.g. "warm-up", "high-difficulty",
  "cool-down"). Emitted into the data log at the moment the state begins so downstream analysis
  can slice trials by phase.
- **`system_state_code`** — the hardware-mode snapshot that the acquisition runtime should
  install for the duration of the state (Mesoscope-VR currently accepts `1` = REST and `2` =
  RUN; other systems will define their own).

The two codes are deliberately decoupled. Two consecutive experiment states can reuse the same
system state (so the hardware mode does not change between phases) while differing in duration
or guidance settings. For example, a "ramp" RUN phase and a "performance" RUN phase that share
the running treadmill mode but apply different guidance counters. This decoupling is why both
fields exist on the schema; collapsing them would force a hardware reconfiguration on every
phase boundary.

`ExperimentState.supports_trials` exists on the schema but is currently unused by the runtime;
treat it as reserved for future use. To plan a trial-free phase today, choose a `system_state_code`
that does not engage trial-driving hardware (e.g. REST on Mesoscope-VR).

### Trials emerge from the template, not from the experiment configuration

The experiment configuration **does not enumerate or schedule trials**. The trial sequence is
determined by the template's segment topology and `transition_probabilities` (see
`/task-templates`): the acquisition runtime materializes a cue sequence at session init from the
template, then identifies trial boundaries within that sequence by motif matching against each
`TrialStructure`.

This is why there is **no `trial_weights` field** on the experiment configuration. Relative
frequencies are encoded in the template's transition probabilities, and the per-session trial
ordering is fully determined once the cue sequence is materialized. The experiment configuration
contributes only the **per-trial-type parameters** — reward volume, tone duration, puff
duration, occupancy threshold — never the schedule.

### Guidance is per-state, not per-trial-class

Each `ExperimentState` carries reinforcing and aversive guidance counters
(`*_initial_guided_trials`, `*_recovery_failed_threshold`, `*_recovery_guided_trials`). At state
entry the runtime configures the counters from these fields and zeros any prior failure streak.
During the state, completed trials count down the guided-trials budget; sustained failure
streaks re-engage guidance for a recovery window.

The guidance counters live on `ExperimentState` rather than on the trial classes because
guidance behavior is fundamentally a **phase-level** decision. A typical training paradigm
sweeps through a "ramp" phase (heavy initial guidance, low recovery threshold), a "performance"
phase (no initial guidance, modest recovery support), and a "no help" phase (zero on both
counters). All reusing the same trial classes and the same template. One experiment
configuration captures that whole arc by chaining states with different guidance settings.

### Why the schema is shaped this way

- **Template-derived fields (`cues`, `segments`, `vr_environment`, `cue_offset_cm`)** are
  copied out of the `TaskTemplate` when the experiment configuration is created so the YAML is
  self-contained for the runtime. It never re-resolves against the template at session time,
  and the per-session frozen snapshot freezes a complete record of what was run for downstream
  analysis.
- **`trial_structures: dict[str, WaterRewardTrial | GasPuffTrial]`** carries the choice of
  trial *class* per trial name. The template only provides the spatial `TrialStructure`; the
  experiment configuration promotes each entry to a concrete subclass and attaches the per-trial
  parameters. The natural pairing is `trigger_type: "lick"` → `WaterRewardTrial` and
  `trigger_type: "occupancy"` → `GasPuffTrial`, because the template's `trigger_type` is what
  Unity used to bake the matching zone prefab. Cross-pairing is schema-legal but produces a
  prefab-vs-runtime mismatch — stick to the matching pairing.
- **`unity_scene_name`** is on the experiment configuration (not the template) because the
  acquisition runtime verifies it against the actual scene loaded in Unity at session start;
  putting the expected scene name on the per-project experiment configuration lets two projects
  point the same template at differently-named scene files.

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
| `list_supported_acquisition_systems_tool`       | Enumerates the `AcquisitionSystems` enum values                 |

---

## Authoring an experiment configuration

### Step 1: Verify prerequisites

- MCP server connected (else `/assets-mcp-environment-setup`).
- The slsa task templates directory is set (via `/working-directory`) **only if you plan to use
  `create_experiment_config_tool`** — that tool resolves the template by name from the
  configured templates directory. The other tools in this skill (write/read/validate/discover
  experiment configurations) take `root_directory` and `template`/`experiment` as explicit
  arguments and never consult the slsa working directory.
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

`root_directory` is required for every read/write/validate/create call — the active system
configuration moved out of `sollertia-shared-assets` and into the acquisition runtime package
(`sollertia-experiment`), so slsa can no longer auto-resolve the data root. If a similar experiment already
exists, prefer reading it (`read_experiment_configuration_tool`) and modifying a copy.

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

Mutate the payload to override the fields the user wants to customize. The schema below applies to
`MesoscopeExperimentConfiguration` — currently the only concrete experiment-config subclass.
Future system-specific subclasses will likely follow a similar shape (template-derived VR fields,
state machine, trial structures), but consult `describe_experiment_configuration_schema_tool` with
the matching `acquisition_system` value rather than assuming the mesoscope schema applies verbatim.

- `cues: list[Cue]`
- `segments: list[Segment]`
- `trial_structures: dict[str, WaterRewardTrial | GasPuffTrial]` — a per-trial dict; each entry is
  either a `WaterRewardTrial` (with `reward_size_ul`, `reward_tone_duration_ms`) or a `GasPuffTrial`
  (with `puff_duration_ms`, `occupancy_duration_ms`), each carrying inherited spatial fields.
- `experiment_states: dict[str, ExperimentState]` — a dict, **not a list**. Access by string key
  (e.g. `experiment_states["state_1"].state_duration_s`), not by integer index. `populate_default_experiment_states`
  generates 1-indexed names (`state_1`, `state_2`, …); the first autopopulated state is `state_1`,
  not `state_0`. `ExperimentState` fields include `experiment_state_code`, `system_state_code`,
  `state_duration_s`, `supports_trials`, and the reinforcing/aversive guidance counters.
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

`validate_experiment_configuration_tool` loads the YAML through the matching experiment-config
`from_yaml` (today: `MesoscopeExperimentConfiguration.from_yaml`), which triggers `__post_init__`
validation:

- cue codes are unique
- cue names are unique
- segment cue sequences reference valid cue names
- each trial structure's `segment_name` references a valid segment

After those checks pass, `__post_init__` populates each trial's `cue_sequence` and `trial_length_cm`
from the referenced segment, then calls `BaseTrial.validate_zones()`, which enforces:

- `stimulus_trigger_zone_end_cm ≥ stimulus_trigger_zone_start_cm`
- `0 ≤ stimulus_trigger_zone_start_cm ≤ trial_length_cm`
- `0 ≤ stimulus_trigger_zone_end_cm ≤ trial_length_cm`
- `0 ≤ stimulus_location_cm ≤ trial_length_cm`
- `stimulus_location_cm ≥ stimulus_trigger_zone_start_cm`

On success the tool returns a `summary` (cue/segment/trial/state counts plus `unity_scene_name`); on
failure it returns an `issues` list. Fix any reported issues and re-write.

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

| Goal                                  | Pattern                                                                                                                                                                                                                                                                                                   |
|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reuse a template across projects      | Call `create_experiment_config_tool` per project, then override per-project fields                                                                                                                                                                                                                        |
| Change reward volume for a trial type | Edit `trial_structures["<trial>"].reward_size_ul` for `WaterRewardTrial` entries                                                                                                                                                                                                                          |
| Change gas-puff duration              | Edit `trial_structures["<trial>"].puff_duration_ms` for `GasPuffTrial` entries                                                                                                                                                                                                                            |
| Adjust a state's duration             | Edit `experiment_states["<state-key>"].state_duration_s` (state machine is a dict)                                                                                                                                                                                                                        |
| Add a new state to the state machine  | Add a new key to the `experiment_states` dict, then re-validate                                                                                                                                                                                                                                           |
| Add a new spatial trial entry         | First hand off to `/task-templates` to add the `TrialStructure` to the template, then either re-run `create_experiment_config_tool` with `overwrite=True` or amend this skill's experiment config via `write_experiment_configuration_tool` to add the matching `WaterRewardTrial` / `GasPuffTrial` entry |

### Migrating an experiment to a new template

1. Read the old configuration with `read_experiment_configuration_tool` (pass `root_directory`).
2. If the new template does not exist, hand off to `/task-templates` to author it.
3. Call `create_experiment_config_tool` with the new template name and `root_directory`.
4. Port the customizations (state durations, per-trial reward sizes, puff and occupancy durations,
   guidance counters) over manually.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] If using create_experiment_config_tool, /working-directory has set the templates directory
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

| Skill                                     | Relationship                                                                                                                                         |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/working-directory`                      | Required only when calling `create_experiment_config_tool` (provides the templates directory). All other tools here take `root_directory` explicitly |
| `/assets-mcp-environment-setup`           | Run first if the MCP server is not connected                                                                                                         |
| `/task-templates`                         | Required upstream — owns template authoring                                                                                                          |
| `/project-hierarchy`                      | Required upstream — owns project creation                                                                                                            |
| experiment plugin `/system-configuration` | Owns MesoscopeSystemConfiguration (moved out of this plugin)                                                                                         |
| unity plugin `/task-prefabs`              | Validates template values against the Unity prefab state                                                                                             |
| experiment plugin `/experiment-pipeline`  | Phase 4 of the experiment lifecycle is owned by this skill                                                                                           |
