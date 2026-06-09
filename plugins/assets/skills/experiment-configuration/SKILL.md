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

An experiment configuration is a **standalone** asset. Its core — the experiment state machine
(`ExperimentState`) and the per-trial parameter structures (e.g. `WaterRewardTrial`, `GasPuffTrial`) carries 
**no Virtual Reality data**. VR is **not** intrinsic to an experiment
configuration; it enters only through a concrete subclass that opts into it. The sole subclass today,
`MesoscopeExperimentConfiguration`, opts in via its `unity_scene_name` field because Mesoscope-VR runs
a Unity task — but a new acquisition system contributes its **own** subclass (by extending
`sollertia-shared-assets`; see `/library-extension`) that may omit VR entirely and drive trials at the
acquisition-runtime (`sollertia-experiment`) level instead.

---

## Scope

**Covers:**
- Authoring a full experiment configuration payload for any acquisition system via
  `write_experiment_configuration_tool` — the generic create / customize / repair path
- Seeding an experiment configuration from a Unity VR task template via
  `create_experiment_from_vr_template_tool` (shared across systems that use Unity VR tasks)
- Experiment state machines (`ExperimentState`, seeded by
  `MesoscopeExperimentConfiguration.from_task_template`)
- Schema introspection for experiment configurations
- Reading the frozen experiment configuration captured at session start (pass the per-session
  snapshot path to `read_experiment_configuration_tool`)

