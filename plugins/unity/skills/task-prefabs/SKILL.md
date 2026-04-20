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

| Skill                                   | Relationship                                              |
|-----------------------------------------|-----------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                  |
| `/scenes`                               | Consumer — places the generated prefab into a scene       |
| `/play-mode`                            | Consumer — exercises the prefab at runtime                |
| assets plugin `/task-templates`         | Upstream — owns the YAML template the prefab is built from|
| assets plugin `/experiment-configuration` | Downstream — per-project instantiation of the template  |
| assets plugin `/assets-mcp-environment-setup`  | Run first — owns the slsa MCP server diagnostic           |
