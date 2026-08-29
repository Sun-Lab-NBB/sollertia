---
name: task-templates
description: >-
  Authors, modifies, and validates reusable TaskTemplate YAMLs, the Virtual-Reality task asset against which every
  corridor-task session is acquired (VR environment, cue catalog, trial structures with per-trial cue sequences and
  zones), via the sollertia-shared-assets MCP server. Owns write_template_tool, validate_template_tool, and
  describe_template_schema_tool. Use when designing or modifying a task template for a VR experiment.
user-invocable: false
---

# Sollertia task templates

Authors and modifies reusable `TaskTemplate` YAML files for `sollertia-shared-assets` using the `slsa mcp` MCP server.
This skill is the **exclusive** owner of the template write and validation surface, meaning `write_template_tool`,
`describe_template_schema_tool`, and `validate_template_tool`. No other skill in the marketplace may call these. This
skill also owns the trigger-type vocabulary as a template uses it, together with the authoring guidance for
`list_supported_trigger_types_tool`, while `/library-extension` calls that same helper read-only as its
extension-verification check. `list_supported_trial_types_tool` belongs to `/experiment-configuration`, and this skill
only reads it to learn which runtime trial classes an experiment configuration can pair with a template's trial
structures.

---

## Scope

**Covers:**
- Authoring `TaskTemplate` YAML files in the configured task templates directory
- The template vocabulary (`Cue`, `TrialStructure`, `VREnvironment`, `TriggerType`)
- Discovering existing templates and reading their contents
- Schema introspection via `describe_template_schema_tool`
- Template validation via `validate_template_tool`
- Reading the acquisition system's runtime trial classes via `list_supported_trial_types_tool`, which
  `/experiment-configuration` owns
- Enumerating supported trigger types (`list_supported_trigger_types_tool`)

**Does not cover:**
- Per-project experiment configuration authoring (see `/experiment-configuration`, and for the Mesoscope-VR concrete
  schema, `mesoscope:mesoscope-vr-experiment-schema`)
- Setting the task templates directory path (see `/working-directory`)
- Generating a Unity task from a template (see `unity:task-prefabs`, whose `create_task_tool` builds both the task
  prefab and the matching scene in one call)

---

## What is a task template

A `TaskTemplate` is the Virtual-Reality task asset against which every corridor-task session is acquired: a reusable
description of a VR behavioral paradigm, holding the VR environment, the cue catalog, and the trial structures. Each
trial owns its own cue sequence, zone geometry, and trigger type, and is materialized into a single segment prefab named
`<TemplateName>-<TrialName>.prefab` at generation time. A session whose type is listed in `SESSION_TYPES_USING_VR_TASK`
is required to carry the template snapshot, and other session types carry none. Every experiment configuration is built
from a corridor task template. A template is project- and system-agnostic, so the same template can back many experiment
configurations across many projects. Mesoscope-VR is the only acquisition system registered today, so
`MesoscopeExperimentConfiguration` is currently the only concrete experiment configuration, while the
`AcquisitionSystems` enum and the registry dispatch behind it are designed for additional acquisition systems.

### Known file locations

A `TaskTemplate` YAML exists in three canonical locations, all parsed by the same `TaskTemplate` dataclass:

| Location                                                  | Populated by                               | Discovery path                                                            |
|-----------------------------------------------------------|--------------------------------------------|---------------------------------------------------------------------------|
| `<templates-directory>/<template-name>.yaml`              | This skill, via `write_template_tool`      | `discover_templates_tool`, with `/working-directory` owning the directory |
| `<session>/raw_data/vr_configuration.yaml`                | `SessionData.create()` at session creation | `inspect_sessions_tool` (`/session-data`), as `vr_configuration_path`     |
| `<dataset_root>/<animal>/<session>/vr_configuration.yaml` | Forging pipeline at dataset assembly       | `/datasets`, whose `inspect_datasets_tool` returns the absolute path      |

- The **live template** is the authoring surface owned by this skill, shared across projects and sessions, and the
  source of truth from which Unity generation reads. Editing here is intentional and affects every future session that
  picks the template.
