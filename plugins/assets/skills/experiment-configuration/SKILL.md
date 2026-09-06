---
name: experiment-configuration
description: >-
  Authors per-project, per-system experiment configuration YAMLs via the sollertia-shared-assets MCP server. Owns the
  system-agnostic create, write, delete, and validate tools, the EXPERIMENT_CONFIGURATION_REGISTRY dispatch keyed by
  AcquisitionSystems, and the generic configuration contract. Use when creating a configuration, customizing per-trial
  parameters, or instantiating a task template. For a system's concrete field schema, defer to its schema skill (for
  Mesoscope-VR, mesoscope:mesoscope-vr-experiment-schema).
user-invocable: false
---

# Sollertia experiment configuration

Authors and modifies per-project, system-specific experiment configuration YAML files for `sollertia-shared-assets`
using the `slsa mcp` MCP server. Both `EXPERIMENT_CONFIGURATION_REGISTRY` and the configuration base are deliberately
extensible, so additional systems contribute their own subclasses, and this skill owns the system-agnostic tooling for
all of them. This skill is the **exclusive** owner of every tool the MCP tool surface below marks `(exclusive)`, and no
other skill in the marketplace may call one of them.

Every Sollertia acquisition system runs in Virtual Reality, presenting a Unity task in the linear infinite corridor.
Therefore, every experiment configuration is seeded from a corridor task template and satisfies one stable contract
(`experiment_states`, `trial_structures`, `unity_scene_name`, and the `from_task_template` builder). The final form is
system-specific, because each acquisition system contributes its **own** `<System>ExperimentConfiguration` subclass by
extending `sollertia-shared-assets` as `/library-extension` describes, satisfying the contract and expanding it with its
own fields and trial classes. This skill stays generic, so read `describe_experiment_configuration_schema_tool` for the
target system and defer to that system's schema skill for its concrete field-level schema, trial classes, per-field
defaults, and trigger-type mapping (for Mesoscope-VR, see `mesoscope:mesoscope-vr-experiment-schema`).

---

## Scope

**Covers:**
- Authoring, repairing, or removing a full experiment configuration for any acquisition system via
  `write_experiment_configuration_tool` and `delete_experiment_configuration_tool`