**Does not cover:**
- Authoring task templates themselves (see `/task-templates`)
- Authoring system-level configuration (see experiment plugin's `/system-configuration`)
- Authoring server configuration (see forging plugin's `/server-configuration`)
- Creating projects (see `/project-hierarchy`)
- Verifying that template values match the Unity prefab state (see unity plugin's `/task-prefabs`)
- Initial working directory setup (see `/working-directory`)

---

## Templates vs experiment configurations (VR only)

A **task template** (`TaskTemplate`) is a **Virtual-Reality-only, optional** asset. It exists solely for
systems that use Unity VR tasks; a system that does not use Unity VR tasks authors **no** template at all.
Among systems that use Unity VR tasks it is project- and system-agnostic — the same template can back many
experiment configurations across many projects.

| Concept                            | What it is                                                                        | Owning skill      |
|------------------------------------|-----------------------------------------------------------------------------------|-------------------|
| `TaskTemplate`                     | Reusable VR environment, trial structure, cue catalog (systems using Unity VR)    | `/task-templates` |
| System-specific experiment config  | Per-project state machine + per-trial parameters                                  | this skill        |
| `MesoscopeExperimentConfiguration` | The only concrete experiment-config subclass today (uses Unity VR)                | this skill        |

For a VR experiment the template defines **what is possible**, and the experiment configuration picks a
template (by `unity_scene_name`) and parameterizes it (state durations, reward volumes, project-specific
overrides). For a non-VR experiment there is no template: the experiment configuration stands alone and
its trial structures are consumed directly by the acquisition runtime. Either way, the template (when
present) and the experiment configuration are owned by two different skills.

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

### Where the trial sequence comes from depends on the system

The experiment configuration **never enumerates or schedules trials** — it contributes only the
**per-trial-type parameters** (reward volume, tone duration, puff duration, occupancy threshold).
*What* drives the trial sequence differs by acquisition system:

- **Systems that use Unity VR tasks (e.g. Mesoscope-VR).** The trial sequence is owned by Unity
  (`sollertia-unity-tasks`): at session init the acquisition runtime requests a cue sequence materialized
  from the template's per-trial `transitions` (see `/task-templates`), then identifies trial boundaries by
  motif matching against each `TrialStructure`. Relative frequencies are encoded in the template's transition
  probabilities. The runtime joins each decomposed trial name back to this configuration's 
  `trial_structures` to attach the per-trial parameters.
- **Systems that do not use Unity VR tasks.** There is no template and no Unity cue sequence. Trial handling
  lives in the acquisition runtime (`sollertia-experiment`) itself, which consumes the experiment
  configuration's `trial_structures` and `experiment_states` directly. This is not hypothetical: `sle` already
  runs fully VR-free sessions today (lick-training, run-training, window-checking) that build no VR driver, so
  an *experiment* system that does not use Unity VR tasks follows the same pattern — its own experiment-config
  subclass, processed at the `sle` level, with no external (VR-driven) trial mechanism.

In neither case does the experiment configuration carry a schedule; it carries per-trial parameters
and the phase-level state machine.

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

Every `<System>ExperimentConfiguration` shares one contract — two required fields — and adds whatever
system-specific fields its acquisition system needs. Mesoscope-VR is one exemplar: its trial classes and its
`unity_scene_name` are exemplar-specific, so always read the actual field set from
`describe_experiment_configuration_schema_tool` for the system you target. Its `nested_classes` are derived from
the resolved configuration class by introspection, so they reflect that system's actual nested dataclasses.

- **`experiment_states: dict[str, ExperimentState]`** — the experiment state machine, required on every subclass
  because every experiment is a state machine.
- **`trial_structures`** — the trials the experiment runs, required on every subclass: a per-trial dict whose
  values are the system's runtime trial classes (Mesoscope-VR's annotation is
  `dict[str, WaterRewardTrial | GasPuffTrial]`). On a system that uses Unity VR tasks, the task template provides
  each trial's spatial `TrialStructure` and the configuration pairs it with a runtime trial class. The natural
  pairing is `trigger_type: "lick"` → `WaterRewardTrial` and `trigger_type: "occupancy"` → `GasPuffTrial`, since
  the template's `trigger_type` selects the zone prefab Unity instantiates (cross-pairing is schema-legal but
  mismatches the prefab). On a system that does not use Unity VR tasks, these entries stand alone as the
  runtime's trial parameter table. `list_supported_trial_types_tool` derives the trial list from this field.
- **`unity_scene_name`** — a system-specific field that systems using Unity VR tasks add. It identifies the
  paired `TaskTemplate` by filename stem and is verified against the scene loaded in Unity at session start, so
  two projects can point the same template at differently-named scene files. A system that does not use Unity VR
  tasks omits it, which opts out of VR template export.

---

## MCP tool surface

| Tool                                            | Purpose                                                                                                            |
|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `discover_experiments_tool`                     | Lists experiment configurations under a project                                                                    |
| `describe_experiment_configuration_schema_tool` | Returns the field schema for the experiment dataclass                                                              |
| `read_experiment_configuration_tool`            | Reads an experiment configuration YAML from any canonical location (project source or per-session frozen snapshot) |
| `write_experiment_configuration_tool`           | Writes a validated experiment configuration payload for any system — create, customize, or repair (exclusive)      |
| `create_experiment_from_vr_template_tool`       | Creates an experiment configuration from a Unity VR task template (shared by systems using Unity VR; exclusive)    |
| `validate_experiment_configuration_tool`        | Validates an experiment configuration YAML (exclusive)                                                             |
| `list_supported_acquisition_systems_tool`       | Enumerates `AcquisitionSystems`, each with a `supports_template_creation` flag                                     |

---

## Path conventions

All read / write / validate / create tools in this skill take **explicit file paths** plus a required
**`acquisition_system`** — the experiment-config dataclass is selected per system through the registry.
The caller resolves both; the tools never consult `root_directory`, a project name, an experiment name,
or a template name. The canonical paths are:

| Asset                                         | Canonical path                                                 |
|-----------------------------------------------|----------------------------------------------------------------|
| Per-project experiment configuration          | `<root>/<project>/configuration/<experiment>.yaml`             |
| Task template                                 | `<templates-directory>/<template-name>.yaml`                   |
| Per-session frozen experiment-config snapshot | `<session>/raw_data/experiment_configuration.yaml`             |

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
- **For VR experiments only:** the target task template exists at a known path. If it doesn't, hand
  off to `/task-templates` to author it — this skill must not call `write_template_tool` directly. The
  templates directory can be enumerated via `discover_templates_tool`, which also returns absolute
  paths. Non-VR experiments skip this prerequisite entirely (there is no template).

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

There are two ways to mint an experiment configuration:

- `write_experiment_configuration_tool` — the **generic** path. It authors a full payload, validated against the
  registered `<System>ExperimentConfiguration`, for **any** acquisition system. No template and no factory are
  involved, and it is always available. Use it to author a configuration for a system that has no Unity VR template.
- `create_experiment_from_vr_template_tool` — seeds a configuration from a Unity VR task template, for systems whose
  configuration is built from a VR template. This tool and `TaskTemplate` are shared by every system that uses Unity
  VR tasks; such a system reuses them by adding a `from_task_template` classmethod to its own
  `<System>ExperimentConfiguration` dataclass and registering that class in `VR_TEMPLATE_CONFIG_REGISTRY` (keyed by its
  `AcquisitionSystems` member). The tool dispatches through that registry; a system absent from it falls back to
  `write_experiment_configuration_tool`. A system that builds its configuration from different inputs adds its own
  dedicated creation tool and a matching creation classmethod instead.

For a system that runs a Unity VR task, pass the destination path and the template path explicitly:

```text
create_experiment_from_vr_template_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    acquisition_system="<system>",  # resolves the config class via VR_TEMPLATE_CONFIG_REGISTRY (resolve it)
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
  or a `GasPuffTrial` (with `puff_duration_ms`, `occupancy_duration_ms`). These are standalone, exemplar-specific
  dataclasses carrying **only** runtime parameters; another system declares its own trial classes. On a system
  that uses Unity VR tasks the matching spatial fields (cue sequence, zones, trigger type) live on the paired
  `TaskTemplate`'s `trial_structures[<same name>]` and are joined at session init; on a system that does not use
  Unity VR tasks there is no template and these entries are the runtime's trial parameter table on their own.
- `experiment_states: dict[str, ExperimentState]` — a dict, **not a list**. Access by string key
  (e.g. `experiment_states["state_1"].state_duration_s`), not by integer index. `from_task_template`
  generates 1-indexed names (`state_1`, `state_2`, …); the first autopopulated state is `state_1`,
  not `state_0`. `ExperimentState` fields include `experiment_state_code`, `system_state_code`,
  `state_duration_s`, `supports_trials`, and the reinforcing/aversive guidance counters.
- `unity_scene_name: str` (added by subclasses of systems that use Unity VR tasks; Mesoscope-VR is the only one
  today) — identifies the paired `TaskTemplate` YAML by filename stem. A system that does not use Unity VR tasks
  omits this field from its subclass.

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

The experiment configuration does not carry VR-side data, so the YAML loader performs only the
basic dataclass instantiation checks (correct field types, required fields present). For VR
experiments, cross-template validation (cue sequences, zone bounds, trigger-type pairing) is the
responsibility of `/task-templates` `validate_template_tool` on the paired VR configuration. At
session init the acquisition runtime joins the two by trial name, validating that every
`trial_structures` key matches a key in the template. For non-VR experiments there is no template and
no such join — the `trial_structures` table stands on its own and is consumed directly by the
acquisition runtime.

On success the tool returns a `summary` (trial/state counts plus, for the Mesoscope-VR subclass,
`unity_scene_name`); on failure it returns an `issues` list. Fix any reported issues and re-write.

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

### Migrating an experiment to a new template (VR experiments only)

For non-VR experiments there is no template; edit the experiment configuration directly via
`write_experiment_configuration_tool`.

1. Read the old configuration with `read_experiment_configuration_tool(file_path=...)`.
2. If the new template does not exist, hand off to `/task-templates` to author it.
3. Call `create_experiment_from_vr_template_tool(file_path=..., template_path=...)` pointing at the new
   template.
4. Port the customizations (state durations, per-trial reward sizes, puff and occupancy durations,
   guidance counters) over manually.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Target project directory exists (create via `/project-hierarchy`'s `create_project_tool` or the `slsa configure project` CLI if missing)
- [ ] For VR experiments: target template exists (handed off to /task-templates if missing) and template_path is known (non-VR experiments use no template)
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
| `/task-templates`                         | Optional, VR experiments only — owns template authoring and exposes `discover_templates_tool` for absolute template paths (non-VR experiments use none)    |
| `/project-hierarchy`                      | Discovers the project tree; owns project creation (`create_project_tool` / `slsa configure project`)                                                       |
| experiment plugin `/system-configuration` | Owns MesoscopeSystemConfiguration (moved out of this plugin)                                                                                               |
| experiment plugin `/data-management`      | Downstream consumer — `SessionData.create` copies the authored `experiment_configuration.yaml` into every new experiment session at acquisition time       |
| unity plugin `/task-prefabs`              | Validates template values against the Unity prefab state                                                                                                   |
| experiment plugin `/experiment-pipeline`  | Phase 4 of the experiment lifecycle is owned by this skill                                                                                                 |
| `/library-extension`                      | Cross-cutting recipe to add a new `AcquisitionSystems`, runtime trial class, or `TriggerType` member; lists the prose here that needs updating in lockstep |
| experiment plugin `/vr-driver-interface`  | Verifies `unity_scene_name` against the live scene; consumes the per-trial parameters at runtime                                                           |