- The **per-session frozen snapshot** is an immutable copy that `SessionData.create()` caches into the session's
  `raw_data` directory when the session names an experiment **and** its session type is listed in
  `SESSION_TYPES_USING_VR_TASK`. The snapshot is keyed off the experiment configuration's `unity_scene_name` and records
  the exact template against which the session was acquired. Downstream processing, covering forgery and analysis, joins
  the snapshot to behavioral data, so it must not drift after the session is created.
- The **forged dataset copy** is written next to the session's `data.feather` when the dataset is assembled. It is a
  required dataset artifact only when the dataset's session type is listed in `SESSION_TYPES_USING_VR_TASK`. `/datasets`
  audits its presence and routes questions about its contents here.

The same MCP tools serve all three. Pass the live path to author or modify a template, and pass a snapshot path to read
or validate a frozen copy. `write_template_tool` targets the live surface only. The frozen copies are produced by
`SessionData.create()` and by the forging pipeline, and are not callers' to overwrite.

A template defines **what is possible** for a VR experiment. The paired experiment configuration picks a template by
`unity_scene_name` and parameterizes it with state durations, per-trial reward volumes and puff durations, and
project-specific overrides. The two are authored by two different skills with two different ownership scopes.

### Where the template name flows

The template basename anchors a three-tier Unity artifact chain, running from template to task prefab to scene, with all
three sharing the same name by convention:

```text
Configurations/<name>.yaml          template (authored here)
        │
        ▼  unity:task-prefabs (create_task_tool)
InfiniteCorridorTask/Tasks/<name>.prefab    task prefab
        │
        ▼  unity:task-prefabs (same create_task_tool call)
Scenes/<name>.unity                 scene (instantiates the task prefab)
```

The canonical reference for this hierarchy is `unity:task-prefabs`. When you rename a template, the regenerated task
prefab and the next scene created from it inherit the new name, and the artifacts generated under the old name remain on
disk. Removing them is owned by `unity:task-prefabs`, which handles it end to end through `delete_task_tool`.

---

## Template vocabulary

A `TaskTemplate` is composed of these classes (all defined in `sollertia_shared_assets.configuration`):

| Primitive        | Purpose                                                                                                                         |
|------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `Cue`            | A visual cue (name, uint8 code, length, required texture) referenced by trial cue sequences                                     |
| `TrialStructure` | Per-trial spatial config: cue sequence, optional transitions, stimulus trigger zone, stimulus location, trigger type            |
| `VREnvironment`  | VR corridor configuration: spacing, segments per corridor, padding prefab, units, cue offset                                    |
| `TriggerType`    | Enum of stimulus trigger zone activators (`interaction`, `collision`, `occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`) |

`VREnvironment` is the one primitive whose every field carries a default:

| Field                   | Default     |
|-------------------------|-------------|
| `corridor_spacing_cm`   | `20.0`      |
| `segments_per_corridor` | `3`         |
| `padding_prefab_name`   | `"Padding"` |
| `cm_per_unity_unit`     | `10.0`      |
| `cue_offset_cm`         | `0.0`       |

Each default matches the one Unity's own `VREnvironment` class declares for the same field, so a key omitted inside a
present `vr_environment` block loads with the geometry Unity would apply to it. The block itself is required, because
`TaskTemplate.vr_environment` carries no default, so `from_yaml` raises dacite's `MissingValueError`, and Unity's
`ConfigLoader.ValidateTemplate` throws `No VR environment configuration defined.`

`TaskTemplate.trial_structures` is a `dict[str, TrialStructure]` keyed by trial name. The template owns how often each
trial runs, through each trial's `TrialStructure.transitions` probability dict, and the experiment configuration owns
the per-trial stimulus parameters and the choice of runtime trial class. No weight field exists on either side. The
experiment configuration is the system-specific class owned by `/experiment-configuration`, which on Mesoscope-VR is
`MesoscopeExperimentConfiguration`. The system's runtime trial classes (on Mesoscope-VR, `MesoscopeWaterRewardTrial` and
`MesoscopeGasPuffTrial`) are experiment-scope classes, and they appear in this skill only when you call
`list_supported_trial_types_tool` to enumerate what an experiment configuration may instantiate to pair with a
template's `TrialStructure` entries.