- Seeding an experiment configuration from a Unity VR task template via `create_experiment_from_vr_template_tool`
- Experiment state machines (`ExperimentState`, seeded by the resolved subclass's `from_task_template` builder)
- Schema introspection for experiment configurations
- Reading the frozen experiment configuration captured at session start (pass the per-session snapshot path to
  `read_experiment_configuration_tool`)

**Does not cover:**
- Authoring task templates themselves (see `/task-templates`)
- Authoring system-level configuration (see `experiment:acquisition-system-design`)
- Authoring server configuration (see `forging:server-configuration`)
- Creating projects (see `/project-hierarchy`)
- Verifying that template values match the Unity prefab state (see `unity:task-prefabs`)
- Initial working directory setup (see `/working-directory`)

---

## Templates vs. experiment configurations

`/task-templates` owns this boundary and states what a `TaskTemplate` carries, so read the template side there.

The configuration side is what this skill owns. An experiment configuration is per-project and system-specific. It
selects its template by `unity_scene_name` and parameterizes it, contributing the phase-level state machine
(`experiment_states`) and the per-trial runtime parameters (`trial_structures`). It enumerates no trials and carries no
spatial data of its own, so the template defines what is possible and the configuration decides how it is run.

---

## How experiment configurations are executed at runtime

An experiment configuration is the contract between the authored definition of what a session should do and the runtime
that conducts it. The description below is conceptual, so defer to the runtime package's own documentation for the
precise call sequence and tool surface of any specific consumer.

### The state machine

`experiment_states` is a **`dict[str, ExperimentState]`** iterated in **insertion order**. That ordering is the sequence
in which states fire, and the string keys give each state a stable, human-readable identifier for analysis and for the
runtime's console output, without forcing positional indexing. The key format is set by the target system's
`from_task_template` builder rather than by the platform, and Mesoscope-VR emits 1-indexed `state_1`, `state_2`, and so
on (see `mesoscope:mesoscope-vr-experiment-schema`). Each state holds for `state_duration_s` seconds, then control falls
through to the next state. The session ends when the last state's timer expires.

The dict-of-named-states shape supports adding, renaming, and reordering states by editing keys and re-emitting the
YAML. The keys themselves never reach the log stream, because the runtime logs only the integer `experiment_state_code`
and `sollertia-forgery` joins the `experiment_states` key back in by that code when it builds the `runtime_state`
column. That join is self-contained within one session, because the forging pipeline reads the code-to-name mapping from
that session's own frozen `experiment_configuration.yaml` snapshot rather than from the live project configuration. No
later edit to the project configuration disturbs a session already acquired, and amending the session's own snapshot is
a separate matter, covered under "Reading the frozen configuration from a session" below. The edit that does carry a
cost is the rename, because the key becomes the user-visible category value of the forged dataset's `runtime_state`
column. Renaming a state relabels it in every session acquired afterwards and breaks label comparability against the
sessions acquired before. Renumbering `experiment_state_code` is invisible to analysis by comparison, since the code is
replaced by its key on the way into the dataset, though the numbering is constrained at both ends (see the field
description below).

`ExperimentState` declares three required fields with no default (`experiment_state_code`, `system_state_code`,
`state_duration_s`), `supports_trials` defaulting to `True`, and six guidance counters each defaulting to `0`. Its
`__post_init__` raises `ValueError` when `state_duration_s` is not positive and finite, refusing `0`, a negative
duration, an infinity, and a `NaN` alike, and again when any of the six guidance counters holds a negative value. The
check runs on construction, so a hand-authored YAML fails through `from_yaml` as a written payload does. Sign and
finiteness are its whole reach, so the plausible-range check in the verification checklist stays agent-side.

### Experiment states vs. system states

Each `ExperimentState` carries two distinct codes:

- **`experiment_state_code: int`** is the phase's unique integer identifier code, required with no default, and emitted
  into the data log when the state begins so downstream analysis can slice trials by phase. Builders seed it 1-based, so
  Mesoscope-VR uses `state_index + 1`, and that seeding matters. On Mesoscope-VR, code `0` is reserved for the implicit
  `idle` state that `sollertia-forgery` injects into the code-to-name mapping. A state numbered `0` is therefore
  overwritten by `idle` and never appears under its own name in the forged `runtime_state` column. The upper bound on
  that system is `255`, because the acquisition runtime serializes the code as an unsigned 8-bit value. Use a
  human-readable phase name only for the `experiment_states` dict key, never for this field, because a string written
  here loads silently, since per-field type checking is disabled, and reaches the runtime as a wrong type.
- **`system_state_code`** is the hardware-mode snapshot that the acquisition runtime should install for the duration of
  the state. The valid codes are system-specific, defined by the target system's system-state enum in
  `sollertia-experiment`. Consult that system's schema skill for the accepted values (for Mesoscope-VR, see
  `mesoscope:mesoscope-vr-experiment-schema`).

The two codes are deliberately decoupled, so a "ramp" phase and a "performance" phase reuse one active hardware mode
while differing in duration or guidance settings. Collapsing the pair would force a hardware reconfiguration on every
phase boundary, which is why both fields exist on the schema.

`ExperimentState.supports_trials` (default `True`) determines whether trials are executed during this experiment state.
It is a declarative annotation, and no production code in `sollertia-experiment` or `sollertia-forgery` reads it. A
trial-free phase is realized by choosing a `system_state_code` whose hardware mode drives no trials, and setting
`supports_trials` to `False` alongside it documents that intent for a human reader.

### The source of the trial sequence

The experiment configuration **never enumerates or schedules trials**, and contributes only the **per-trial-type
parameters**, meaning the system-specific runtime fields on each trial class. Unity (`sollertia-virtual-reality`) owns
the trial sequence. At session init the acquisition runtime requests a cue sequence materialized from the template's
per-trial `transitions` (see `/task-templates`), then identifies trial boundaries by motif matching against each
`TrialStructure`. It joins each decomposed trial name back to this configuration's `trial_structures` to attach the
per-trial parameters. Relative frequencies are encoded in the template's transition probabilities.

