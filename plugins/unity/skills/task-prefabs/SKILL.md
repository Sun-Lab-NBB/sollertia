---
name: task-prefabs
description: >-
  Generates, inspects, and validates Unity task prefabs for the sollertia-unity-tasks project from
  YAML task templates. Owns generate_task_prefab_tool, inspect_prefab_tool, and
  validate_prefab_against_template_tool. Use when a new task template needs a matching Unity prefab,
  when verifying that prefab zone positions match template values, or when auditing prefab hierarchy
  and colliders.
user-invocable: true
---

# Sollertia task prefabs

Authors and validates Unity task prefabs for the `sollertia-unity-tasks` project using the Unity
relay exposed by `slsa mcp`. This skill is the **exclusive** owner of `generate_task_prefab_tool`,
`inspect_prefab_tool`, and `validate_prefab_against_template_tool` — no other skill in the
marketplace may call these.

---

## Scope

**Covers:**
- Generating a Task prefab in Unity from a YAML task template (`generate_task_prefab_tool`)
- Inspecting a prefab's hierarchy, components, transforms, and colliders (`inspect_prefab_tool`)
- Validating segment prefab zone positions against a template's expected values
  (`validate_prefab_against_template_tool`)
- Template naming, header, and commenting conventions required for correct prefab generation

