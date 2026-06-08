---
name: experiment-configuration
description: >-
  Authors per-project experiment configuration YAMLs (currently only
  MesoscopeExperimentConfiguration) via the sollertia-shared-assets MCP server. Owns the
  create / write / validate experiment configuration tools and schema introspection. Use when
  creating a new experiment configuration, customizing trial parameters, or instantiating a
  task template for a project.
user-invocable: false
---

# Sollertia experiment configuration

Authors and modifies per-project, system-specific experiment configuration YAML files for
`sollertia-shared-assets` using the `slsa mcp` MCP server. The only concrete subclass that exists
today is `MesoscopeExperimentConfiguration`, but the system factory registry
(`_experiment_config_factory_registry`) and the `AcquisitionSystems` enum are deliberately
extensible — additional systems may be added in the future, at which point this skill will own
their experiment configurations as well. This skill is the **exclusive** owner of:

- `write_experiment_configuration_tool`
- `create_experiment_configuration_tool`
- `describe_experiment_configuration_schema_tool`
- `validate_experiment_configuration_tool`

No other skill in the marketplace may call these tools.

---

## Scope

**Covers:**
- Authoring per-project experiment configurations (currently only `MesoscopeExperimentConfiguration`)
- Experiment state machines (`ExperimentState`, `populate_default_experiment_states`)
- Schema introspection for experiment configurations
- Reading the frozen experiment configuration captured at session start (pass the per-session
  snapshot path to `read_experiment_configuration_tool`)
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
  RUN, as defined by `sollertia-experiment`'s system-state enum; other systems will define
  their own).

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
determined by the template's per-trial `transitions` (see `/task-templates`): the acquisition
runtime materializes a cue sequence at session init from the template, then identifies trial
boundaries within that sequence by motif matching against each `TrialStructure`.

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

- **`unity_scene_name`** identifies the paired `TaskTemplate` by filename stem. The experiment
  configuration carries **no VR data of its own** — it references the template by name, and the
  runtime joins the two by trial name at session init. The per-session frozen snapshot consists
  of both files (the experiment configuration YAML and the matching VR template YAML).
- **`trial_structures: dict[str, WaterRewardTrial | GasPuffTrial]`** carries the choice of
  trial *class* per trial name. The template only provides the spatial `TrialStructure`; the
  experiment configuration pairs each entry with a concrete runtime trial class and attaches the per-trial
  parameters. The natural pairing is `trigger_type: "lick"` → `WaterRewardTrial` and
  `trigger_type: "occupancy"` → `GasPuffTrial`, because the template's `trigger_type` selects
  which zone prefab Unity instantiates. Cross-pairing is schema-legal but produces a
  prefab-vs-runtime mismatch — stick to the matching pairing.
- **`unity_scene_name`** is on the experiment configuration (not the template) because the
  acquisition runtime verifies it against the actual scene loaded in Unity at session start;
  putting the expected scene name on the per-project experiment configuration lets two projects
  point the same template at differently-named scene files.

---

## MCP tool surface

| Tool                                            | Purpose                                                                                                            |
|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `discover_experiments_tool`                     | Lists experiment configurations under a project                                                                    |
| `describe_experiment_configuration_schema_tool` | Returns the field schema for the experiment dataclass                                                              |
| `read_experiment_configuration_tool`            | Reads an experiment configuration YAML from any canonical location (project source or per-session frozen snapshot) |
| `write_experiment_configuration_tool`           | Writes a new experiment configuration (exclusive to this skill)                                                    |
| `create_experiment_configuration_tool`          | Creates a config from a template + parameters (exclusive)                                                          |
| `validate_experiment_configuration_tool`        | Validates an experiment configuration YAML (exclusive)                                                             |
| `list_supported_acquisition_systems_tool`       | Enumerates the `AcquisitionSystems` enum values                                                                    |

---

## Path conventions

All read / write / validate / create tools in this skill take **explicit file paths**. The caller
resolves the path; the tools never consult `root_directory`, a project name, an experiment name, or
a template name. The canonical paths are:

| Asset                                         | Canonical path                                                 |
|-----------------------------------------------|----------------------------------------------------------------|
| Per-project experiment configuration          | `<root>/<project>/configuration/<experiment>.yaml`             |
| Task template                                 | `<templates-directory>/<template-name>.yaml`                   |
| Per-session frozen experiment-config snapshot | `<session>/raw_data/experiment_configuration.yaml`             |

Use `discover_experiments_tool(root_directory=..., project=...)` to enumerate existing configs and
their absolute paths; use `discover_templates_tool()` to enumerate template paths.

---

## Authoring an experiment configuration

### Step 1: Verify prerequisites