### Guidance is per-state, not per-trial-class

Each `ExperimentState` carries reinforcing and aversive guidance counters (`*_initial_guided_trials`,
`*_recovery_failed_threshold`, `*_recovery_guided_trials`). At state entry the runtime configures the counters from
these fields and zeros any prior failure streak. During the state, completed trials count down the guided-trials budget,
and sustained failure streaks re-engage guidance for a recovery window.

The counters live on `ExperimentState` rather than on the trial classes because guidance is a **phase-level** decision.
One configuration captures a whole training arc by chaining a "ramp" state carrying heavy initial guidance and a low
recovery threshold. A "performance" state then carries modest recovery support alone, and a "no help" state carries zero
on both counters, all three reusing the same trial classes and the same template.

### The experiment configuration contract

Every `<System>ExperimentConfiguration` shares one **stable contract**, holding `experiment_states`, `trial_structures`,
`unity_scene_name`, and the `from_task_template` builder. Everything else is **system-specific**: the concrete class
adds whatever further fields its acquisition system needs and defines its own runtime trial classes, so the contract is
the floor rather than the whole schema. Always read the actual field set from
`describe_experiment_configuration_schema_tool` for the system you target, because its `nested_classes` are introspected
from the resolved class and reflect that system's own dataclasses. The bullets below name the contract members only.

- **`experiment_states: dict[str, ExperimentState]`** is the experiment state machine, required on every subclass
  because every experiment is a state machine.
- **`trial_structures`** holds the trials the experiment runs, required on every subclass as a per-trial dict whose
  values are the **system's own** runtime trial classes. The task template provides each trial's spatial
  `TrialStructure`, the configuration pairs it with a runtime trial class by trial name, and
  `list_supported_trial_types_tool` derives the trial list from this field. The platform `trigger_type` taxonomy has
  **five** modes (`interaction`, `collision`, `occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`). Each
  acquisition system maps only the **subset** it supports through its `from_task_template` builder, pairing every
  supported mode with one of its own runtime trial classes that matches the zone prefab Unity instantiates. An unmapped
  mode raises a clear "not mapped to a runtime trial class" error, and the system's schema skill names the mapped set. A
  hand-authored trial entry must also carry whatever discriminator field the target system's trial union requires, so
  the loader can pick the right runtime trial class (for Mesoscope-VR, `trial_kind`).
- **`unity_scene_name`** is a mandatory contract field. It identifies the paired `TaskTemplate` by filename stem and is
  verified against the scene loaded in Unity at session start. `SessionData.create` resolves
  `<templates-directory>/<unity_scene_name>.yaml` to cache the session's VR snapshot, so the value must equal the
  template filename stem exactly. Nothing validates the match when the configuration is built, so a stem typo surfaces
  only at session creation.

---

## MCP tool surface

| Tool                                                  | Purpose                                                                                                            |
|-------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `discover_experiments_tool`                           | Lists experiment configurations under a project                                                                    |
| `describe_experiment_configuration_schema_tool`       | Returns the field schema for the experiment dataclass (exclusive)                                                  |
| `read_experiment_configuration_tool`                  | Reads an experiment configuration YAML from any canonical location (project source or per-session frozen snapshot) |
| `write_experiment_configuration_tool`                 | Writes a validated experiment configuration payload for any system, author or repair (exclusive)                   |
| `delete_experiment_configuration_tool`                | Removes one experiment configuration YAML at an explicit path (exclusive)                                          |
| `create_experiment_from_vr_template_tool`             | Creates an experiment configuration from a Unity VR task template (exclusive)                                      |
| `validate_experiment_configuration_tool`              | Validates an experiment configuration YAML (exclusive)                                                             |
| `list_supported_acquisition_systems_tool`             | Enumerates `AcquisitionSystems`                                                                                    |
| `list_supported_trial_types_tool(acquisition_system)` | Lists the runtime trial classes the resolved system declares, each with its full field schema                      |