**Does not cover:**
- Authoring the YAML task template itself (see assets plugin's `/task-templates`)
- Authoring per-project experiment configurations (see assets plugin's `/experiment-configuration`)
- Scene and Unity asset enumeration (see `/scenes`)
- Entering / exiting Play Mode (see `/play-mode`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

---

## Template vs prefab

- A **template** is a YAML file under
  `Assets/InfiniteCorridorTask/Configurations/<name>.yaml`, authored by assets plugin's
  `/task-templates`. It describes the VR environment abstractly (cues, segments, trials).
- A **task prefab** is the concrete Unity GameObject hierarchy under
  `Assets/InfiniteCorridorTask/Tasks/<name>.prefab`, built by `generate_task_prefab_tool` from the
  template. It is the runtime representation the Unity scene instantiates.
- Segment prefabs under `Assets/InfiniteCorridorTask/Prefabs/Segment_*.prefab` are the reusable
  building blocks that both the template and the generated task prefab reference.

---

## MCP tool surface

| Tool                                      | Purpose                                                              |
|-------------------------------------------|----------------------------------------------------------------------|
| `generate_task_prefab_tool`               | Builds a Task prefab in Unity from a template (exclusive)            |
| `inspect_prefab_tool`                     | Returns a prefab's recursive GameObject tree (exclusive)             |
| `validate_prefab_against_template_tool`   | Checks segment prefab zone geometry against template values (exclusive) |

`list_unity_assets_tool` (owned by `/scenes`) may be called as a natural share when enumerating
prefabs before inspection.

---

## Template conventions (required for generation)

### File naming

```text
ProjectAbbreviation_TaskDescription.yaml
```

| Abbreviation | Project           |
|--------------|-------------------|
| MF           | MaalstroomicFlow  |
| SSO          | StateSpaceOdyssey |

- Use `_Base` suffix for single-segment training configurations.
- Capitalize each word in the task description.
- Template filename (without `.yaml`) is reused verbatim as the generated Unity scene / prefab name.

### Template header

Every template must begin with a YAML comment header:

```yaml
# Project: [Full project name]
# Purpose: [Single sentence describing the task structure]
# Layout:  [Segment names with cue letters and zone placements]
# Related: [Related template file (parenthetical explanation)]
```

Multi-line values align continuation text with the first character after the field name:

```yaml
# Project: MaalstroomicFlow
# Purpose: Extends the base MF_Reward trial structure to also include an aversive stimulus.
# Layout:  Segment ABCD with occupancy zone at cue C and aversive stimulus trigger zone in cue D.
#          Segment EFGH with the rewarding stimulus (water) trigger zone in cue H.
# Related: MF_Reward (the base version of this task that only includes the reward zone)
```

Header guidelines:
- **Project:** Full project name, not the abbreviation.
- **Purpose:** Single sentence starting with a verb (Defines, Extends, Teaches).
- **Layout:** Segment names with cue letters, zone types, and stimulus clarifications.
- **Related:** Parenthetical explanation of the relationship.

### Inline comments

Add inline YAML comments to clarify non-obvious values:

```yaml
cues:
  - name: "Gray"  # Placeholder — task does not use Gray cues.
    code: 0
    length_cm: 1.0
```

Templates that do not follow these conventions may still generate prefabs, but downstream tooling
(session preprocessing, dataset forging) relies on the naming contract.

---

## Generation workflow

### Step 1: Verify prerequisites

- `slsa mcp` server connected (else assets plugin's `/assets-mcp-environment-setup`).
- Unity Editor running with McpBridge listening (else `/unity-mcp-environment-setup` in this plugin).
- Template exists under `Assets/InfiniteCorridorTask/Configurations/<template_name>.yaml`. If not,
  hand off to assets plugin's `/task-templates` to author it first.

### Step 2: Generate the prefab

```text
generate_task_prefab_tool(template_name="<template-name>")
```

Or, to override the save path:

```text
generate_task_prefab_tool(
    template_name="<template-name>",
    save_path="Assets/InfiniteCorridorTask/Tasks/<custom-name>.prefab"
)
```

The tool delegates to Unity's CreateTask pipeline, which builds cue prefabs, segment prefabs, and
the full corridor hierarchy.

### Step 3: Inspect the result

```text
inspect_prefab_tool(prefab_path="Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab")
```

Verify the hierarchy matches the template — cue count, segment order, trial zones.

### Step 4: Validate zone geometry (required)

Programmatic validation is **mandatory**. Do not declare the prefab ready for downstream consumers
until `validate_prefab_against_template_tool` has been run and every segment reports `match: true`.

```text
validate_prefab_against_template_tool(template_name="<template-name>")
```

The tool checks each segment prefab referenced by the template's trial structures:
- Prefab file exists under `Assets/InfiniteCorridorTask/Prefabs/`.
- `StimulusTriggerZone` position matches the template's zone start/end cm values, converted using
  `cm_per_unity_unit`.
- Zone collider size matches the expected range width.

A `match: false` result means either the template or the prefab drifted. Resolve by:
1. If the prefab is authoritative (freshly generated), update the template via assets plugin's
   `/task-templates`.
2. If the template is authoritative (hand-edited), regenerate the prefab in Step 2.

If the tool itself cannot run (Unity Editor offline, McpBridge unreachable, or `slsa mcp` down),
**stop and warn the user**. Do not attempt to reconstruct the values by reading prefab YAML
manually — the validator is the single source of truth. Restore connectivity via
`/unity-mcp-environment-setup` (Unity side) or assets plugin's `/assets-mcp-environment-setup`
(slsa side), then re-run this step before handing off.

### Step 5: Hand off

- For scene placement: hand off to `/scenes` (`create_scene_tool` with `task_prefab_path`).
- For runtime testing: hand off to `/play-mode`.
- For per-project experiment configuration: hand off to assets plugin's `/experiment-configuration`.

---

## End-to-end task authoring workflow

Use this composite flow when a user asks for "a new task" rather than a single-step operation. Each step is owned by
a different skill; this section is the canonical ordering.

| Step | Skill (owner)                           | Action                                                                    |
|------|-----------------------------------------|---------------------------------------------------------------------------|
| 1    | assets `/task-templates`                | Author `Assets/InfiniteCorridorTask/Configurations/<name>.yaml`           |
| 2    | `/task-prefabs` (this skill)            | `generate_task_prefab_tool(template_name="<name>")`                       |
| 3    | `/task-prefabs` (this skill)            | `inspect_prefab_tool(prefab_path=…)` — sanity-check hierarchy             |
| 4    | `/task-prefabs` (this skill)            | `validate_prefab_against_template_tool(template_name="<name>")`           |
| 5    | `/scenes`                               | `create_scene_tool(scene_name="<name>", task_prefab_path=…)`              |
| 6    | `/scenes`                               | `open_scene_tool(scene_path=…)` to switch the Editor                      |
| 7    | `/scene-setup`                          | Configure Display rig and optional `SimulatedLinearTreadmill`             |
| 8    | `/play-mode`                            | `enter_play_mode_tool()` → exercise → `exit_play_mode_tool()`             |
| 9    | assets `/experiment-configuration`      | (Optional) Bind the template to a per-project experiment configuration    |

Checkpoints between steps:

- **Step 2 → 3:** stop if `generate_task_prefab_tool` returns `success: false`. Common cause is a mistyped
  `template_name` or a malformed YAML header.
- **Step 3 → 4:** stop if the hierarchy contradicts the template (e.g. missing corridor count). Fix the template,
  regenerate (Step 2), do not hand-patch the prefab.
- **Step 4 → 5:** stop if any segment reports `match: false`. Regenerate from the template or correct the template;
  never ship a task prefab with drift.
- **Step 7 → 8:** stop if a display panel is missing — Play Mode without displays throws runtime null-reference
  errors in `ActorObject.Display`.

---

## Template / prefab round-trip invariants

The same values live in two places: the YAML template and the generated Unity prefab. These invariants determine which
side is authoritative and how drift is resolved.

### Fields that flow YAML → prefab (at generation)

| Template field                      | Becomes                                                                |
|-------------------------------------|------------------------------------------------------------------------|
| `cues[].name`                       | `Cue_<name>.prefab` file name and material name                        |
| `cues[].texture`                    | Main texture on the `Cue_<name>` material                              |
| `cues[].length_cm` + `cm_per_unity_unit` | Cue quad mesh scale (Z axis)                                      |
| `segments[].name`                   | `<segment-name>.prefab` file name                                      |
| `segments[].cue_sequence`           | Ordered cue instances along Z inside the segment                       |
| `vr_environment.padding_prefab_name`| Padding prefab loaded and appended past every corridor                 |
| `vr_environment.segments_per_corridor` | Depth parameter for the `Corridor<indices>` hierarchy under the task|
| `vr_environment.cm_per_unity_unit`  | Conversion factor for **all** cm-valued fields below                   |
| `trial_structures[].trigger_type`   | Selects `StimulusTriggerZone.prefab` vs `OccupancyTriggerZone.prefab`  |
| `trial_structures[].stimulus_trigger_zone_start_cm` / `_end_cm` | Root `BoxCollider.size.z` and `.center.z`  |
| `trial_structures[].stimulus_location_cm` | Child `GuidanceRegion` / `OccupancyRegion` collider center       |
| `trial_structures[].show_stimulus_collision_boundary` | `StimulusTriggerZone.showBoundary` on the root    |

### Fields authored only in the prefab

These have no YAML representation and are set by hand in the Editor when authoring a segment prefab for the first time:

- Floor and wall mesh scale, material references, and colliders
- Camera rig (owned by `ExperimentTemplate.unity`, not by segment prefabs)
- Reset zone position inside each segment (placed automatically by `CreateTask` at local Z = 1)
- `ResetZone.prefab`, `StimulusTriggerZone.prefab`, `OccupancyTriggerZone.prefab` internal hierarchies (the template
  references them by trigger type, but not their contents)

### Fields the validator compares (prefab → YAML)

`validate_prefab_against_template_tool` reads the generated prefab back and re-derives values to compare against the
template. These are the only fields it asserts round-trip equality on:

| Validated field           | Tolerance |
|---------------------------|-----------|
| `zone_z` (center)         | ±0.01 Unity units |
| `zone_size` (collider Z)  | ±0.01 Unity units |

Other fields (cue lengths, materials, segment length) are **not** round-trip validated by the tool. If you change a
cue length, you must regenerate the prefab — nothing will tell you the prefab is stale.

### Which side wins on drift

Use this decision table when `validate_prefab_against_template_tool` reports `match: false`:

| Situation                                                    | Authoritative | Action                                                 |
|--------------------------------------------------------------|---------------|--------------------------------------------------------|
| You just edited the template and the prefab hasn't been regenerated | Template | Run `generate_task_prefab_tool` again                 |
| You just edited the segment prefab by hand in the Editor     | Prefab        | Update the template via assets `/task-templates`       |
| Both were edited in parallel                                 | Template      | Regenerate; re-apply prefab edits manually afterward   |
| Neither was edited recently                                  | Template      | Treat drift as a bug; regenerate and investigate why   |

Rule of thumb: **the template is the source of truth**. Only promote the prefab to authoritative when you deliberately
hand-edited a segment prefab to explore a design before updating the template.

---

## Reading inspect_prefab_tool output

`inspect_prefab_tool` returns a recursive JSON hierarchy. The top-level object is the task prefab; its children are
`Corridor<indices>` objects, each containing segment instances. Within each first-segment instance of a corridor, the
stimulus zone hierarchy varies by `trigger_type`. Use the following shapes to interpret the JSON.

### Lick mode (trigger_type == "lick")

```text
<SegmentName>
└── StimulusTriggerZone              ← collider_size.z = zone width
    │                                  components: ["StimulusTriggerZone", "BoxCollider", "MeshRenderer"]
    └── GuidanceRegion               ← collider_center.z offset = stimulus_location - zone_center
                                       components: ["GuidanceZone", "BoxCollider"]
```

Key markers:
- Root has `StimulusTriggerZone` in `components` and a `BoxCollider` of size.z ≈ `(zone_end - zone_start) / cm_per_unit`.
- Exactly one child named `GuidanceRegion` with `GuidanceZone` in components.
- `MeshRenderer` on the root is only visible when `showBoundary == true` (template field
  `show_stimulus_collision_boundary`).

### Occupancy mode (trigger_type == "occupancy")

```text
<SegmentName>
└── StimulusTriggerZone              ← collider is the boundary past the occupancy range
    ├── OccupancyRegion              ← collider covers the wait range (offset by center.z)
    │                                  components: ["OccupancyZone", "BoxCollider"]
    └── OccupancyGuidanceRegion      ← placed at the downstream end of the occupancy range
                                       components: ["OccupancyGuidanceZone", "BoxCollider"]
```

Key markers:
- Root has `StimulusTriggerZone` in `components`. Its position is the stimulus boundary — this is **past** the
  occupancy waiting range by design.
- Two children: `OccupancyRegion` (wait zone) and `OccupancyGuidanceRegion` (guidance activation near the boundary).
- If only one child exists, the prefab is miswired. Regenerate from the template.

### Disarmed segments (not the first segment of a corridor)

`CreateTask` strips zones from segments at corridor depth > 0 because they are visual-only. An `inspect_prefab_tool`
result where every `Segment_*` under a corridor except the first has no `StimulusTriggerZone` child is expected
behavior, not a bug.

### Components you should ignore

`MeshFilter`, `MeshRenderer`, `BoxCollider` without one of the zone scripts, `Transform` — these are either visual
geometry (floor, walls, cue quads) or the standard Unity components every GameObject carries.

---

## Zone behavior reference

Both trigger zone prefabs use `StimulusTriggerZone.cs` as the root script. Behavior is determined by
the child zone GameObject:

### Lick mode (`StimulusTriggerZone.prefab`)

Structure: `StimulusTriggerZone` (trigger collider) → `GuidanceRegion` (`GuidanceZone.cs`).

Runtime:
- `requireLick=true`: animal must lick inside the trigger zone to receive stimulus.
- `requireLick=false`: stimulus delivered when animal reaches the GuidanceZone OR licks in the
  trigger zone.

Template fields:
- `stimulus_trigger_zone_start_cm` / `stimulus_trigger_zone_end_cm`: licking range.
- `stimulus_location_cm`: position inside the trigger zone range.
- `trigger_type: "lick"`.

### Occupancy mode (`OccupancyTriggerZone.prefab`)

Structure: `StimulusTriggerZone` (boundary collider) → `OccupancyRegion` (`OccupancyZone.cs`,
offset by `m_Center.z`) → `OccupancyGuidanceRegion` (`OccupancyGuidanceZone.cs`).

Runtime:
1. Animal enters `OccupancyRegion` (offset from boundary by collider center) → occupancy timer
   starts.
2. If animal stays for `occupancyDurationMs` → boundary **disarmed**, animal passes safely.
3. If animal reaches the boundary while armed → stimulus fires.

Template fields:
- `stimulus_trigger_zone_start_cm` / `stimulus_trigger_zone_end_cm`: occupancy waiting range
  (derived from the `OccupancyRegion` collider).
- `stimulus_location_cm`: boundary position (derived from the root collider). This is **outside**
  the waiting range by design.
- `trigger_type: "occupancy"`.

---

## Troubleshooting

| Symptom                                         | Cause                                                | Resolution                                      |
|-------------------------------------------------|------------------------------------------------------|-------------------------------------------------|
| `generate_task_prefab_tool` returns "template not found" | Template file missing from `Configurations/`   | Hand off to assets plugin's `/task-templates`   |
| `validate_prefab_against_template_tool` reports `match: false` | Template or prefab drifted                | Regenerate prefab (Step 2) or fix template      |
| `inspect_prefab_tool` returns "prefab path missing" | Prefab not saved to `Tasks/`                     | Re-run Step 2 with an explicit `save_path`      |
| All Unity tools return "Unity Editor is not reachable" | Editor or McpBridge offline                   | `/unity-mcp-environment-setup` in this plugin         |
| Trigger type mismatch between template and prefab | GUID reference drift                               | Open the prefab in the Editor and re-link zone  |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] slsa mcp server is connected
- [ ] Target template exists under Assets/InfiniteCorridorTask/Configurations/
- [ ] Template filename follows the ProjectAbbreviation_TaskDescription convention
- [ ] generate_task_prefab_tool succeeded and returned a prefab_path
- [ ] inspect_prefab_tool returned a hierarchy matching the template's cue / segment / trial counts
- [ ] validate_prefab_against_template_tool reported match: true for every segment
- [ ] Did not hand-edit the generated prefab — always regenerate from the template
```

---

## Related skills

| Skill                                         | Relationship                                              |
|-----------------------------------------------|-----------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                  |
| `/scenes` (this plugin)                       | Consumer — places the generated prefab into a scene       |
| `/play-mode` (this plugin)                    | Consumer — exercises the prefab at runtime                |
| `/scene-setup` (this plugin)                  | Consumer — configures displays / controller before Play Mode |
| `/task-generator` (this plugin)               | Reference for the `CreateTask` pipeline this tool invokes |
| `/mqtt-contract` (this plugin)                | Reference for MQTT topics wired by generated zone scripts |
| `/gimbl-framework` (this plugin)              | Reference for `ActorObject` coordinate frame usage        |
| assets plugin `/task-templates`               | Upstream — owns the YAML template the prefab is built from|
| assets plugin `/experiment-configuration`     | Downstream — per-project instantiation of the template    |
| assets plugin `/assets-mcp-environment-setup` | Run first — owns the slsa MCP server diagnostic           |
