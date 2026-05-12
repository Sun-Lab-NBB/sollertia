---
name: task-prefabs
description: >-
  Generates, inspects, validates, and regenerates Unity task prefabs for sollertia-unity-tasks
  from YAML task templates. Owns generate_task_prefab_tool, inspect_prefab_tool,
  validate_prefab_against_template_tool, and delete_unity_asset_tool. Use when a template needs a
  matching prefab, when verifying prefab geometry against the template, when forcing regeneration
  after a template edit, or when auditing prefab hierarchy and colliders.
user-invocable: true
---

# Sollertia task prefabs

Authors and validates Unity task prefabs for the `sollertia-unity-tasks` project using the Unity
relay exposed by `slsa mcp`. This skill is the **exclusive** owner of `generate_task_prefab_tool`,
`inspect_prefab_tool`, `validate_prefab_against_template_tool`, and `delete_unity_asset_tool` — no
other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Generating a Task prefab in Unity from a YAML task template (`generate_task_prefab_tool`)
- Inspecting a prefab's hierarchy, components, transforms, and colliders (`inspect_prefab_tool`)
- Validating cue inventory, segment geometry, cue ordering, and zone positions against a template
  (`validate_prefab_against_template_tool`)
- Deleting a regenerable cue, segment, or task prefab to force a fresh build on the next
  generation pass (`delete_unity_asset_tool`)
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
- Segment prefabs under `Assets/InfiniteCorridorTask/Prefabs/<template_name>_<trial_name>.prefab`
  are the building blocks that the generated task prefab references. Segment prefab filenames are
  derived directly from the template filename and the trial key under `trial_structures`, so each
  template owns its segments outright; identical-geometry trials in different templates no longer
  collapse to a shared prefab.

---

## MCP tool surface

| Tool                                      | Purpose                                                                 |
|-------------------------------------------|-------------------------------------------------------------------------|
| `generate_task_prefab_tool`               | Builds a Task prefab in Unity from a template (exclusive)               |
| `inspect_prefab_tool`                     | Returns a prefab's recursive GameObject tree (exclusive)                |
| `validate_prefab_against_template_tool`   | Checks cue inventory, segment geometry, and zone positions (exclusive)  |
| `delete_unity_asset_tool`                 | Removes a regenerable prefab so the next generation pass rebuilds it    |

`list_unity_assets_tool` (owned by `/scenes`) may be called as a natural share when enumerating
prefabs before inspection. `delete_unity_asset_tool` is exclusive to this skill for prefab and
material deletion under `Assets/InfiniteCorridorTask/` but may be called by `/scenes` as a natural
share for scene deletion under `Assets/Scenes/`.

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

The tool returns a top-level `cue_prefabs` list with per-cue prefab existence, plus a `trials`
list. Each trial entry reports:

