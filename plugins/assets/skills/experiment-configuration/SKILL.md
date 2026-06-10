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
today is `MesoscopeExperimentConfiguration`. The configuration class is resolved through
`EXPERIMENT_CONFIGURATION_REGISTRY` keyed by `AcquisitionSystems`, and both are deliberately
extensible — additional systems may be added in the future, at which point this skill will own
their experiment configurations as well. This skill is the **exclusive** owner of:

- `write_experiment_configuration_tool`
- `create_experiment_from_vr_template_tool`
- `describe_experiment_configuration_schema_tool`
- `validate_experiment_configuration_tool`

No other skill in the marketplace may call these tools.

Every Sollertia acquisition system runs in Virtual Reality, presenting a Unity task in the linear infinite
corridor, so every experiment configuration is seeded from a corridor task template and satisfies one stable
contract (`experiment_states`, `trial_structures`, `unity_scene_name`, and the `from_task_template` builder). The
final form is system-specific: `MesoscopeExperimentConfiguration` is the only concrete subclass today, and a new
acquisition system contributes its **own** subclass (by extending `sollertia-shared-assets`; see
`/library-extension`) that satisfies the contract and expands it with its own fields and trial classes.

---

## Scope

**Covers:**
- Authoring or repairing a full experiment configuration payload for any acquisition system via
  `write_experiment_configuration_tool`
- Seeding an experiment configuration from a Unity VR task template via
  `create_experiment_from_vr_template_tool`
- Experiment state machines (`ExperimentState`, seeded by
  `MesoscopeExperimentConfiguration.from_task_template`)
- Schema introspection for experiment configurations
- Reading the frozen experiment configuration captured at session start (pass the per-session
  snapshot path to `read_experiment_configuration_tool`)

