---
name: task-prefabs
description: >-
  Creates, deletes, and inspects Unity tasks for sollertia-unity-tasks from YAML task templates.
  Owns create_task_tool (single-step template → prefab + scene), delete_task_tool (single-step
  removal of every generated artifact for a task), inspect_prefab_tool, and delete_asset_tool
  (individual cue / material cleanup). Use when a template needs a matching task built or removed,
  or when auditing prefab hierarchy and colliders.
user-invocable: true
---

# Sollertia Unity task prefabs

Creates, deletes, and inspects Unity tasks for the `sollertia-unity-tasks` project through the
Unity relay exposed by `slsa mcp` — the **exclusive** owner of `create_task_tool`,
`delete_task_tool`, `inspect_prefab_tool`, and `delete_asset_tool`, which no other skill in the
marketplace may call.

---

## Scope

**Covers:**
- Creating a Unity task end-to-end from a YAML task template — task prefab plus matching scene
  in one call (`create_task_tool`)
- Deleting a Unity task end-to-end — scene plus per-scene companion plus task prefab plus every
  segment prefab in one call (`delete_task_tool`)
- Inspecting a prefab's hierarchy, components, transforms, and colliders (`inspect_prefab_tool`)
- Deleting a regenerable cue prefab or cue material individually (`delete_asset_tool`)
- Template naming, header, and commenting conventions required for correct task creation