- `trial`: The trial name (the key in the template's `trial_structures` dict).
- `canonical_name`: The derived segment prefab filename — always `<template_name>_<trial_name>`
  (e.g. `MF_Reward_Base_ABCD`).
- `prefab_exists`: Segment prefab file is present under `Assets/InfiniteCorridorTask/Prefabs/`.
- `cue_order` / `expected_cue_order` / `cue_order_match`: Cue child names along the segment's local
  Z axis match the trial's `cue_sequence`.
- `segment_length_unity` / `expected_segment_length_unity` / `segment_length_match`: Measured
  prefab Z-extent matches the cue-sum length (tolerance ±0.01 Unity units).
- `has_zone`: Segment prefab carries a `StimulusTriggerZone`.
- `zone_z` / `expected_zone_z` / `zone_z_match`: Zone collider center on Z matches the trial's
  zone start/end midpoint, converted via `cm_per_unity_unit`.
- `zone_size` / `expected_zone_size` / `zone_size_match`: Zone collider Z-size matches the trial's
  zone range width.

Any `*_match: false` result means the template or the prefab drifted. Resolve by regenerating from
the template (see [Regenerating after template edits](#regenerating-after-template-edits)) or by
correcting the template via assets plugin's `/task-templates` when the prefab is authoritative.

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

## Regenerating after template edits

`generate_task_prefab_tool` always rebuilds the **task prefab** and every **segment prefab** the
template owns: the Unity-side `CreateTask` pipeline deletes the previous `<template>_<trial>.prefab`
files before regenerating them, so trial-parameter edits in the YAML (cue sequence, zone math,
trigger type) take effect on the next generation pass without any manual cleanup.

**Cue prefabs and materials are different.** They are keyed by cue name and length only
(`Cue_<name>_<length>cm.prefab` / `.mat`) and are shared across every template that declares a
matching cue. `CreateTask` reuses an existing cue prefab whenever it finds one on disk, so editing
a cue's **texture** while keeping the same name and length will not propagate until the cue prefab
and its material are deleted. The validator catches the resulting drift on the segment side; the
fix is to delete the cue and regenerate.

### Picking what to delete

| Template change                                                       | Delete                                                                                                                                 |
|-----------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `cues[].texture` for cue X (name and length unchanged)                | `Assets/InfiniteCorridorTask/Cues/Cue_X_<length>cm.prefab` **and** `Assets/InfiniteCorridorTask/Materials/Cue_X_<length>cm.mat`         |
| `cues[].length_cm` for cue X                                          | Nothing — a new length yields a new `Cue_X_<new-length>cm.prefab` automatically                                                        |
| `trial_structures[T].cue_sequence` for trial T                        | Nothing — the segment prefab is regenerated automatically                                                                              |
| `trial_structures[T]` zone math for trial T                           | Nothing — the segment prefab is regenerated automatically                                                                              |
| `vr_environment.cm_per_unity_unit` (rescaled units affect every cue)  | All cue prefabs *and* materials under `Cues/` / `Materials/` whose lengths are affected                                                |

The task prefab itself (`Assets/InfiniteCorridorTask/Tasks/<template>.prefab`) is rebuilt by every
`generate_task_prefab_tool` call.

### Workflow

1. **Confirm the drift** — run `validate_prefab_against_template_tool` to identify which cues or
   segments report `*_match: false`.
2. **Delete stale cue assets** (only when cue textures or shared cue geometry changed):
   ```text
   delete_unity_asset_tool(asset_path="Assets/InfiniteCorridorTask/Cues/Cue_X_30cm.prefab")
   delete_unity_asset_tool(asset_path="Assets/InfiniteCorridorTask/Materials/Cue_X_30cm.mat")
   ```
   The bridge refuses paths outside the InfiniteCorridorTask asset roots and refuses the four
   hand-authored protected assets (`StimulusTriggerZone.prefab`, `OccupancyTriggerZone.prefab`,
   `ResetZone.prefab`, `ExperimentTemplate.unity`). If the bridge rejects a path, do not bypass —
   the asset is hand-authored and the template should reference a different name. Because cue
   assets are shared across templates, deleting them will force every dependent template to
   regenerate its cues on its next `generate_task_prefab_tool` call (a no-op when the new cue
   matches what the dependent template would have produced).
3. **Regenerate** — re-run `generate_task_prefab_tool` (Step 2 of the generation workflow). The
   pipeline rebuilds every segment prefab from scratch and fills in any missing cue prefabs.
4. **Re-validate** — run `validate_prefab_against_template_tool` and confirm every `*_match` field
   is `true` before handing off downstream.

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
- **Step 4 → 5:** stop if any segment reports `*_match: false` (cue order, segment length, zone z, or zone size) or
  any cue reports `exists: false`. See [Regenerating after template edits](#regenerating-after-template-edits) and
  the [drift decision table](#which-side-wins-on-drift). Never ship a task prefab with drift.
- **Step 7 → 8:** stop if a display panel is missing — Play Mode without displays throws runtime null-reference
  errors in `ActorObject.Display`.

---

## Template / prefab round-trip invariants

The same values live in two places: the YAML template and the generated Unity prefab. These invariants determine which
side is authoritative and how drift is resolved.

### Fields that flow YAML → prefab (at generation)

| Template field                                                  | Becomes                                                                |
|-----------------------------------------------------------------|------------------------------------------------------------------------|
| `cues[].name`                                                   | `Cue_<name>_<length>cm.prefab` file name and material name             |
| `cues[].texture`                                                | Main texture on the cue material                                       |
| `cues[].length_cm` + `cm_per_unity_unit`                        | Cue quad mesh scale (Z axis)                                           |
| `trial_structures[].cue_sequence` + `cues[].length_cm`          | Drives cue child ordering and segment geometry (filename is template-and-trial-derived) |
| Template filename + `trial_structures` key                      | Generates the `<template>_<trial>.prefab` segment filename and in-prefab `m_Name`       |
| `vr_environment.padding_prefab_name`                            | Padding prefab loaded and appended past every corridor                 |
| `vr_environment.segments_per_corridor`                          | Depth parameter for the `Corridor<indices>` hierarchy under the task   |
| `vr_environment.cm_per_unity_unit`                              | Conversion factor for **all** cm-valued fields below                   |
| `vr_environment.cue_offset_cm`                                  | Upstream shift of the segment prefab's local origin and ResetZone z    |
| `trial_structures[].trigger_type`                               | Selects `StimulusTriggerZone.prefab` vs `OccupancyTriggerZone.prefab`  |
| `trial_structures[].stimulus_trigger_zone_start_cm` / `_end_cm` | Root `BoxCollider.size.z` and `.center.z`                              |
| `trial_structures[].stimulus_location_cm`                       | Child `GuidanceRegion` / `OccupancyRegion` collider center             |
| `trial_structures[].show_stimulus_collision_boundary`           | `StimulusTriggerZone.showBoundary` on the root                         |

### Fields authored only in the prefab

These have no YAML representation and are set by hand in the Editor when authoring a segment prefab for the first time:

- Floor and wall mesh scale, material references, and colliders
- Camera rig (owned by `ExperimentTemplate.unity`, not by segment prefabs)
- Reset zone position inside each segment (placed automatically by `CreateTask` at local Z = 1)
- `ResetZone.prefab`, `StimulusTriggerZone.prefab`, `OccupancyTriggerZone.prefab` internal hierarchies (the template
  references them by trigger type, but not their contents)

### Fields the validator compares (prefab → YAML)

`validate_prefab_against_template_tool` reads the generated prefab back and re-derives values to compare against the
template. These are the fields it asserts round-trip equality on:

| Validated field            | Scope          | Tolerance         |
|----------------------------|----------------|-------------------|
| Cue prefab existence       | Per cue        | Exact (file present) |
| `cue_order`                | Per segment    | Exact (sequence equality) |
| `segment_length_unity`     | Per segment    | ±0.01 Unity units |
| `zone_z` (center)          | Per segment    | ±0.01 Unity units |
| `zone_size` (collider Z)   | Per segment    | ±0.01 Unity units |

Cue textures and materials are not round-trip validated — Unity-side material edits are silent until you delete the
cue prefab and regenerate. The lick-mode zone-math comparison still applies as before; occupancy-mode segments will
report `zone_z_match: false` even when correctly generated (see `/task-generator` for the formula divergence).

### Which side wins on drift

Use this decision table when `validate_prefab_against_template_tool` reports any `*_match: false`:

| Failure                                              | Likely cause                                                          | Action                                                                                                                |
|------------------------------------------------------|-----------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| `cue_prefabs[].exists: false`                        | Cue referenced by a segment was never generated, or was deleted       | Run `generate_task_prefab_tool` again to rebuild missing cue prefabs                                                  |
| `trials[].prefab_exists: false`                      | Segment prefab missing from `Prefabs/`                                | Run `generate_task_prefab_tool` again — segments are always rebuilt                                                   |
| `cue_order_match: false`                             | Stale segment prefab from before the segment-always-regenerate flow   | Run `generate_task_prefab_tool` again — the segment is rebuilt from the current `cue_sequence`                        |
| `segment_length_match: false`                        | Cue lengths were edited without deleting the affected cue prefabs     | [Regenerate](#regenerating-after-template-edits) — delete the cue prefabs (and materials), re-run generation          |
| `zone_z_match: false` / `zone_size_match: false`     | Trial zone math drift (likely from a pre-fix prefab); or occupancy mode (known divergence) | Lick mode → re-run `generate_task_prefab_tool`. Occupancy mode → cross-check against `/task-generator` formula reference. |
| All `*_match: true`, but the prefab is hand-edited   | Editor hand-edits drifted out of the validator's coverage             | Update the template via assets `/task-templates`, then regenerate so the YAML is the source of truth                  |

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
result where every segment instance under a corridor except the first has no `StimulusTriggerZone` child is expected
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

| Symptom                                                          | Cause                                              | Resolution                                                       |
|------------------------------------------------------------------|----------------------------------------------------|------------------------------------------------------------------|
| `generate_task_prefab_tool` returns "template not found"         | Template file missing from `Configurations/`       | Hand off to assets plugin's `/task-templates`                    |
| `validate_prefab_against_template_tool` reports `*_match: false` | Template or prefab drifted                         | See [drift decision table](#which-side-wins-on-drift)            |
| `inspect_prefab_tool` returns "prefab path missing"              | Prefab not saved to `Tasks/`                       | Re-run Step 2 with an explicit `save_path`                       |
| All Unity tools return "Unity Editor is not reachable"           | Editor or McpBridge offline                        | `/unity-mcp-environment-setup` in this plugin                    |
| Trigger type mismatch between template and prefab                | GUID reference drift                               | Open the prefab in the Editor and re-link zone                   |
| `delete_unity_asset_tool` rejects the path with "Refusing to delete" | Path is outside the InfiniteCorridorTask roots, or names a hand-authored asset | Reference a different name in the template; do not bypass the protection |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] slsa mcp server is connected
- [ ] Target template exists under Assets/InfiniteCorridorTask/Configurations/
- [ ] Template filename follows the ProjectAbbreviation_TaskDescription convention
- [ ] generate_task_prefab_tool succeeded and returned a prefab_path
- [ ] inspect_prefab_tool returned a hierarchy matching the template's cue / segment / trial counts
- [ ] validate_prefab_against_template_tool reported every cue_prefabs[].exists, prefab_exists,
      cue_order_match, segment_length_match, and zone_*_match field as true (or documented occupancy
      drift for any zone_*_match: false)
- [ ] After a template edit that changed a cue texture without renaming the cue, deleted the stale
      cue prefab and material via delete_unity_asset_tool before regenerating
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