`delete_experiment_configuration_tool` removes one configuration YAML at the `file_path` it is given, confined to a
`.yaml` file whose parent directory is named `configuration` and which resolves under the data root the call targets.
That root is the configured platform data root unless the optional `root_directory` argument names another one, such as
an archival mirror on mounted server storage, and the response echoes the resolved root back as `root_directory`. The
confinement is what keeps the per-session frozen snapshot at `<session>/raw_data/experiment_configuration.yaml` out of
reach. Every session already acquired keeps its own frozen snapshot and stays readable, while a new session naming the
removed experiment can no longer be created, because `SessionData.create` copies the project configuration into each
experiment session.

`list_supported_trial_types_tool` is experiment-configuration introspection rather than template introspection, so it
belongs to this skill. It resolves the configuration class through `EXPERIMENT_CONFIGURATION_REGISTRY` and derives its
entries from that class's `trial_structures` annotation, returning one `class_name` plus a full field `schema` per
runtime trial class. Its `acquisition_system` argument is **required**, so a bare `list_supported_trial_types_tool()`
call is invalid. `list_supported_acquisition_systems_tool` takes no arguments and enumerates the `AcquisitionSystems`
members as `value` / `name` pairs, which is the canonical way to learn what to pass as `acquisition_system`.

---

## Path conventions

All read, write, validate, and create tools in this skill take **explicit file paths** plus a required
**`acquisition_system`**, because the experiment-config dataclass is selected per system through the registry. The
caller resolves both, and the tools never consult `root_directory`, a project name, an experiment name, or a template
name. The canonical paths are:

| Asset                                         | Canonical path                                                    |
|-----------------------------------------------|-------------------------------------------------------------------|
| Per-project experiment configuration          | `<root>/<project>/configuration/<experiment>.yaml`                |
| Task template                                 | `<templates-directory>/<template-name>.yaml`                      |
| Per-session frozen experiment-config snapshot | `<session>/raw_data/experiment_configuration.yaml`                |
| Forged dataset per-session snapshot           | `<dataset_root>/<animal>/<session>/experiment_configuration.yaml` |

Use `discover_experiments_tool(root_directory=..., project=...)` to enumerate existing configs and their absolute paths,
and use `discover_templates_tool()` to enumerate template paths. The forged copy is discovered through
`inspect_datasets_tool` (`/datasets`), which returns its absolute path for every session the dataset claims. That copy
is conditional, because the dataset marker does not carry the per-session experiment, so read its `present` flag off the
same report before passing the path to any tool.

**Resolving `acquisition_system`.** It is required and carries no default. Resolve it **automatically** wherever
possible and prompt the user only as a fallback. For a **per-session snapshot**, read it from the session's own
`SessionData`, where `inspect_sessions_tool` (`/session-data`) reports `identity.acquisition_system`, or read the marker
directly via `read_session_data_tool`. For a **forged dataset copy**, no sibling `session_data.yaml` is reachable, so
read the value from the dataset marker instead, which `/datasets` surfaces as the `acquisition_system` field of
`dataset.yaml`. For **per-project authoring**, use the system the project or host targets, and enumerate the options
with `list_supported_acquisition_systems_tool`. Pass the resolved value to every tool below.

---

## Authoring an experiment configuration

### Step 1: Verify prerequisites

- MCP server connected (else `/assets-mcp-environment-setup`).
- The target project directory does **not** have to exist before authoring. Both `write_experiment_configuration_tool`
  and `create_experiment_from_vr_template_tool` create any missing parent directories, including the project and its
  `configuration` subdirectory, so an absent project never blocks the write. Mint the project explicitly anyway, by
  handing off to `/project-hierarchy`, which owns `create_project_tool` and the `slsa configure project -p <name>` CLI
  command (the CLI takes the project name alone and uses the configured platform data root). The reason is downstream:
  `SessionData.create` raises `FileNotFoundError` when the project is missing, so a session cannot be created against a
  tree that only the configuration write brought into existence.