For canonical field definitions and valid values, call `describe_template_schema_tool` rather than relying on
handwritten documentation that may drift from the slsa source of truth. For the canonical trigger-type enum values, use
`list_supported_trigger_types_tool`, and for the runtime trial classes of an acquisition system, use
`list_supported_trial_types_tool`.

---

## How a template drives Unity and the acquisition runtime

A template has two consumers. **Unity** generates the VR environment from it, and the **acquisition runtime** decomposes
the resulting cue sequence back into a trial timeline. The schema is shaped by what those two consumers need. Unity-side
prefab generation, scene loading, and runtime corridor mechanics are owned by the `unity:task-prefabs` and
`unity:task-scenes` skills. Trial decomposition and runtime trial state are owned by the experiment library and
documented at the conceptual level in `/experiment-configuration`. The overview below is the slsa-side conceptual model,
so defer to those skills for implementation specifics.

### The VR paradigm: the infinite corridor

The VR task paradigm is the **infinite linear corridor**, and the `TaskTemplate` is the corridor task template. The
animal is head-fixed on a treadmill and runs forward through a one-dimensional corridor whose wall geometry is built
from segment prefabs. The corridor has no end: the entire cue sequence the animal will see is **pre-resolved at session
init** from the template, and the animal then traverses a deterministic chain of segments without branching, finish
lines, or on-the-fly geometry generation.

Four levels describe the corridor's composition, finest to coarsest: cue, segment, corridor, task.

***Note,*** a **segment** is the Unity prefab that materializes a **trial**, so the two terms refer to the same unit of
behavior at different layers. "Trial" is how the template describes it abstractly, and every entry under
`trial_structures` is a trial. "Segment" is how Unity names the prefab that the runtime instantiates and that the animal
traverses.

```text
Task
  │
  ├── Corridor          ─┐  A corridor is a fixed sequence of segments stacked along the animal's
  ├── Corridor           │  forward axis. A task pre-instantiates one corridor for every possible
  ├── Corridor           │  combination of segments, so corridors enumerate the configuration
  └── … (every          ─┘  space the corridor sequence can take.
       combination)

      one Corridor
        ├── Segment          ← active: the trial currently driving behavior
        ├── Segment          ← lookahead: visible only, no behavior
        └── Segment          ← lookahead: visible only, no behavior

      one Segment (= one trial)
        ├── Cue
        ├── Cue              ← the cue sequence declared by this trial, laid out along the corridor
        └── …
```

- **Cues** are individual visual panels displayed along the walls of the corridor. They are the smallest unit, and are
  shared across every trial, and every template, that declares the same cue identity, meaning the cue `name` paired with
  its `length_cm`.
- **Segments are trials.** Each entry in the template's `trial_structures` dict produces exactly one segment. A segment
  owns its cue sequence and the behavioral element associated with that trial (the stimulus trigger zone, the reward or
  aversive contingency, etc.).
- **Corridors** are fixed-length windows of segments. The first segment in a corridor is the **active** trial, and it
  drives behavior. The remaining segments are pure visual lookahead, so the animal sees what is coming without yet
  experiencing it. The window length is set by `vr_environment.segments_per_corridor`, and setting it to one collapses
  corridor and segment into the same thing.
- **Tasks** are the full set of corridors plus the transition graph that describes how trials chain during a session.
  Each `TrialStructure` may declare a probability distribution over the trials that can follow it via `transitions`.
  Trials without an explicit distribution are followed by a uniformly random trial.

**Iterative corridor traversal.** At session start, the runtime walks the trial-transition graph to build a flat
sequence of trials that overshoots the configured track length. It then slides a window the size of the corridor (in
segments) over that sequence. The current window identifies one corridor from the pre-built catalog, and the animal is
teleported to that corridor's start. Whenever the animal finishes the first segment of the current corridor, the window
slides one trial forward and the animal jumps to the corridor whose segment combination matches the new window. Adjacent
corridors share `segments_per_corridor - 1` segments, so the visible cue sequence stays continuous across teleports.