- MCP server connected (else `/assets-mcp-environment-setup`).
- The target project directory exists (i.e. `<root>/<project>/configuration/` is on disk).
  Project directories are created with the `create_project_tool` MCP tool or the
  `slsa configure project -p <name> -r <root>` CLI command (both create
  `<root>/<project>/configuration/`). `SessionData.create` raises `FileNotFoundError` when the
  project is missing, so the project must be created before any experiment configuration or session
  can be authored. This skill does not create project directories on its own; hand off to
  `/project-hierarchy`, which owns `create_project_tool`.
- The target task template exists at a known path. If it doesn't, hand off to `/task-templates`
  to author it — this skill must not call `write_template_tool` directly. The templates directory
  can be enumerated via `discover_templates_tool`, which also returns absolute paths.

### Step 2: Discover existing experiments under the project

```text
discover_experiments_tool(
    root_directory="<absolute path to data root>",
    project="<project>",
)
```

`discover_experiments_tool` returns every experiment's absolute `path`, which is what you'll pass
to the read/write/validate tools below. If a similar experiment already exists, prefer reading it
and modifying a copy.

### Step 3: Inspect the experiment configuration schema

```text
describe_experiment_configuration_schema_tool(acquisition_system="mesoscope")
```

Use the schema as the source of truth for field names and nesting.

### Step 4: Use create_experiment_configuration_tool for the standard path

For most cases, the convenience tool handles template loading and default state-machine
population in one call. Pass the destination file path and the template path explicitly:

```text
create_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    template_path="<templates-directory>/<template-name>.yaml",
    state_count=1,
    overwrite=False,
    # unity_scene_name defaults to Path(template_path).stem, i.e. the template's filename
    # without the .yaml extension. Override if the project uses a different scene name.
)
```

This loads the template via `TaskTemplate.from_yaml`, builds the configuration with
`create_experiment_configuration` (system fixed to `MESOSCOPE_VR`), then calls
`populate_default_experiment_states` with `state_count` to seed the `experiment_states` dict
with default-valued runtime states.

**Heads up:** the autopopulated states have `system_state_code=0`, which is not a valid
Mesoscope-VR hardware mode (Mesoscope-VR accepts `1` = REST or `2` = RUN). Each state must
be edited to set `system_state_code` to a valid value before the session can run. The
reinforcing and aversive guidance counter defaults are only populated for trial classes that
exist in `trial_structures`; absent trial classes leave their counters at `0`.

### Step 5: Customize state machine and trial parameters

Read the just-created configuration:

```text
read_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
)
```

Mutate the payload to override the fields the user wants to customize. The schema below applies to
`MesoscopeExperimentConfiguration` — currently the only concrete experiment-config subclass.
Future system-specific subclasses will likely follow a similar shape (template-derived VR fields,
state machine, trial structures), but consult `describe_experiment_configuration_schema_tool` with
the matching `acquisition_system` value rather than assuming the mesoscope schema applies verbatim.

- `trial_structures: dict[str, WaterRewardTrial | GasPuffTrial]` — a per-trial dict; each entry is
  either a `WaterRewardTrial` (with `reward_size_ul`, `reward_tone_duration_ms`) or a `GasPuffTrial`
  (with `puff_duration_ms`, `occupancy_duration_ms`). These are standalone dataclasses carrying
  **only** runtime parameters; the matching spatial fields (cue sequence, zones, trigger type) live
  on the paired `TaskTemplate`'s `trial_structures[<same name>]` and are joined at session init.
- `experiment_states: dict[str, ExperimentState]` — a dict, **not a list**. Access by string key
  (e.g. `experiment_states["state_1"].state_duration_s`), not by integer index. `populate_default_experiment_states`
  generates 1-indexed names (`state_1`, `state_2`, …); the first autopopulated state is `state_1`,
  not `state_0`. `ExperimentState` fields include `experiment_state_code`, `system_state_code`,
  `state_duration_s`, `supports_trials`, and the reinforcing/aversive guidance counters.
- `unity_scene_name: str` — also identifies the paired `TaskTemplate` YAML by filename stem.

There is no `trial_weights` field and no `water_reward_volume_uL` field — water-reward sizing lives
on `WaterRewardTrial.reward_size_ul`, and the relative frequency of trial types is determined by
the template's per-trial `transitions` (not by per-trial weights here).

Then write it back:

```text
write_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    configuration_payload={ ... },
    overwrite=True,
)
```

The kwarg is `configuration_payload` (not `configuration`).

### Step 6: Validate and verify

```text
validate_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
)
read_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
)
```

`validate_experiment_configuration_tool` loads the YAML through the matching experiment-config
`from_yaml` (today: `MesoscopeExperimentConfiguration.from_yaml`), which triggers `__post_init__`
validation:

The experiment configuration does not carry VR-side data, so the YAML loader performs only the
basic dataclass instantiation checks (correct field types, required fields present). Cross-template
validation (cue sequences, zone bounds, trigger-type pairing) is the responsibility of
`/task-templates` `validate_template_tool` on the paired VR configuration. At session init, the
acquisition runtime joins the two by trial name and validates that every `trial_structures` key in
the experiment configuration matches a key in the template.