**Does not cover:**
- Authoring task templates themselves (see `/task-templates`)
- Authoring system-level configuration (see the experiment plugin's `/acquisition-system-design`)
- Authoring server configuration (see forging plugin's `/server-configuration`)
- Creating projects (see `/project-hierarchy`)
- Verifying that template values match the Unity prefab state (see unity plugin's `/task-prefabs`)
- Initial working directory setup (see `/working-directory`)

---

## Templates vs experiment configurations

A **task template** (`TaskTemplate`) is the corridor task asset every experiment is seeded from. It is project-
and system-agnostic — the same template can back many experiment configurations across many projects.

| Concept                            | What it is                                         | Owning skill      |
|------------------------------------|----------------------------------------------------|-------------------|
| `TaskTemplate`                     | Reusable corridor environment, trials, cue catalog | `/task-templates` |
| System-specific experiment config  | Per-project state machine + per-trial parameters   | this skill        |
| `MesoscopeExperimentConfiguration` | The only concrete experiment-config subclass today | this skill        |

The template defines **what is possible**, and the experiment configuration picks a template (by
`unity_scene_name`) and parameterizes it (state durations, reward volumes, project-specific overrides). The
template and the experiment configuration are owned by two different skills.

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

`ExperimentState.supports_trials` (default `True`) is **not** a runtime control. The acquisition
runtime always drives experiment-state behavior through hardware via `system_state_code`; it never
consults this flag to decide whether trials run. The field is metadata for the downstream forging and
analysis pipelines (`sollertia-forgery`): it records whether a phase is *expected* to contain trials,
so dataset forging and analysis know whether to look for and process trial data for that phase. A
trial-free phase is realized by choosing a `system_state_code` whose hardware mode drives no trials
(e.g. REST on Mesoscope-VR); set `supports_trials` to match (`False`) so the forging/analysis side
reads the phase correctly.

### Where the trial sequence comes from

The experiment configuration **never enumerates or schedules trials** — it contributes only the
**per-trial-type parameters** (reward volume, tone duration, puff duration). The trial
sequence is owned by Unity (`sollertia-unity-tasks`): at session init the acquisition runtime requests a cue
sequence materialized from the template's per-trial `transitions` (see `/task-templates`), then identifies trial
boundaries by motif matching against each `TrialStructure`. Relative frequencies are encoded in the template's
transition probabilities. The runtime joins each decomposed trial name back to this configuration's
`trial_structures` to attach the per-trial parameters. The configuration carries no schedule; it carries
per-trial parameters and the phase-level state machine.

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

### The experiment configuration contract

Every `<System>ExperimentConfiguration` shares one **stable contract** — `experiment_states`,
`trial_structures`, `unity_scene_name`, and the `from_task_template` builder. Everything else is
**system-specific**: the concrete class adds whatever further fields its acquisition system needs and defines its
own runtime trial classes, so the contract is the floor, not the whole schema. Always read the actual field set
from `describe_experiment_configuration_schema_tool` for the system you target — its `nested_classes` are
introspected from the resolved class and reflect that system's own dataclasses. The bullets below name the
contract members; `MesoscopeExperimentConfiguration` is the concrete example, and its trial classes and pairings
are Mesoscope-VR's own — never assume they apply verbatim to another system.

- **`experiment_states: dict[str, ExperimentState]`** — the experiment state machine, required on every subclass
  because every experiment is a state machine.
- **`trial_structures`** — the trials the experiment runs, required on every subclass: a per-trial dict whose
  values are the **system's own** runtime trial classes. The task template provides each trial's spatial
  `TrialStructure` and the configuration pairs it with a runtime trial class by trial name.
  `list_supported_trial_types_tool` derives the trial list from this field. The platform `trigger_type` taxonomy
  has **five** modes (`interaction`, `collision`, `occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`), but
  each acquisition system maps only the **subset** it supports. The Mesoscope-VR system's annotation for example is
  `dict[str, WaterRewardTrial | GasPuffTrial]`, and its `from_task_template` maps `trigger_type: "interaction"`
  → `WaterRewardTrial` and `trigger_type: "occupancy_disarm"` → `GasPuffTrial` to match the zone prefab Unity
  instantiates; `collision`, `occupancy_arm`, and `occupancy_trigger` are not mapped on Mesoscope-VR, so a config that
  uses one raises a clear "not mapped to a runtime trial class" error. Another system defines and pairs its own trial
  classes for whatever subset it supports.
- **`unity_scene_name`** — a mandatory contract field. It identifies the paired `TaskTemplate` by filename stem
  and is verified against the scene loaded in Unity at session start, so two projects can point the same template
  at differently-named scene files.

---

## MCP tool surface

| Tool                                            | Purpose                                                                                                            |
|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `discover_experiments_tool`                     | Lists experiment configurations under a project                                                                    |
| `describe_experiment_configuration_schema_tool` | Returns the field schema for the experiment dataclass                                                              |
| `read_experiment_configuration_tool`            | Reads an experiment configuration YAML from any canonical location (project source or per-session frozen snapshot) |
| `write_experiment_configuration_tool`           | Writes a validated experiment configuration payload for any system — author or repair (exclusive)                  |
| `create_experiment_from_vr_template_tool`       | Creates an experiment configuration from a Unity VR task template (exclusive)                                      |
| `validate_experiment_configuration_tool`        | Validates an experiment configuration YAML (exclusive)                                                             |
| `list_supported_acquisition_systems_tool`       | Enumerates `AcquisitionSystems`                                                                                    |

---

## Path conventions

All read / write / validate / create tools in this skill take **explicit file paths** plus a required
**`acquisition_system`** — the experiment-config dataclass is selected per system through the registry.
The caller resolves both; the tools never consult `root_directory`, a project name, an experiment name,
or a template name. The canonical paths are:

| Asset                                         | Canonical path                                     |
|-----------------------------------------------|----------------------------------------------------|
| Per-project experiment configuration          | `<root>/<project>/configuration/<experiment>.yaml` |
| Task template                                 | `<templates-directory>/<template-name>.yaml`       |
| Per-session frozen experiment-config snapshot | `<session>/raw_data/experiment_configuration.yaml` |

Use `discover_experiments_tool(root_directory=..., project=...)` to enumerate existing configs and
their absolute paths; use `discover_templates_tool()` to enumerate template paths.

**Resolving `acquisition_system`.** It is required — there is no default. Resolve it **automatically**
wherever possible and prompt the user only as a fallback. For a **per-session snapshot**, read it from
the session's own `SessionData` — `inspect_sessions_tool` (`/session-data`) reports
`identity.acquisition_system`, or read the marker directly via `read_session_data_tool`. For
**per-project authoring**, use the system the project/host targets (enumerate options with
`list_supported_acquisition_systems_tool`). Pass the resolved value to every tool below.

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
- The target task template exists at a known path. If it doesn't, hand off to `/task-templates` to author
  it — this skill must not call `write_template_tool` directly. The templates directory can be enumerated via
  `discover_templates_tool`, which also returns absolute paths.

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
describe_experiment_configuration_schema_tool(acquisition_system="<system>")
```

Use the schema as the source of truth for field names and nesting. It always reports the contract fields
(`experiment_states` and `trial_structures`) plus whatever system-specific fields the resolved subclass adds.
Its `nested_classes` are derived from the resolved configuration class, so they reflect that system's actual
nested dataclasses — never assume the Mesoscope-VR set applies verbatim.

### Step 4: Create the configuration

Seed the configuration from a Unity VR task template with `create_experiment_from_vr_template_tool`. The tool
dispatches through `EXPERIMENT_CONFIGURATION_REGISTRY` (keyed by `AcquisitionSystems`) to the resolved
`<System>ExperimentConfiguration` and calls its `from_task_template` builder. To author or repair a full payload
directly — without re-seeding from the template — use `write_experiment_configuration_tool` (see Step 5).

Pass the destination path and the template path explicitly:

```text
create_experiment_from_vr_template_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    acquisition_system="<system>",  # resolves the config class via EXPERIMENT_CONFIGURATION_REGISTRY (resolve it)
    template_path="<templates-directory>/<template-name>.yaml",
    state_count=1,
    overwrite=False,
    # The embedded Unity scene name is inferred from the template filename stem
    # (the filename without the .yaml extension).
)
```

This loads the template via `TaskTemplate.from_yaml`, then calls the resolved config class's
`from_task_template` classmethod (today: `MesoscopeExperimentConfiguration.from_task_template`). That classmethod
maps the template's trial structures to runtime trials and seeds `state_count` default-valued runtime states in the
`experiment_states` dict.

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
    acquisition_system="<system>",
)
```

Mutate the payload to override the fields the user wants to customize. The schema below describes
`MesoscopeExperimentConfiguration`, currently the only concrete experiment-config subclass; consult
`describe_experiment_configuration_schema_tool` with the matching `acquisition_system` value for any other system.

- `trial_structures: dict[str, WaterRewardTrial | GasPuffTrial]` (Mesoscope-VR's exemplar annotation) — a
  per-trial dict; each entry is either a `WaterRewardTrial` (with `reward_size_ul`, `reward_tone_duration_ms`)
  or a `GasPuffTrial` (with `puff_duration_ms`). These are standalone, exemplar-specific
  dataclasses carrying **only** runtime parameters; another system declares its own trial classes. The matching
  spatial fields (cue sequence, zones, trigger type, occupancy duration) live on the paired `TaskTemplate`'s
  `trial_structures[<same name>]` and are joined at session init.
- `experiment_states: dict[str, ExperimentState]` — a dict, **not a list**. Access by string key
  (e.g. `experiment_states["state_1"].state_duration_s`), not by integer index. `from_task_template`
  generates 1-indexed names (`state_1`, `state_2`, …); the first autopopulated state is `state_1`,
  not `state_0`. `ExperimentState` fields include `experiment_state_code`, `system_state_code`,
  `state_duration_s`, `supports_trials`, and the reinforcing/aversive guidance counters.
- `unity_scene_name: str` — a mandatory contract field identifying the paired `TaskTemplate` YAML by filename
  stem.

Then write it back:

```text
write_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    acquisition_system="<system>",
    configuration_payload={ ... },
    overwrite=True,
)
```

The kwarg is `configuration_payload` (not `configuration`).

### Step 6: Validate and verify

```text
validate_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    acquisition_system="<system>",
)
read_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    acquisition_system="<system>",
)
```

`validate_experiment_configuration_tool` loads the YAML through the matching experiment-config
`from_yaml` (today: `MesoscopeExperimentConfiguration.from_yaml`), which triggers `__post_init__`
validation:

The experiment configuration does not carry the corridor task's spatial data, so the YAML loader performs only
the basic dataclass instantiation checks (correct field types, required fields present). Cross-template
validation (cue sequences, zone bounds, trigger-type pairing) is the responsibility of `/task-templates`
`validate_template_tool` on the paired template. At session init the acquisition runtime joins the two by trial
name, validating that every `trial_structures` key matches a key in the template.

On success the tool returns a `summary` (trial/state counts plus `unity_scene_name`); on failure it returns an
`issues` list. Fix any reported issues and re-write.

---

## Reading the frozen configuration from a session

After a session has run, the experiment configuration that was active at session start is captured
as a frozen YAML at `<session>/raw_data/experiment_configuration.yaml`. To read it:

```text
read_experiment_configuration_tool(
    file_path="<session>/raw_data/experiment_configuration.yaml",
    acquisition_system="<system>",  # read from the session's own SessionData (identity.acquisition_system)
)
```

This is a read-only operation. Do not attempt to write to the frozen file — modifying historical
session metadata is the responsibility of `/mesoscope-vr-snapshots` (which deals with position
snapshots, not the experiment config). If a frozen experiment config needs to be amended for some
reason, that is currently not supported by the sollertia-shared-assets MCP layer.

---

## Common patterns

| Goal                                  | Pattern                                                                                                                                                                                                                                                                                                             |
|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reuse a template across projects      | Call `create_experiment_from_vr_template_tool` per project (one `file_path` per destination), then override per-project fields                                                                                                                                                                                      |
| Change reward volume for a trial type | Edit `trial_structures["<trial>"].reward_size_ul` for `WaterRewardTrial` entries                                                                                                                                                                                                                                    |
| Change gas-puff duration              | Edit `trial_structures["<trial>"].puff_duration_ms` for `GasPuffTrial` entries                                                                                                                                                                                                                                      |
| Adjust a state's duration             | Edit `experiment_states["<state-key>"].state_duration_s` (state machine is a dict)                                                                                                                                                                                                                                  |
| Add a new state to the state machine  | Add a new key to the `experiment_states` dict, then re-validate                                                                                                                                                                                                                                                     |
| Add a new spatial trial entry         | First hand off to `/task-templates` to add the `TrialStructure` to the template, then either re-run `create_experiment_from_vr_template_tool` with `overwrite=True` or amend this skill's experiment config via `write_experiment_configuration_tool` to add the matching `WaterRewardTrial` / `GasPuffTrial` entry |

### Moving an experiment to a new template

1. Read the old configuration with `read_experiment_configuration_tool(file_path=...)`.
2. If the new template does not exist, hand off to `/task-templates` to author it.
3. Call `create_experiment_from_vr_template_tool(file_path=..., template_path=...)` pointing at the new
   template.
4. Port the customizations (state durations, per-trial reward sizes, puff durations,
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

| Skill                                          | Relationship                                                                                                                                                                                                                                                                |
|------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/working-directory`                           | Provides the templates directory so `/task-templates` knows where to enumerate; this skill needs only absolute paths                                                                                                                                                        |
| `/assets-mcp-environment-setup`                | Run first if the MCP server is not connected                                                                                                                                                                                                                                |
| `/task-templates`                              | Required dependency — owns corridor template authoring and exposes `discover_templates_tool` for absolute template paths                                                                                                                                                    |
| `/project-hierarchy`                           | Discovers the project tree; owns project creation (`create_project_tool` / `slsa configure project`)                                                                                                                                                                        |
| experiment plugin `/acquisition-system-design` | Documents the per-system system-configuration pattern (Mesoscope-VR instance: `MesoscopeSystemConfiguration`)                                                                                                                                                               |
| experiment plugin `/data-management`           | Downstream consumer — `SessionData.create` copies the authored `experiment_configuration.yaml` into every new experiment session at acquisition time                                                                                                                        |
| unity plugin `/task-prefabs`                   | Validates template values against the Unity prefab state                                                                                                                                                                                                                    |
| experiment plugin `/pipeline`                  | Phase 4 of the experiment lifecycle (experiment authoring) hands off to this skill                                                                                                                                                                                          |
| `/library-extension`                           | Cross-cutting recipe to add a new `AcquisitionSystems`, runtime trial class, or `TriggerType` member (a new `TriggerType` member does **not** require a `from_task_template` branch — a system may leave it unmapped); lists the prose here that needs updating in lockstep |
| experiment plugin `/vr-driver-interface`       | Verifies `unity_scene_name` against the live scene; consumes the per-trial parameters at runtime                                                                                                                                                                            |