The corridor count grows as `trial_count ^ segments_per_corridor`, so raising the lookahead depth is a deliberate choice
that adds visual context at the cost of an exponentially larger task. Most paradigms use one or two segments per
corridor. Every concept in this skill (cues, segments, transition probabilities, trial structures, `cue_offset_cm`,
`segments_per_corridor`) is built around this hierarchy.

**Autonomy boundary.** The infinite corridor is the **only** supported VR paradigm, and the only one with an
author-derived recipe. You can author and validate new corridor templates autonomously. Extending *beyond* the corridor
has no recipe, whether that is a new topology such as a T-maze, an open field, or branching, a new `VREnvironment`
class, or new runtime mechanics. Escalate those to the human supervisor and co-design them in a generative,
collaborative mode. What is missing there is a deterministic recipe rather than capability, so the work must be
human-supervised.

### Deeper field reference

The per-field consumer roles, the firing rule of each of the five `TriggerType` modes, and the rationale behind the
template's shape live in [references/field-semantics.md](references/field-semantics.md). Read it when authoring a new
trial structure, picking a trigger mode, or deciding on which side of the template / experiment split a parameter sits.

---

## MCP tool surface

| Tool                                | Purpose                                                                                                                                 |
|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| `discover_templates_tool`           | Lists all task templates in the configured templates directory                                                                          |
| `read_template_tool`                | Reads an existing template at an explicit path                                                                                          |
| `write_template_tool`               | Writes a new template or overwrites an existing one (exclusive to this skill)                                                           |
| `describe_template_schema_tool`     | Returns the field schema for `TaskTemplate` (exclusive to this skill)                                                                   |
| `validate_template_tool`            | Validates a template against its schema and cross-reference constraints                                                                 |
| `list_supported_trial_types_tool`   | Enumerates the runtime trial classes the named `acquisition_system`'s experiment configuration declares (requires `acquisition_system`) |
| `list_supported_trigger_types_tool` | Enumerates the `TriggerType` enum values                                                                                                |

`list_supported_trial_types_tool` is a reference here rather than an owned tool. `/experiment-configuration` owns it and
the trial-class vocabulary it returns, so hand off there when a caller needs more than the class names.

`list_supported_trigger_types_tool` is read-only on both sides that call it. This skill owns the trigger-type vocabulary
as a template uses it, and `/library-extension` calls the same helper to confirm that a newly added `TriggerType` member
reached the tooling.

`read_template_tool`, `write_template_tool`, and `validate_template_tool` take an explicit `file_path`, and path
resolution is the caller's responsibility. The canonical home for **live** templates is the directory set via
`/working-directory`'s `set_task_templates_directory_tool`. `discover_templates_tool()` returns each live template's
absolute path, which is what you pass to `read_template_tool`, `write_template_tool`, and `validate_template_tool` when
authoring.

To inspect a per-session **frozen snapshot**, pass the session's `<session>/raw_data/vr_configuration.yaml` path to
`read_template_tool` or `validate_template_tool`. `write_template_tool` is for the live surface only. Snapshots are
produced by `SessionData.create()` and must not be overwritten through this tool.

---

## Authoring workflow

### Step 1: Verify prerequisites

- The `sollertia-shared-assets` MCP server is connected (else hand off to `/assets-mcp-environment-setup`).
- The task templates directory is set (else hand off to `/working-directory`).

### Step 2: Discover existing templates

```text
discover_templates_tool()
```

If a similar template already exists, prefer reading it with `read_template_tool(file_path=...)` and modifying a copy.
Avoid creating near-duplicates. The response includes each template's absolute `path`, so capture it to pass into
`read_template_tool`, `write_template_tool`, and `validate_template_tool`.

Discovery lists top-level `*.yaml` files only, so a template saved with a `.yml` suffix never appears even though the
loader accepts it. A template that fails to load is still listed, carrying an `error` field in place of `cue_count`,
`trial_count`, and `cue_offset_cm`, and `total_templates` counts it, so a non-zero count is not proof that every
template loads. The success payload keys are `templates`, `total_templates`, and `templates_directory`.