- The target task template exists at a known path. If it does not, hand off to `/task-templates` to author it, because
  this skill must not call `write_template_tool` directly. The templates directory can be enumerated via
  `discover_templates_tool`, which also returns absolute paths.

### Step 2: Discover existing experiments under the project

```text
discover_experiments_tool(
    root_directory="<absolute path to data root>",
    project="<project>",
)
```

`discover_experiments_tool` returns every experiment's absolute `path`, which is what you pass to the read, write, and
validate tools below. If a similar experiment already exists, prefer reading it and modifying a copy.

`project` is optional. Omitting it enumerates every project under the root, silently skipping any project directory that
has no `configuration` subdirectory. `root_directory` is **required** and has no data-root fallback, unlike
`create_project_tool`, so the caller must resolve the root before calling. Discovery globs `*.yaml` only, so a
configuration saved with a `.yml` extension never appears in the listing even though `from_yaml` would load it.

### Step 3: Inspect the experiment configuration schema

```text
describe_experiment_configuration_schema_tool(acquisition_system="<system>")
```

Use the schema as the source of truth for field names and nesting. It always reports the contract fields
(`experiment_states`, `trial_structures`, and `unity_scene_name`) plus whatever system-specific fields the resolved
subclass adds. Its `nested_classes` are derived from the resolved configuration class, so they reflect that system's
actual nested dataclasses. Never assume the Mesoscope-VR set applies verbatim.

### Step 4: Create the configuration

Seed the configuration from a Unity VR task template with `create_experiment_from_vr_template_tool`. The tool dispatches
through `EXPERIMENT_CONFIGURATION_REGISTRY` (keyed by `AcquisitionSystems`) to the resolved
`<System>ExperimentConfiguration` and calls its `from_task_template` builder. To author or repair a full payload
directly, without re-seeding from the template, use `write_experiment_configuration_tool` (see Step 5).

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

This loads the template via `TaskTemplate.from_yaml`, then calls the resolved config class's `from_task_template`
classmethod. That classmethod maps the template's trial structures to runtime trials and seeds `state_count`
default-valued runtime states in the `experiment_states` dict.

**Heads up:** the autopopulated states have a placeholder `system_state_code` that is typically not a valid hardware
mode for the target system. Each state must be edited to set `system_state_code` to a value the system accepts before
the session can run. Consult the system's schema skill for the accepted codes (for Mesoscope-VR, see
`mesoscope:mesoscope-vr-experiment-schema`).

How a builder seeds the guidance counters is **system-specific**, and `ExperimentState` itself defaults every counter to
`0`. Mesoscope-VR seeds the reinforcing block only when the template produced a water trial and the aversive block only
when it produced a puff trial (see `mesoscope:mesoscope-vr-experiment-schema`).

### Step 5: Customize state machine and trial parameters

Read the just-created configuration:

```text
read_experiment_configuration_tool(
    file_path="<root>/<project>/configuration/<experiment>.yaml",
    acquisition_system="<system>",
)
```

Mutate the fields the user wants to customize, using the contract members described under "The experiment configuration
contract" above.

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

`validate_experiment_configuration_tool` loads the YAML through the resolved experiment-config subclass's `from_yaml`
and re-runs whatever `__post_init__` that subclass declares.

Field types are **not** checked, so a wrong-typed value loads exactly as written. What the loader enforces is
required-field presence plus the resolved subclass's own `__post_init__`, which is system-specific and can reject a
payload on semantic grounds (for Mesoscope-VR, every trial must resolve to a runtime trial class, see
`mesoscope:mesoscope-vr-experiment-schema`). For what the write path checks before it persists, read the
`## Response contract` section of `/assets-mcp-environment-setup`.