On success the tool returns a `summary` (trial/state counts plus `unity_scene_name`); on failure it
returns an `issues` list. Fix any reported issues and re-write.

---

## Reading the frozen configuration from a session

After a session has run, the experiment configuration that was active at session start is captured
as a frozen YAML at `<session>/raw_data/experiment_configuration.yaml`. To read it:

```text
read_experiment_configuration_tool(
    file_path="<session>/raw_data/experiment_configuration.yaml",
)
```

This is a read-only operation. Do not attempt to write to the frozen file — modifying historical
session metadata is the responsibility of `/mesoscope-vr-snapshots` (which deals with position
snapshots, not the experiment config). If a frozen experiment config needs to be amended for some
reason, that is currently not supported by the sollertia-shared-assets MCP layer.

---

## Common patterns

| Goal                                  | Pattern                                                                                                                                                                                                                                                                                                          |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reuse a template across projects      | Call `create_experiment_configuration_tool` per project (one `file_path` per destination), then override per-project fields                                                                                                                                                                                      |
| Change reward volume for a trial type | Edit `trial_structures["<trial>"].reward_size_ul` for `WaterRewardTrial` entries                                                                                                                                                                                                                                 |
| Change gas-puff duration              | Edit `trial_structures["<trial>"].puff_duration_ms` for `GasPuffTrial` entries                                                                                                                                                                                                                                   |
| Adjust a state's duration             | Edit `experiment_states["<state-key>"].state_duration_s` (state machine is a dict)                                                                                                                                                                                                                               |
| Add a new state to the state machine  | Add a new key to the `experiment_states` dict, then re-validate                                                                                                                                                                                                                                                  |
| Add a new spatial trial entry         | First hand off to `/task-templates` to add the `TrialStructure` to the template, then either re-run `create_experiment_configuration_tool` with `overwrite=True` or amend this skill's experiment config via `write_experiment_configuration_tool` to add the matching `WaterRewardTrial` / `GasPuffTrial` entry |

### Migrating an experiment to a new template

1. Read the old configuration with `read_experiment_configuration_tool(file_path=...)`.
2. If the new template does not exist, hand off to `/task-templates` to author it.
3. Call `create_experiment_configuration_tool(file_path=..., template_path=...)` pointing at the new
   template.
4. Port the customizations (state durations, per-trial reward sizes, puff and occupancy durations,
   guidance counters) over manually.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Target project directory exists (create via `/project-hierarchy`'s `create_project_tool` or the `slsa configure project` CLI if missing)
- [ ] Target template exists (handed off to /task-templates if missing) and template_path is known
- [ ] describe_experiment_configuration_schema_tool was used as the source of truth for field names
- [ ] file_path was passed to every read/write/validate/create call (absolute path)
- [ ] Payload was passed as configuration_payload (the correct kwarg name)
- [ ] write_experiment_configuration_tool succeeded without schema errors
- [ ] validate_experiment_configuration_tool returned valid=True with no issues
- [ ] read_experiment_configuration_tool returned the expected configuration after the write
- [ ] experiment_states was treated as a dict (string keys), not a list (integer indices)
- [ ] No reference to a non-existent trial_weights or water_reward_volume_uL field
- [ ] Reward sizes (reward_size_ul) and state durations are within plausible biological ranges
- [ ] Did not call write_template_tool from this skill
```

---

## Related skills

| Skill                                     | Relationship                                                                                                                                               |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/working-directory`                      | Provides the templates directory so `/task-templates` knows where to enumerate; this skill needs only absolute paths                                       |
| `/assets-mcp-environment-setup`           | Run first if the MCP server is not connected                                                                                                               |
| `/task-templates`                         | Required upstream — owns template authoring and exposes `discover_templates_tool` for absolute template paths                                              |
| `/project-hierarchy`                      | Discovers the project tree; owns project creation (`create_project_tool` / `slsa configure project`)                                                       |
| experiment plugin `/system-configuration` | Owns MesoscopeSystemConfiguration (moved out of this plugin)                                                                                               |
| experiment plugin `/data-management`      | Downstream consumer — `SessionData.create` copies the authored `experiment_configuration.yaml` into every new experiment session at acquisition time       |
| unity plugin `/task-prefabs`              | Validates template values against the Unity prefab state                                                                                                   |
| experiment plugin `/experiment-pipeline`  | Phase 4 of the experiment lifecycle is owned by this skill                                                                                                 |
| `/library-extension`                      | Cross-cutting recipe to add a new `AcquisitionSystems`, runtime trial class, or `TriggerType` member; lists the prose here that needs updating in lockstep |
| experiment plugin `/vr-driver-interface`  | Verifies `unity_scene_name` against the live scene; consumes the per-trial parameters at runtime                                                           |