### Step 3: Inspect the schema and enums

```text
describe_template_schema_tool()
list_supported_trial_types_tool(acquisition_system="<acquisition-system>")
list_supported_trigger_types_tool()
```

Use the schema and the tool responses as the source of truth, because they pin down the exact strings `trigger_type`
accepts and the exact class names of the runtime trial variants the target acquisition system supports. This avoids
silent typos that slip past YAML syntax but fail at runtime. `list_supported_trial_types_tool` requires an
`acquisition_system` argument, and the valid values for it come from `list_supported_acquisition_systems_tool`.
Mesoscope-VR is the only registered system today. The tool returns a `class_name` and a full field schema for each trial
class, and `/experiment-configuration` owns that surface.

### Step 4: Author the template

Build the template dictionary in this order:

1. **VR environment.** Define the `VREnvironment`, covering corridor spacing, segments per corridor, padding prefab
   name, cm-per-unity-unit conversion, and cue offset. Every field carries a default, so individual keys may be omitted,
   but the `vr_environment` block itself is required on both the Python and the Unity side.
2. **Cue catalog.** Define every `Cue`, covering name, uint8 code, length, and required texture filename. A cue name may
   hold only ASCII letters, digits, and underscores, because the name is embedded in the generated
   `Cue_<name>_<length>cm` asset filename and in the cue-sequence signature that identifies a trial.
3. **Trial structures.** Populate the `trial_structures` dict, mapping each trial name to a `TrialStructure`. That
   covers the cue sequence, the stimulus trigger zone start and end, the stimulus location, whether the collision
   boundary is visible, and the trigger type from `list_supported_trigger_types_tool`. It also covers an optional
   `transitions` dict that maps target trial names to probabilities summing to 1.0. Set `occupancy_duration_ms` on every
   occupancy-mode trial and leave it `None` on every other trial. `None` is how a template says the field is unused,
   while `0` is a real duration and is rejected on every trial whatever its trigger type.

Reward sizes, gas-puff durations, experiment states, and the choice of trial class (on Mesoscope-VR,
`MesoscopeWaterRewardTrial` versus `MesoscopeGasPuffTrial`) are **not** part of the template, and they are added
per-experiment by `/experiment-configuration`. How often each trial runs is the template's own concern and lives in that
trial's `transitions` dict. Occupancy dwell time is likewise the template's, carried by the
`TrialStructure.occupancy_duration_ms` field shared by all three occupancy modes.

### Step 5: Write, validate, and re-read

Use `read_task_templates_directory_tool` (from `/working-directory`) or `discover_templates_tool` to learn the canonical
templates directory, then construct the destination file path as `<templates-directory>/<template-name>.yaml`:

```text
write_template_tool(
    file_path="<templates-directory>/<template-name>.yaml",
    template_payload={ ... full nested dict ... },
    overwrite=False,
)
validate_template_tool(file_path="<templates-directory>/<template-name>.yaml")
read_template_tool(file_path="<templates-directory>/<template-name>.yaml")
```

The kwargs are `file_path` and `template_payload` (the latter accepts a JSON-friendly dict). Pass `overwrite=True` only
when intentionally replacing an existing template.

`validate_template_tool` loads the template through `TaskTemplate.from_yaml`, which triggers `__post_init__` validation:

- cue codes are unique
- cue codes are in `[0, 255]`
- cue names are unique
- each cue name is non-empty and matches `^[A-Za-z0-9_]+$`, the same pattern trial names use, because the name is
  embedded in the `Cue_<name>_<length>cm` asset filename and in the space-joined cue-sequence signature on which Unity
  compares trials. Unity enforces the same pattern independently on its own side (see `unity:task-generator`)
- each cue `length_cm` is positive and finite
- each cue `texture` is a non-empty filename, because the field is required and carries no default
- each trial name matches `^[A-Za-z0-9_]+$`, used verbatim in the Unity-side `<TemplateName>-<TrialName>.prefab` segment
  filename. Barring the hyphen from both halves is what lets that filename split at its only hyphen and resolve to
  exactly one owning template