The experiment configuration does not carry the corridor task's spatial data, so cross-template validation (cue
sequences, zone bounds, trigger-type pairing) is the responsibility of `/task-templates` and its
`validate_template_tool` on the paired template. At session init the acquisition runtime joins the two by trial name,
validating that every trial name the cue-sequence decomposer produces from the template has a matching entry in the
configuration's `trial_structures`. That check runs in one direction only, so a `trial_structures` key with no template
counterpart survives acquisition and then fails the session's processing. The `sollertia-forgery` Mesoscope-VR runtime
parser raises a `ValueError` for it after the data is already on disk. Key order is load-bearing in a narrower way. The
join itself is by name, so the template's own trial ordering is irrelevant. A name's position in `trial_structures` is
the canonical trial index the parser writes into the runtime feathers, and the dataset assembly resolves those indices
back to names through the same enumeration. Reordering or inserting into the `trial_structures` of a snapshot already
parsed therefore silently relabels every trial in the resulting dataset. Keep `trial_structures` and the template's
trial set in one-to-one correspondence by name, and leave the key order of a frozen snapshot alone.

The tool reports three distinct outcomes:

- `valid=True` inside a success envelope, carrying a `summary` with `trial_count`, `state_count`, and
  `unity_scene_name`.
- `valid=False` inside a **success** envelope, carrying a single-element `issues` list with the load or `__post_init__`
  failure. The envelope is still `success: true`, so branching on `success` alone reports a broken configuration as a
  passing one. Always read `valid`.
- `success: false` with an `error` message when the file does not exist or the `acquisition_system` does not resolve. A
  missing file is **not** reported as `valid=False`.

Fix any reported issues and re-write. The `## Response contract` section of `/assets-mcp-environment-setup` documents
this envelope split for every tool that follows it.

---

## Reading the frozen configuration from a session

After a session has run, the experiment configuration that was active at session start is captured as a frozen YAML at
`<session>/raw_data/experiment_configuration.yaml`. To read it:

```text
read_experiment_configuration_tool(
    file_path="<session>/raw_data/experiment_configuration.yaml",
    acquisition_system="<system>",  # read from the session's own SessionData (identity.acquisition_system)
)
```

The frozen copy is conventionally immutable once written, and downstream pipelines expect it that way. Amendment is
still possible, because `write_experiment_configuration_tool` writes exactly the one file its `file_path` names, and
that path may be the frozen snapshot. Confirm the planned write with the user, pass `overwrite=True` explicitly against
the tool's `overwrite=False` default, and verify the result by re-reading the snapshot and diffing it field by field
against the payload you intended. Renaming an `experiment_states` key or reordering `trial_structures` is the riskiest
amendment of all, because the forged dataset's `runtime_state` and `trial_type` labels resolve through those keys. An
amendment landing between two processing stages relabels the session's data rather than correcting it. The write is
local to the path passed and never reaches the project source configuration at
`<root>/<project>/configuration/<experiment>.yaml`, so a correction that must apply to both is written to each file.

---

## Common patterns

The goal-to-pattern table for the recurring edits lives in
[references/authoring-patterns.md](references/authoring-patterns.md), which also holds the walkthrough for moving an
experiment to a new template. Those edits are reusing a template across projects, changing a per-trial parameter,
adjusting or adding a state, and adding a spatial trial entry.

---

## Troubleshooting

### Resolving `acquisition_system`

`read_experiment_configuration_tool`, `write_experiment_configuration_tool`,
`describe_experiment_configuration_schema_tool`, `validate_experiment_configuration_tool`, and
`list_supported_trial_types_tool` all route `acquisition_system` through the same private resolver, which fails in two
distinct ways:

| Failure                                          | Where to go                               |
|--------------------------------------------------|-------------------------------------------|
| The value is not an `AcquisitionSystems` member  | `list_supported_acquisition_systems_tool` |
| The member has no registered configuration class | `/library-extension`                      |

The first failure names the offending value and lists the valid enum values. It is a typo or a system that does not
exist on this platform, so re-read the enum with `list_supported_acquisition_systems_tool` and pass one of the reported
`value` entries verbatim. The second failure names the value and lists the registered systems. It means the enum member
exists but `EXPERIMENT_CONFIGURATION_REGISTRY` holds no class for it, which is a library gap rather than a caller
mistake, so hand off to `/library-extension` to register the subclass.