**Does not cover:**
- Authoring the YAML task template itself (see assets plugin's `/task-templates`)
- Authoring per-project experiment configurations (see assets plugin's `/experiment-configuration`)
- Listing, opening, or inspecting scenes (see `/task-scenes`)
- Entering / exiting Play Mode (see `/play-mode`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

---

## Asset chain on disk

A single task is three name-aligned files on disk. The base name (`MF_Reward`, `SSO_Reversal`, …)
is greppable across all three:

```text
Assets/InfiniteCorridorTask/Configurations/<name>.yaml      template          (authored by assets /task-templates)
                                │
                                │  create_task_tool (single call)
                                ▼
Assets/InfiniteCorridorTask/Tasks/<name>.prefab             task prefab
Assets/Scenes/<name>.unity                                  scene
```

`create_task_tool` is the **only** path that produces the prefab + scene pair, and the basename
convention is enforced by the tool itself: `template_name="MF_Reward"` produces
`Tasks/MF_Reward.prefab` and `Scenes/MF_Reward.unity` unconditionally. Both MCP `create_task_tool`
and the `CreateTask → New Task` Editor menu reject templates outside the `Configurations/`
directory so the cross-template cue-texture preflight, the runtime config-path resolver, and
downstream tooling all see a single canonical home — use assets plugin's `/working-directory`
(`set_task_templates_directory_tool`) to configure the MCP-side path.

Two derived artifact tiers sit below the task prefab:

- **Segment prefabs** under `Assets/InfiniteCorridorTask/Prefabs/<name>_<trial-name>.prefab` —
  one per template `trial_structures` entry, template-owned (every trial gets its own segment
  prefab even when two trials in different templates have identical geometry).
- **Cue prefabs** under `Assets/InfiniteCorridorTask/Cues/Cue_<cuename>_<length>cm.prefab` —
  **shared** across every template that declares the same `(name, length_cm)` cue identity.

For the abstract template model (cue / segment / trial vocabulary, transition graph,
sliding-window corridor traversal), see assets plugin's `/task-templates`. For the
`CreateTask.CreateFromTemplate` pipeline internals (cue / segment build passes, shader chain,
zone placement math, hand-authored protected assets), see `/task-generator`.

---

## MCP tool surface

| Tool                  | Purpose                                                                                                       |
|-----------------------|---------------------------------------------------------------------------------------------------------------|
| `create_task_tool`    | Builds a task prefab and the matching scene from a template in one call (exclusive)                           |
| `delete_task_tool`    | Removes the scene, the per-scene companion, the task prefab, and every segment prefab in one call (exclusive) |
| `inspect_prefab_tool` | Returns a prefab's recursive GameObject tree (exclusive)                                                      |
| `delete_asset_tool`   | Removes a regenerable cue prefab or cue material (exclusive)                                                  |

`list_assets_tool` (owned by `/task-scenes`) is callable as a natural share when enumerating
prefabs before inspection. `delete_asset_tool` is scoped to non-scene assets under
`Assets/InfiniteCorridorTask/`; the handler rejects scene paths so scene cleanup goes through
`delete_task_tool` and preserves the per-scene `savedFullScreenViews` companion cascade.
`delete_task_tool` is the inverse of `create_task_tool`.

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
- Template exists under `Assets/InfiniteCorridorTask/Configurations/<template-name>.yaml`. If not,
  hand off to assets plugin's `/task-templates` to author it first.

### Step 2: Create the task

```text
create_task_tool(template_name="<template-name>")
```

A single call builds the task prefab at
`Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab` and the matching scene at
`Assets/Scenes/<template-name>.unity`. Both paths are auto-resolved from the template basename
and cannot be overridden — every artifact of one task is greppable by one name. The tool
delegates to Unity's CreateTask pipeline (cue prefabs, segment prefabs, corridor hierarchy)
followed by `CreateSceneFromTemplate` (copies `ExperimentTemplate.unity`, instantiates the new
prefab, runs `EnsureControllers` / `EnsureMqttDefaults` / `SyncDisplayBrightnessToSettings`).

If a scene already exists at the resolved path, the tool refuses and points at `delete_task_tool`
(this skill); a regeneration cycle is always `delete_task_tool` → `create_task_tool`.

### Step 3: Inspect the result

```text
inspect_prefab_tool(prefab_path="Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab")
```

Verify the hierarchy matches the template — cue count, segment order, trial zones.

### Step 4: Hand off

- The scene already exists at `Assets/Scenes/<template-name>.unity` — `create_task_tool` produced
  it in step 2. For navigation between scenes, hand off to `/task-scenes`.
- For runtime testing: hand off to `/play-mode`.
- For per-project experiment configuration: hand off to assets plugin's `/experiment-configuration`.

---

## Deleting a task

`delete_task_tool` is the inverse of `create_task_tool`. A single call removes every Unity
artifact that `create_task_tool` produced for a template:

```text
delete_task_tool(template_name="<template-name>")
```

Removes the scene at `Assets/Scenes/<template-name>.unity` (plus its
`savedFullScreenViews.asset` companion via the same cascade `delete_task_tool` uses), the task
prefab at `Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab`, and every segment prefab
under `Assets/InfiniteCorridorTask/Prefabs/` whose filename begins with `<template-name>_`. The
template YAML and the shared cue prefabs / materials are preserved — cues live in
`Assets/InfiniteCorridorTask/Cues/` and are referenced by sibling tasks, so individual cue
cleanup goes through `delete_asset_tool`. The response carries `deleted_paths` (every
asset removed) and `companion_deleted` (the saved-views asset when present); when no artifacts
exist for the supplied template the call returns an error.

---

## Regenerating after template edits

The regeneration cycle is always **`delete_task_tool` → `create_task_tool`**: `create_task_tool`
refuses to overwrite an existing scene, so the agent removes the existing bundle (scene + per-scene
companion + task prefab + every segment prefab) before rebuilding. `create_task_tool` always
rebuilds the **task prefab** and every **segment prefab** the template owns regardless of which
files survived the deletion. The Unity-side `CreateTask` pipeline deletes the previous
`<template-name>_<trial-name>.prefab` files before regenerating them, so trial-parameter edits in the YAML
(cue sequence, zone math, trigger type) take effect on the next call without any manual cleanup
beyond the bundle delete.

**Cue prefabs and materials are different.** They are keyed by cue name and length only
(`Cue_<name>_<length>cm.prefab` / `.mat`) and are shared across every template that declares a
matching cue. `CreateTask` reuses an existing cue prefab whenever it finds one on disk, so editing
a cue's **texture** while keeping the same name and length will not propagate until the cue prefab
and its material are deleted. The validator catches the resulting drift on the segment side; the
fix is to delete the cue and regenerate.

### Picking what to delete

| Template change                                                      | Delete                                                                                                                          |
|----------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `cues[].texture` for cue X (name and length unchanged)               | `Assets/InfiniteCorridorTask/Cues/Cue_X_<length>cm.prefab` **and** `Assets/InfiniteCorridorTask/Materials/Cue_X_<length>cm.mat` |
| `cues[].length_cm` for cue X                                         | Nothing — a new length yields a new `Cue_X_<new-length>cm.prefab` automatically                                                 |
| `trial_structures[T].cue_sequence` for trial T                       | Nothing — the segment prefab is regenerated automatically                                                                       |
| `trial_structures[T]` zone math for trial T                          | Nothing — the segment prefab is regenerated automatically                                                                       |
| `vr_environment.cm_per_unity_unit` (rescaled units affect every cue) | All cue prefabs *and* materials under `Cues/` / `Materials/` whose lengths are affected                                         |

The task prefab and the scene are rebuilt by every successful `create_task_tool` call.

### Workflow

1. **Remove the existing task bundle** (always required — `create_task_tool` refuses to overwrite
   an existing scene):
   ```text
   delete_task_tool(template_name="<template-name>")
   ```
   Cue prefabs and materials are deliberately preserved.
2. **Delete stale cue assets** (only when cue textures or shared cue geometry changed):
   ```text
   delete_asset_tool(asset_path="Assets/InfiniteCorridorTask/Cues/Cue_X_30cm.prefab")
   delete_asset_tool(asset_path="Assets/InfiniteCorridorTask/Materials/Cue_X_30cm.mat")
   ```
   Allowed roots and protected paths are summarized in
   [Troubleshooting](#troubleshooting). Because cue assets are shared across templates,
   deleting them forces every dependent template to regenerate the cue on its next
   `create_task_tool` call.
3. **Regenerate** — re-run `create_task_tool(template_name="<template-name>")`. The pipeline
   rebuilds the prefab, every segment prefab from scratch, any missing cue prefabs, and the scene.
4. **Spot-check** — run `inspect_prefab_tool` against the rebuilt task prefab to confirm the
   hierarchy matches the template (cue count, segment order, trial zones).

---

## End-to-end task authoring workflow

Use this composite flow for end-to-end task creation; each step is owned by a different skill.

| Step | Skill (owner)                      | Action                                                                                       |
|------|------------------------------------|----------------------------------------------------------------------------------------------|
| 1    | assets `/task-templates`           | Author `Assets/InfiniteCorridorTask/Configurations/<name>.yaml`                              |
| 2    | `/task-prefabs` (this skill)       | `create_task_tool(template_name="<name>")` — builds the task prefab AND the matching scene   |
| 3    | `/task-prefabs` (this skill)       | `inspect_prefab_tool(prefab_path="Assets/InfiniteCorridorTask/Tasks/<name>.prefab")`         |
| 4    | `/task-scenes`                     | `open_scene_tool(scene_path="Assets/Scenes/<name>.unity")` — the scene was created in step 2 |
| 5    | `/scene-setup`                     | Configure Display rig and optional `SimulatedLinearTreadmill`                                |
| 6    | `/play-mode`                       | `enter_play_mode_tool()` → exercise → `exit_play_mode_tool()`                                |
| 7    | assets `/experiment-configuration` | (Optional) Bind the template to a per-project experiment configuration                       |

Checkpoints between steps:

- **Step 2 → 3:** stop if `create_task_tool` returns `success: false`. Common causes are a mistyped
  `template_name`, a malformed YAML header, or an existing scene at the resolved path (regenerate via
  `delete_task_tool` → `create_task_tool`).
- **Step 3 → 4:** stop if `inspect_prefab_tool` shows a hierarchy that contradicts the template (missing
  corridor count, wrong segment names). Fix the template and regenerate via the
  `delete_task_tool` → `create_task_tool` cycle; you MUST NOT hand-patch the prefab.
- **Step 5 → 6:** stop if a display panel is missing — Play Mode without displays throws runtime
  null-reference errors in `ActorObject.Display`.

---

## Reading inspect_prefab_tool output

`inspect_prefab_tool` returns the same recursive node tree that `inspect_scene_tool` does (each
node carries `name`, `position`, `rotation`, `scale`, `components`, optional `collider_*` keys,
optional `children`); the canonical shape contract — and the warning about silently-dropped
missing scripts and key-presence checks — lives in `/task-scenes` "Inspect the active scene".
This section covers only the **task-prefab-specific** interpretation: the top-level object is the
task prefab, its children are `Corridor<indices>` objects, and each first segment's stimulus zone
hierarchy varies by `trigger_type`.

### Lick mode (trigger_type == "lick")

```text
<SegmentName>
└── StimulusTriggerZone              ← collider_size.z = zone width
    │                                  components: ["StimulusTriggerZone", "BoxCollider", "MeshRenderer"]
    └── GuidanceRegion               ← collider_center.z offset = stimulus_location - zone_center
                                       components: ["GuidanceZone", "BoxCollider"]
```

Key markers:
- Root has `StimulusTriggerZone` in `components` and a `BoxCollider` of
  size.z ≈ `(zone_end - zone_start) / cm_per_unit`.
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

### Disarmed segments and ignorable components

`CreateTask` strips zones from segments at corridor depth > 0 because they are visual-only — an
`inspect_prefab_tool` result where every segment except the first under a corridor has no
`StimulusTriggerZone` child is expected, not a bug. When reading the JSON, ignore `Transform`,
`MeshFilter`, `MeshRenderer`, and any `BoxCollider` not paired with a zone script — those are
either visual geometry or standard Unity components every GameObject carries.

---

## Troubleshooting

| Symptom                                                                   | Cause                                                                                                                                                                                                                                                                                                                                                                      | Resolution                                                                                                                                                                                                                                                                                                                                                                                            |
|---------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `create_task_tool` returns "Template not found"                           | Template file missing from `Configurations/`                                                                                                                                                                                                                                                                                                                               | Hand off to assets plugin's `/task-templates`                                                                                                                                                                                                                                                                                                                                                         |
| `create_task_tool` returns "Scene already exists at: …"                   | The target scene exists; regeneration is an explicit two-step action                                                                                                                                                                                                                                                                                                       | Call `delete_task_tool` for the existing scene, then re-run `create_task_tool`                                                                                                                                                                                                                                                                                                                        |
| `create_task_tool` returns "Cross-template cue-texture conflict detected" | Two or more templates in `Configurations/` declare the same `(cue name, length_cm)` identity with different `texture` values — the preflight aborts before any prefab is touched                                                                                                                                                                                           | Rename or re-length the colliding cue in one of the templates, or unify the textures, then re-run. Hand off to assets plugin's `/task-templates` for the YAML edits                                                                                                                                                                                                                                   |
| `inspect_prefab_tool` returns "Prefab not found at: …"                    | Prefab missing from `Tasks/` (deleted, or `create_task_tool` failed silently)                                                                                                                                                                                                                                                                                              | Re-run `create_task_tool` for the template; if the prefab still does not appear, check the Unity Console for `CreateTask` errors                                                                                                                                                                                                                                                                      |
| All Unity tools return "Unity Editor is not reachable"                    | Editor or McpBridge offline                                                                                                                                                                                                                                                                                                                                                | `/unity-mcp-environment-setup` in this plugin                                                                                                                                                                                                                                                                                                                                                         |
| Trigger type mismatch between template and prefab                         | GUID reference drift                                                                                                                                                                                                                                                                                                                                                       | Open the prefab in the Editor and re-link zone                                                                                                                                                                                                                                                                                                                                                        |
| `delete_asset_tool` rejects the path with "Refusing to delete"            | Path is outside the `InfiniteCorridorTask` roots, or names one of the hand-authored protected assets enumerated in `/task-generator` "Required shared assets"                                                                                                                                                                                                              | Reference a different name in the template; you MUST NOT bypass the protection. Restore the protected asset from git if it is genuinely missing                                                                                                                                                                                                                                                       |

---

## Related skills

| Skill                                         | Relationship                                                        |
|-----------------------------------------------|---------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                            |
| `/task-scenes` (this plugin)                  | Consumer — opens / inspects the scene this skill produced           |
| `/play-mode` (this plugin)                    | Consumer — exercises the prefab at runtime                          |
| `/scene-setup` (this plugin)                  | Consumer — configures displays / controller before Play Mode        |
| `/task-parameters` (this plugin)              | Consumer — reads / writes the generated `Task` component fields     |
| `/task-generator` (this plugin)               | Reference for the `CreateTask` pipeline this tool invokes           |
| `/mqtt-contract` (this plugin)                | Reference for MQTT topics wired by generated zone scripts           |
| `/gimbl-framework` (this plugin)              | Reference for `ActorObject` coordinate frame usage                  |
| assets plugin `/task-templates`               | Upstream — owns the YAML template the prefab is built from          |
| assets plugin `/experiment-configuration`     | Downstream — per-project instantiation of the template              |
| assets plugin `/assets-mcp-environment-setup` | Run first — owns the slsa MCP server diagnostic                     |
| experiment plugin `/vr-driver-interface`      | Host consumes the cues and zones in the generated prefab at runtime |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change that calls
`create_task_tool`, `delete_task_tool`, `inspect_prefab_tool`, or `delete_asset_tool`.

```text
Task Prefabs Compliance:
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] slsa mcp server is connected
- [ ] Target template exists under Assets/InfiniteCorridorTask/Configurations/
- [ ] Template filename follows the ProjectAbbreviation_TaskDescription convention
- [ ] create_task_tool returns both prefab_path and scene_path on success
- [ ] inspect_prefab_tool hierarchy matches the template's cue / segment / trial counts
- [ ] Stale cue prefabs and materials are removed via delete_asset_tool before regeneration when
      a cue texture changes without a name or length rename
- [ ] Generated prefabs are not hand-edited; regeneration goes through delete_task_tool →
      create_task_tool
```