- each trial `cue_sequence` is non-empty and references valid cue names
- each trial `cue_sequence` is unique within the template, so two trials must not share an identical cue sequence. To
  make trials look identical to the animal yet stay distinguishable to the system, define distinct cue codes that share
  the same texture
- each trial `transitions`, when provided, references valid trial names, and every probability in it is finite and
  within `[0.0, 1.0]` before the set is checked to sum to 1.0 within a tolerance of `0.001`. The per-probability range
  is checked first because a negative weight would otherwise let the set still sum to 1.0 while silently removing its
  target from the sampled distribution
- `occupancy_duration_ms` is required on every trial whose `trigger_type` is `occupancy_disarm`, `occupancy_arm`, or
  `occupancy_trigger`, and any supplied value must be positive and finite. A non-occupancy trial leaves it `None`,
  because `0` is a real duration and is rejected on every trial whatever its trigger type
- each `TrialStructure.trigger_type` is a valid `TriggerType` value
- per-trial zone positions satisfy `start <= end` and `stimulus_location_cm >= start`, and all three are within the
  trial's segment length. This check is mode-aware. `collision` validates only the boundary (`stimulus_location`), since
  it has no trigger zone. `occupancy_trigger` validates only the trigger zone, since it has no boundary. The other three
  modes validate the zone, the boundary, and their ordering.
- `vr_environment.segments_per_corridor` is exactly an integer of at least 1, so a YAML `3.0` is rejected. The loader
  runs with per-field type checking disabled, which is why a float depth reaches the check unconverted
- `vr_environment.cm_per_unity_unit` and `vr_environment.corridor_spacing_cm` are positive and finite, and
  `vr_environment.cue_offset_cm` is finite. Zero and negative are legal for `cue_offset_cm`, because it is an offset.
  `vr_environment.padding_prefab_name` is not validated

`validate_template_tool` reports three outcomes. A missing file returns `success=False` carrying `error`. A load or
validation failure returns `success=True` with `valid=False` and a single-element `issues` list holding the first
violated constraint. A pass returns `valid=True` with a `summary` carrying `cue_count`, `trial_count`, and
`cue_offset_cm`. `__post_init__` stops at the first failure, so fix one issue and re-validate to surface the next. The
envelope in which this verdict rides is documented in the `## Response contract` section of
`/assets-mcp-environment-setup`.

`write_template_tool` re-serializes through the canonical `to_yaml`, so it writes no comments and drops any that a
rewritten file already carried. Re-apply the mandatory header block and the filename convention owned by
`unity:task-prefabs`, under "Template header" and "File naming", before handing off in Step 6.

### Step 6: Hand off for Unity task creation

When the template targets a Unity scene, hand off to **`unity:task-prefabs`** and run
`create_task_tool(template_name="<name>")`. The tool builds the task prefab and the matching scene in one call from the
same template basename, and both artifacts then live at `Assets/InfiniteCorridorTask/Tasks/<name>.prefab` and
`Assets/Scenes/<name>.unity`. If the Unity Editor, McpBridge, or `slsa mcp` is unreachable, restore connectivity via
`unity:unity-mcp-environment-setup` or `/assets-mcp-environment-setup` before proceeding.

### Step 7: Hand off for experiment configuration

Once the template is written and validated, projects that want to use it must author a per-project
`<System>ExperimentConfiguration` for their acquisition system via `/experiment-configuration`, which dispatches the
concrete class through `EXPERIMENT_CONFIGURATION_REGISTRY`. Today that is `MesoscopeExperimentConfiguration` for
Mesoscope-VR, whose field schema is owned by `mesoscope:mesoscope-vr-experiment-schema`. This skill is not responsible
for instantiating templates into experiment configurations.

---

## Common patterns

### Add a new trial entry to an existing paradigm

1. Read the template with `read_template_tool`.
2. Confirm the trigger type via `list_supported_trigger_types_tool`.
3. Add a new `TrialStructure` entry to the `trial_structures` dict, keyed by trial name. Specify the `cue_sequence`, the
   stimulus zone bounds, the `trigger_type`, and (if other trials should route to this one or vice versa) update the
   relevant `transitions` dicts.