`create_experiment_from_vr_template_tool` does not use that resolver. It validates the enum itself and subscripts the
registry directly, so an invalid value comes back worded as a create failure rather than a resolve failure, and that
path has no separate "no class is registered" message.

---

## Related skills

| Skill                                      | Relationship                                                                                                                                                                                                                                                                      |
|--------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/cli-reference`                           | Owns the `slsa configure project` command that seeds a configuration directory                                                                                                                                                                                                    |
| `/working-directory`                       | Provides the templates directory so `/task-templates` knows where to enumerate. This skill needs only absolute paths                                                                                                                                                              |
| `/assets-mcp-environment-setup`            | Run first if the MCP server is not connected                                                                                                                                                                                                                                      |
| `/task-templates`                          | Required dependency that owns corridor template authoring and exposes `discover_templates_tool` for template paths                                                                                                                                                                |
| `/project-hierarchy`                       | Discovers the project tree and owns project creation (`create_project_tool` and `slsa configure project`)                                                                                                                                                                         |
| `/datasets`                                | Owns the forged-dataset container and resolves the forged `experiment_configuration.yaml` copy's path via `inspect_datasets_tool`                                                                                                                                                 |
| `/session-data`                            | Owns the `SessionData` marker from which this skill reads `acquisition_system` for a per-session snapshot                                                                                                                                                                         |
| `mesoscope:mesoscope-vr-experiment-schema` | Owns Mesoscope-VR's concrete instance, covering the trial-class and experiment field schema, the trigger-type-to-trial mapping, the system-state codes, and the `from_task_template` defaults to which this skill defers                                                          |
| `experiment:acquisition-system-design`     | Documents the per-system system-configuration pattern (Mesoscope-VR instance: `MesoscopeSystemConfiguration`)                                                                                                                                                                     |
| `experiment:acquisition-system-runtime`    | Downstream consumer whose `SessionData.create` copies the authored `experiment_configuration.yaml` into every new experiment session at acquisition time                                                                                                                          |
| `unity:task-prefabs`                       | Validates template values against the Unity prefab state                                                                                                                                                                                                                          |
| `experiment:pipeline`                      | Phase 4 of the experiment lifecycle (experiment authoring) hands off to this skill                                                                                                                                                                                                |
| `/library-extension`                       | Cross-cutting recipe to add a new `AcquisitionSystems`, runtime trial class, or `TriggerType` member. A new `TriggerType` member does **not** require a `from_task_template` branch, because a system may leave it unmapped. Lists the prose here that needs updating in lockstep |
| `experiment:vr-driver-interface`           | Verifies `unity_scene_name` against the live scene and consumes the per-trial parameters at runtime                                                                                                                                                                               |

---

## Verification checklist

```text
Tool-settled (run write, validate, and read against the target path):
- [ ] sollertia-shared-assets MCP server is connected
- [ ] write_experiment_configuration_tool succeeded without schema errors
- [ ] validate_experiment_configuration_tool returned valid=True with no issues
- [ ] read_experiment_configuration_tool returned the expected configuration after the write

Reader-judged:
- [ ] Target project was minted via /project-hierarchy (the write tools create parents, SessionData.create does not)
- [ ] Target template exists (handed off to /task-templates if missing) and template_path is known
- [ ] describe_experiment_configuration_schema_tool was used as the source of truth for field names
- [ ] file_path was passed to every read/write/validate/create call (absolute path)
- [ ] Payload was passed as configuration_payload (the correct kwarg name)
- [ ] experiment_states was treated as a dict (string keys), not a list (integer indices)
- [ ] Field names were taken from describe_experiment_configuration_schema_tool (no invented fields)
- [ ] Each state_duration_s and per-trial parameter was compared against the project's existing configurations
- [ ] Every value outside the range those configurations use was confirmed with the user before the write
- [ ] Did not call write_template_tool from this skill
```