4. Write the template back with `write_template_tool` (use `overwrite=True`).
5. Re-run `validate_template_tool` to confirm the new entry passes cross-reference checks.
6. Hand off to `/experiment-configuration` if a per-project experiment needs to pair this trial with a runtime trial
   class (on Mesoscope-VR, `MesoscopeWaterRewardTrial` or `MesoscopeGasPuffTrial`) and set its stimulus parameters. How
   often the new trial runs is set by the `transitions` dicts in step 3 above, not by the experiment configuration.

### Migrate a template to a new VR scene

1. Read the template with `read_template_tool`.
2. Mutate the `vr_environment` fields (corridor spacing, segments per corridor, padding prefab, cm-per-unity-unit) to
   match the new scene's geometry. The Unity scene name itself is bound by per-project experiment configurations, not by
   the template.
3. Run `validate_template_tool` to catch structural regressions.
4. Hand off to `unity:task-prefabs` to regenerate the prefab and re-verify segment zones against the new prefab state.

### Audit which projects use a template

1. List templates with `discover_templates_tool`.
2. Hand off to `/experiment-configuration` to enumerate consuming experiments per project.

---

## Related skills

| Skill                                      | Relationship                                                                                                 |
|--------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| `/cli-reference`                           | Owns the `slsa get templates` and `slsa configure templates` commands                                        |
| `/working-directory`                       | Required prerequisite that owns the templates directory path                                                 |
| `/assets-mcp-environment-setup`            | Run first if the MCP server is not connected                                                                 |
| `/experiment-configuration`                | Consumer that instantiates templates into per-project experiments and owns `list_supported_trial_types_tool` |
| `mesoscope:mesoscope-vr-experiment-schema` | Owns Mesoscope-VR's concrete trial-class field schema for the classes named here                             |
| `/library-extension`                       | Cross-cutting recipe to add a new `TriggerType` or runtime trial class                                       |
| `/datasets`                                | Owns the forged dataset copy of the template snapshot and the `inspect_datasets_tool` audit                  |
| `/session-data`                            | Owns `inspect_sessions_tool`, which locates the raw per-session template snapshot                            |
| `/session-discovery`                       | Resolves the session roots from which snapshot paths are built                                               |
| `unity:task-prefabs`                       | Downstream step that generates and validates the Unity prefab                                                |
| `unity:task-scenes`                        | Downstream step that opens and inspects the scene `create_task_tool` produced                                |
| `unity:zone-prefabs`                       | Owns the zone prefab each `TriggerType` mode bakes                                                           |
| `unity:mqtt-contract`                      | Owns the wire contract over which every trigger mode publishes                                               |
| `unity:task-generator`                     | Owns the `CreateTask` pipeline and the two-repo mirror recipe for adding a new template-driven field         |
| `experiment:vr-driver-interface`           | Consumer that decomposes the cue sequence into trials using these motifs and trigger types                   |

---

## Verification checklist

```text
- [ ] /working-directory has been run and the templates directory is set
- [ ] discover_templates_tool was called before creating a new template (avoid duplicates)
- [ ] describe_template_schema_tool was used as the source of truth for field names
- [ ] list_supported_trigger_types_tool was consulted for the trigger_type values
- [ ] list_supported_trial_types_tool was called with an acquisition_system argument for the runtime trial classes
- [ ] file_path was constructed as <templates-directory>/<template-name>.yaml (absolute path)
- [ ] Payload was passed as template_payload (the correct kwarg name)
- [ ] write_template_tool succeeded without schema errors
- [ ] validate_template_tool returned valid=True with no issues
- [ ] read_template_tool returned the expected content after the write
- [ ] If the template targets a Unity scene, unity:task-prefabs was invoked for prefab
      generation and validation
- [ ] If a NEW schema field was added, rather than just a new value, confirmed it is a two-repo mirror change, covering
      the Python dataclass in sollertia-shared-assets AND the matching C# [Serializable] field in
      sollertia-virtual-reality, which is the camelCase counterpart of the underscored YAML key with matching
      optionality and default. See the unity plugin's unity:task-generator "Adding a new template-driven field"
```
