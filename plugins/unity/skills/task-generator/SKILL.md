---
name: task-generator
description: >-
  Documents the `CreateTask.cs` editor pipeline that builds cue, segment, and task prefabs from
  YAML task templates, the anatomy of the generated cue and segment prefabs, and the placement of
  the hand-authored zone prefabs. Use when modifying `CreateTask.cs`, adding a new zone type,
  hand-authoring a zone prefab, or diagnosing generated-prefab geometry mismatches.
user-invocable: false
---

# Sollertia Unity task generator

Documents the `CreateTask.cs` editor pipeline and the prefab anatomy it assumes — the reference for
**extending** generation rather than invoking it (invocation is owned by `/task-prefabs`).

**Reference-only skill.** No upstream — agents arrive here on demand from `/task-prefabs` (the
pipeline `create_task_tool` invokes), `/zone-prefabs` (Step 7 wiring), and assets plugin's
`assets:library-extension` (when extending the `TriggerType` enum).

---

## Scope

**Covers:**
- The `CreateTask.CreateFromTemplate` pipeline (cue synthesis, segment synthesis, task assembly)
- Cue prefab internal layout (`Right`/`Left` quads) generated from templates
- Segment prefab internal layout (cue instances, `Floor`, `Walls`, `ResetZone`, trigger zone)
- Zone placement math (`PlaceInteractionZone`, `PlaceCollisionZone`, and `PlaceOccupancyZone`)
- Constraints on adding new zone types, cue shapes, or segment layouts
- The `CreateTask → New Task` Editor menu entry and its relationship to `create_task_tool`

**Does not cover:**
- Invoking prefab generation from MCP (see `/task-prefabs`)
- YAML template schema (see `assets:task-templates`)
- MQTT topic wiring inside zones (see `/mqtt-contract`)
- GIMBL actor / display / controller systems (see `/gimbl-framework`)
- Runtime behavior of generated prefabs (see `Task.cs`; no dedicated skill)

---

## Pipeline architecture

```text
CreateTask.CreateFromTemplate(absoluteTemplatePath, relativeConfigPath, savePath)
│
├── ValidateCueDefinitionsAcrossTemplates    ← Scans every YAML in Configurations/ and aborts before any
│                                              mutation if two templates declare a cue with the same
│                                              (name, length_cm) identity but different textures
│
├── ConfigLoader.LoadTemplate                ← YAML → TaskTemplate
│
├── CleanGeneratedSegments(template)         ← Deletes every Prefabs/<template>_<trial>.prefab the template owns
│
├── BuildCuePrefabs(template)                ← Cues/Cue_<name>_<length>cm.prefab  (shared, skip-if-exists)
│   │
│   ├── LoadReferenceCueShader               ← Reads shader from Materials/_CueShaderReference.mat (canonical)
│   ├── Load/create Materials/Cue_<name>_<length>cm.mat from Textures/<cue.texture>
│   ├── Build Right + Left quads scaled to cue.length_cm / cm_per_unity_unit
│   └── Save Cues/Cue_<name>_<length>cm.prefab
│
├── BuildSegmentPrefabs(template)            ← Prefabs/<template_name>_<trial_name>.prefab  (always rebuilt)
│   │
│   ├── Place cue instances sequentially along +Z
│   ├── Build Floor (plane) and Walls (LeftWall + RightWall quads)
│   ├── For each trial_structure[]:
│   │   ├── trigger_type == "interaction"        → PlaceInteractionZone
│   │   ├── trigger_type == "collision"          → PlaceCollisionZone
│   │   ├── trigger_type == "occupancy_disarm"   → PlaceOccupancyZone (OccupancyDisarm sub-mode)
│   │   ├── trigger_type == "occupancy_arm"      → PlaceOccupancyZone (OccupancyArm sub-mode)
│   │   └── trigger_type == "occupancy_trigger"  → PlaceOccupancyZone (OccupancyTrigger sub-mode)
│   └── Place ResetZone at local Z = cueOffsetUnity (segment root is shifted upstream by the same
│                                                    amount, so the ResetZone lands at world Z = 0,
│                                                    the actor's per-corridor spawn point)
│
└── Assemble task GameObject
    │
    ├── Iterate all (trial_count ^ depth) corridor permutations
    ├── Instantiate segment prefabs along +Z inside each Corridor<indices>
    ├── Strip StimulusTriggerZone and ResetZone from non-first segments in each corridor
    ├── Set StimulusTriggerZone.showBoundary on the first segment from template
    └── PrefabUtility.SaveAsPrefabAsset(task, savePath)


CreateTask.CreateSceneFromTemplate(sceneSavePath, taskPrefabPath, overwriteExisting)
│
├── Copy ExperimentTemplate.unity → sceneSavePath           ← refuses to clobber unless overwriteExisting
├── EditorSceneManager.OpenScene(sceneSavePath)
├── Optionally instantiate the task prefab (non-fatal if missing)
├── MainWindow.EnsureControllers                            ← creates one GameObject per ControllerTypes enum value
│                                                             under the scene's "Controllers" root (no-op if missing)
├── MainWindow.EnsureMqttDefaults                           ← applies EditorPrefs MQTT IP/port with 127.0.0.1:1883 fallback
├── MainWindow.SyncDisplayBrightnessToSettings              ← sets DisplayObject.currentBrightness = settings.brightness
└── Save the new scene + return SceneCreationResult         ← {Success, Message, SimulatedControllerAdded, TaskPrefabNotFound}
```

### Pipeline notes

- **Cross-template cue-texture preflight runs first**: `ValidateCueDefinitionsAcrossTemplates` enumerates every
  `*.yaml` / `*.yml` under `Assets/InfiniteCorridorTask/Configurations/`, loads each through `ConfigLoader`, and
  builds a `(cue name, length label) → list[(texture, template name)]` map. Any identity that resolves to more
  than one distinct texture aborts the entire generation request with a consolidated error before any cue or
  segment is touched. The check exists because cue prefabs and materials are shared filesystem-keyed assets;
  without it, the second template to generate would silently render the wrong texture on the cue prefab the
  first template owns. Templates outside `Configurations/` are invisible to the preflight, which is why both
  the Editor menu and the MCP surface reject them.
- **Cue prefabs are shared across templates**: `BuildCuePrefabs` keys cue assets by `name + length_cm`, so two
  templates that declare an `A` at 30 cm reuse a single `Cue_A_30cm.prefab` / `Cue_A_30cm.mat`. The pass is
  skip-if-exists; an edit to a cue's **texture** without renaming the cue requires deleting the cue prefab and
  material manually before regenerating (see `/task-prefabs` regeneration workflow). When two templates that
  declare the same `(name, length_cm)` identity diverge on texture, the cross-template preflight above catches
  the conflict before generation runs.
- **Cue shader is canonical**: `LoadReferenceCueShader` reads the shader from
  `Assets/InfiniteCorridorTask/Materials/_CueShaderReference.mat` — a hand-authored material protected from
  deletion via `McpBridge.DeleteProtectedPaths`. The shader is Unity's built-in `Legacy Shaders/Diffuse`,
  chosen because it renders both walls of a cue correctly even when the Right wall uses a negative geometry
  scale to mirror its texture; the Standard shader breaks under negative scales and Unlit shaders drop
  lighting altogether. Fallbacks exist (`Cue*.mat` heuristic, `Shader.Find("Legacy Shaders/Diffuse")`,
  `Standard`) but log a warning — the reference material is the canonical source and must be restored from
  git when missing.
- **Segment prefabs are template-owned and always rebuilt**: `CleanGeneratedSegments` deletes every
  `<template>_<trial>.prefab` declared by the template before `BuildSegmentPrefabs` runs, so trial-parameter edits
  in the YAML (cue sequence, zone math, trigger type) take effect on the next generation pass without any manual
  cleanup. Each template's segment prefabs are scoped to that template; no segment prefab is shared across
  templates.
- **The final task prefab** is always overwritten at `savePath`.
- **Validation warning** (not an error): if the measured segment-prefab length disagrees with
  `sum(cue.length_cm / cm_per_unity_unit)` by more than `0.01`, `CreateTask` logs a warning but proceeds using the
  template's computed length.
- **Scene generation is bundled with prefab generation**: `CreateSceneFromTemplate` is the companion method that
  copies `ExperimentTemplate.unity`, instantiates the just-built task prefab, runs `MainWindow.EnsureControllers`,
  and saves the scene. Both the `CreateTask → New Task` Editor menu and the `create_task_tool` MCP surface call
  `CreateFromTemplate` and `CreateSceneFromTemplate` back-to-back from a single template selection, so the manual
  and agentic paths produce byte-equivalent assets.

---

## Cue prefab anatomy

Generated under `Assets/InfiniteCorridorTask/Cues/Cue_<name>_<length>cm.prefab` — the length suffix lets a single
cue name resolve to distinct prefabs when different templates declare different lengths (e.g. `Cue_A_30cm.prefab`
and `Cue_A_50cm.prefab`).

```text
Cue_<name>_<length>cm
├── Right                     localPosition = (0.49, 0.5, lengthUnity/2)
│                             rotation      = (0, 90, 0)
│                             scale         = (-lengthUnity, 1, 1)   ← negative X flips the quad
│                             components    = [MeshFilter, MeshRenderer]
│                             material      = Materials/Cue_<name>_<length>cm.mat
│
└── Left                      localPosition = (-0.49, 0.5, lengthUnity/2)
                              rotation      = (0, -90, 0)
                              scale         = (lengthUnity, 1, 1)
                              components    = [MeshFilter, MeshRenderer]
                              material      = Materials/Cue_<name>_<length>cm.mat
```

- `lengthUnity = cue.length_cm / cm_per_unity_unit`.
- Both quads use the built-in `Quad.fbx` mesh and the shader resolved by `LoadReferenceCueShader`
  (`Legacy Shaders/Diffuse`, sourced from `Materials/_CueShaderReference.mat`).
- No collider, no script — cues are purely visual.
- The material's `_MainTex` is assigned from `Assets/InfiniteCorridorTask/Textures/<cue.texture>`; texture must be
  imported into Unity before generation.

See [Adding a new cue or segment](#adding-a-new-cue-or-segment) for the procedure to add a new cue texture.

---

## Segment prefab anatomy

Generated under `Assets/InfiniteCorridorTask/Prefabs/<template_name>_<trial_name>.prefab`. The filename is derived
directly from the template filename (without extension) and the trial key under `trial_structures`, so each
template owns its segments outright. `ConfigLoader` rejects trial names that contain characters outside
`[A-Za-z0-9_]` so the filesystem layout is always well-formed.

```text
<template_name>_<trial_name>
│   localPosition = (0, 0, -cueOffsetUnity)   ← vr_environment.cue_offset_cm / cm_per_unity_unit shifts the cues upstream
│
├── Cue<CueA>                 localPosition.z = 0
├── Cue<CueB>                 localPosition.z = length(Cue<CueA>)
├── ...                       (one per entry in trial.cue_sequence, placed end-to-end)
│
├── Floor                     localPosition = (0, 0, totalLengthUnity/2)
│                             scale         = (0.1, 1, totalLengthUnity/10)
│                             mesh          = New-Plane.fbx
│                             material      = Materials/Floor.mat
│
├── Walls
│   ├── LeftWall              localPosition = (-0.5, 0.5, totalLengthUnity/2)
│   │                         rotation      = (0, -90, 0)
│   │                         scale         = (totalLengthUnity, 1, 1)
│   │                         material      = Materials/Wall.mat
│   │
│   └── RightWall             localPosition = (0.5, 0.5, totalLengthUnity/2)
│                             rotation      = (0, 90, 0)
│                             scale         = (totalLengthUnity, 1, 1)
│                             material      = Materials/Wall.mat
│
├── <StimulusTriggerZone or OccupancyTriggerZone>   ← only if a trial structure references this segment
│
└── <ResetZone>               localPosition = (0, 0.5, cueOffsetUnity)   ← placed only when a trial structure exists.
                              The segment root carries (0, 0, -cueOffsetUnity), so the ResetZone resolves to world Z = 0.
```

### Required shared assets

`BuildCuePrefabs` and `BuildSegmentPrefabs` abort if any of these are missing. Every entry below
is also in `McpBridge.DeleteProtectedPaths` and cannot be deleted via `delete_asset_tool`:

| Asset                                 | Type       | Purpose                                                          |
|---------------------------------------|------------|------------------------------------------------------------------|
| `Prefabs/StimulusTriggerZone.prefab`  | GameObject | Base prefab for interaction-mode and collision-mode zones        |
| `Prefabs/OccupancyTriggerZone.prefab` | GameObject | Base prefab for all three occupancy-mode zones                   |
| `Prefabs/ResetZone.prefab`            | GameObject | Placed at every segment's start                                  |
| `Prefabs/Padding.prefab`              | GameObject | Appended past every corridor to cap the visible corridor depth   |
| `Materials/_CueShaderReference.mat`   | Material   | Canonical shader source for every generated cue material         |
| `Materials/Floor.mat`                 | Material   | Shared floor material baked into every generated segment         |
| `Materials/Wall.mat`                  | Material   | Shared wall material baked into every generated segment          |
| `Materials/TargetMat.mat`             | Material   | Renderer material referenced by both hand-authored trigger zones |

You MUST NOT rename these assets — `BuildSegmentPrefabs` / `LoadReferenceCueShader` resolve them by
hardcoded path, and the trigger zone prefabs reference `TargetMat.mat` by serialized GUID. The
scene base template `Assets/Scenes/ExperimentTemplate.unity` is also in
`McpBridge.DeleteProtectedPaths` but is consumed by `CreateSceneFromTemplate` rather than by the
prefab build pass.

### Padding prefab

The `vr_environment.padding_prefab_name` template field names the padding prefab under
`Prefabs/`. `CreateTask` appends one padding instance at `depth * min(segmentLengths) - 1` past
every corridor to prevent the camera from seeing past the last real segment. The padding prefab
is **hand-authored** and referenced by name; there is no automatic synthesis. It is in the
required-shared-assets table above and is protected by `McpBridge.DeleteProtectedPaths`.

---

## Zone placement math

`CreateTask` positions zones using the template's cm-valued fields, converted by `cm_per_unity_unit`. All math below
runs per trial structure — a single segment may have at most one `StimulusTriggerZone` or `OccupancyTriggerZone`.

`CreateTask` sets a `TriggerMode` enum field (`Interaction`, `Collision`, `OccupancyDisarm`, `OccupancyArm`,
`OccupancyTrigger`) on the placed `StimulusTriggerZone` directly from `trigger_type`. At runtime the zone dispatches
on this enum and applies the per-mode firing rule.
`PlaceCollisionZone` reuses `StimulusTriggerZone.prefab` (stripping its `GuidanceRegion` child and setting the root
collider as a thin boundary wall at `stimulus_location`). `occupancy_arm` and `occupancy_trigger` reuse
`OccupancyTriggerZone.prefab` through `PlaceOccupancyZone`, which only sets the occupancy sub-mode — no mode has its
own prefab file. The occupancy sub-modes share a single `OccupancyZone.occupancyMet` signal (the generic "occupancy
requirement met" flag); the parent `StimulusTriggerZone` applies the per-mode firing rule.

### Interaction mode (`PlaceInteractionZone`)

```text
zoneStartUnity        = trial.stimulus_trigger_zone_start_cm / cm_per_unity_unit
zoneEndUnity          = trial.stimulus_trigger_zone_end_cm   / cm_per_unity_unit
zoneCenterUnity       = (zoneStartUnity + zoneEndUnity) / 2
zoneSizeUnity         = zoneEndUnity - zoneStartUnity
stimulusLocationUnity = trial.stimulus_location_cm          / cm_per_unity_unit

StimulusTriggerZone.localPosition      = (0, 0.505, zoneCenterUnity)
StimulusTriggerZone.BoxCollider.size   = (1, 1, zoneSizeUnity)
StimulusTriggerZone.BoxCollider.center = (0, 0, 0)

GuidanceRegion.BoxCollider.size   = (1, 1, 0.4)                                ← fixed 0.4 Unity units wide
GuidanceRegion.BoxCollider.center = (0, 0, stimulusLocationUnity - zoneCenterUnity)
```

- Y offset `0.505` is deliberate — raises the zone just above the floor to avoid collider overlap.
- GuidanceRegion width is **not** template-driven (the hardcoded `0.4` unit collider width is a design choice).

### Collision mode (`PlaceCollisionZone`)

```text
stimulusLocationUnity = trial.stimulus_location_cm / cm_per_unity_unit

StimulusTriggerZone.localPosition      = (0, 0.505, stimulusLocationUnity)
StimulusTriggerZone.BoxCollider.size   = (1, 1, thinWallWidth)               ← thin boundary wall at stimulus_location
StimulusTriggerZone.BoxCollider.center = (0, 0, 0)

(GuidanceRegion child stripped — collision mode has no sensor / occupancy region)
```

- Collision mode fires the stimulus **unconditionally** when the actor crosses the invisible boundary wall at
  `stimulus_location` — no sensor, no occupancy. `PlaceCollisionZone` reuses `StimulusTriggerZone.prefab` and strips
  its `GuidanceRegion` child.
- Collision mode keeps the `showStimulusCollisionBoundary` visibility toggle (the per-trial
  `show_stimulus_collision_boundary` template field), surfaced on the first segment in corridor assembly.

### Occupancy modes (`PlaceOccupancyZone`)

`PlaceOccupancyZone` serves all three occupancy sub-modes (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`)
from the same `OccupancyTriggerZone.prefab`; `CreateTask` only sets the occupancy sub-mode on the placed zone. The
geometry below is identical across the three sub-modes — they differ only in the runtime firing rule the parent
`StimulusTriggerZone` applies to the shared `OccupancyZone.occupancyMet` signal:

- **`occupancy_disarm`**: a collision with the boundary fires while occupancy is **not** met (occupancy "disarms" the
  boundary).
- **`occupancy_arm`**: occupying the zone **arms** the boundary; colliding with the now-armed boundary (occupancy
  **met**) fires. It is the inverse of `occupancy_disarm`.
- **`occupancy_trigger`**: occupying the zone for the required duration fires the stimulus **immediately**, with no
  boundary collision.

All three occupancy modes keep the occupancy-guidance brake (`OccupancyGuidanceZone` publishing `Delay`).


```text
zoneStartUnity, zoneEndUnity, zoneCenterUnity, zoneSizeUnity, stimulusLocationUnity   (same derivations)

rootZ = stimulusLocationUnity + zoneSizeUnity / 2
occupancyCenterOffset = zoneCenterUnity - rootZ                                ← negative: occupancy is upstream

StimulusTriggerZone.localPosition      = (0, 0.505, rootZ)
StimulusTriggerZone.BoxCollider.size   = (1, 1, zoneSizeUnity)
StimulusTriggerZone.BoxCollider.center = (0, 0, 0)

OccupancyRegion.BoxCollider.size   = (1, 1, zoneSizeUnity)
OccupancyRegion.BoxCollider.center = (0, 0, occupancyCenterOffset)

OccupancyGuidanceRegion.BoxCollider.size   = (1, 1, 0.4)
OccupancyGuidanceRegion.BoxCollider.center = (0, 0, occupancyCenterOffset + zoneSizeUnity/2 - 0.2)
```

- The root is positioned **past** the waiting range at the stimulus boundary. This boundary collider is the
  "tripwire": for `occupancy_disarm` it fires when occupancy is **not** met, for `occupancy_arm` it fires when
  occupancy **is** met. For `occupancy_trigger` the boundary collider is unused — occupancy met fires immediately.
- `OccupancyGuidanceRegion` sits at the downstream end of the occupancy range, offset by `-0.2` to keep it inside the
  occupancy collider.

### Critical invariant

For interaction-mode trials the generator places the zone root at the segment's center such that
`zone_z = zone.transform.localPosition.z` equals `(zone_end + zone_start) / (2 * cm_per_unity_unit)`,
and the `BoxCollider.size.z` equals `(zone_end - zone_start) / cm_per_unity_unit`. For the three
occupancy modes (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`) the generator places the
root at `rootZ` (past the waiting range) instead — the root collider marks the boundary, and the wait
region lives on the child `OccupancyRegion`. For collision mode the root is a thin boundary wall at
`stimulus_location` with no occupancy region or guidance child. Anyone auditing a generated segment via
`inspect_prefab_tool` should apply the appropriate formula per `trigger_type` when comparing the prefab
against the template's zone-cm fields.

---

## Corridor assembly

After `BuildSegmentPrefabs` finishes, `CreateTask` builds the top-level task hierarchy:

```text
<TaskName>
│   components: [Task]
│   Task.configPath    = relativeConfigPath (stored for runtime load)
│   Task.requireInteraction = true           (default; overridden at runtime by MQTT)
│
├── Corridor000          localPosition = (0, 0, 0)
│   ├── <segment0>       localPosition = (0, 0, 0)
│   ├── <segment0>       localPosition = (0, 0, segmentLengths[0])   ← same segment, re-instantiated
│   ├── ... (segments_per_corridor instances total)
│   └── Padding          localPosition = (0, 0, depth * min(segmentLengths) - 1)
│
├── Corridor001          localPosition = (corridorSpacingUnity, 0, 0)
│   └── ...
│
└── Corridor<N>          (one per permutation of segment indices, segments_per_corridor deep)
```

Rules enforced by the assembly loop:
- Only the **first** segment in each corridor retains its `StimulusTriggerZone` and `ResetZone`. Later segments are
  visual-only and have their zones stripped (`DestroyImmediate`).
- The first segment's `StimulusTriggerZone.showBoundary` is read directly from
  `trials[segment].showStimulusCollisionBoundary` (the per-trial `show_stimulus_collision_boundary` template field).
- Corridor X-spacing is `vr_environment.corridor_spacing_cm / cm_per_unity_unit`.
- The padding Z-shift uses the **minimum** segment length, not the current segment's length. Segments longer than
  the minimum will overlap the padding by design (the overlap is outside the camera's view).

---

## Entry points

### Editor menu

`CreateTask.CreateNewTask` is registered at `CreateTask → New Task` in the Unity menu bar. The menu opens a single
template-selector file dialog seeded at `Assets/InfiniteCorridorTask/Configurations/` **and rejects any selection
outside that directory**: the menu normalizes the chosen path, checks the `Configurations/` prefix, and logs an
error before any mutation when the prefix fails. This matches the MCP surface (which is hard-coded to the same
folder) and ensures the cross-template cue-texture preflight sees every template that can drive generation.

Output paths are **auto-resolved** from the template filename — the prefab lands at
`Assets/InfiniteCorridorTask/Tasks/<template>.prefab` and the scene at `Assets/Scenes/<template>.unity`. The menu
shows a single overwrite-confirmation dialog if either target already exists, auto-creates
`Assets/InfiniteCorridorTask/Tasks/` when the folder is missing, and then runs `CreateFromTemplate` followed by
`CreateSceneFromTemplate` so the same template selection produces both the prefab and a runnable scene in one
pass. The scene step is skipped entirely when `CreateFromTemplate` returns anything other than a `success:`
result, so a failed cue-texture preflight or YAML-load error short-circuits before any scene work begins. When
the prefab step succeeds, the scene step uses Unity's native unsaved-changes dialog before opening; a Cancel
there leaves the already-generated prefab in place for a follow-up run.

### MCP bridge

The `create_task_tool` dispatches to `McpBridge.GenerateTask`, which chains `CreateFromTemplate` and
`CreateSceneFromTemplate` in one call with:
- `absoluteTemplatePath = Application.dataPath/InfiniteCorridorTask/Configurations/<template_name>.yaml`
- `relativeConfigPath = InfiniteCorridorTask/Configurations/<template_name>.yaml` (relative; stored on `Task`)
- `savePath = Assets/InfiniteCorridorTask/Tasks/<template_name>.prefab` (auto-resolved from the template name)
- `sceneSavePath = Assets/Scenes/<template_name>.unity` (auto-resolved from the template name)

The bridge auto-creates `Assets/InfiniteCorridorTask/Tasks/` when missing and refuses to clobber an existing
scene at the resolved path (it calls `CreateSceneFromTemplate` with `overwriteExisting: false`); regeneration
is the two-step `delete_task_tool` → `create_task_tool` cycle. The `SceneCreationResult` fields (`Success`,
`Message`, `SimulatedControllerAdded`, `TaskPrefabNotFound`) are flattened into the JSON response as `message`,
`prefab_path`, `scene_path`, and `simulated_controller_added`.

There is no separate scene-only MCP tool — `create_task_tool` is the single entry point for both prefab and
scene generation, mirroring the Editor menu's single user-driven flow. The companion `delete_task_tool`
removes the scene + per-scene `savedFullScreenViews` companion + task prefab + every owned segment prefab
atomically.

All entry points converge on `CreateFromTemplate` and `CreateSceneFromTemplate`, so any change to either method
affects both flows. Test through both the menu and the MCP tools after any pipeline modification.

---

## Extending the pipeline

### Adding a new zone trigger type

**Recipe boundary.** A new trigger mode is agent-doable even when its firing behavior is genuinely novel — the
`/zone-prefabs` worked examples cover a speed-gated interaction reward and a cumulative-occupancy variant end to end.
The recipe holds as long as the new mode is a zone modifier (subclass an existing zone, or a standalone `IResettable`
registered in `ResetZone`) on a copied zone prefab whose root subclasses `StimulusTriggerZone` and publishes the
standard `Stimulus{trialName}` event. Escalate to the human supervisor only when the behavior needs a new MQTT topic,
new `Task.cs` runtime mechanics, or geometry outside a single corridor segment — those are paradigm-level and have no
author-derived recipe.

This skill owns the **`CreateTask` pipeline edits** for a new `TriggerType`. The full cross-cutting
recipe is split three ways:

| Slice                                        | Owning skill                                  |
|----------------------------------------------|-----------------------------------------------|
| Python registry + `TriggerType` enum         | `assets:library-extension`            |
| Hand-authored zone prefab manufacturing      | `/zone-prefabs`                               |
| `CreateTask` pipeline edits                  | this skill (steps 1–3 below)                  |

Apply your three skills' bullets in order. The pipeline-side touches owned here:

1. Extend the `trigger_type` literal check in `ConfigLoader.ValidateTemplate` (currently accepts
   `"interaction"`, `"collision"`, `"occupancy_disarm"`, `"occupancy_arm"`, and `"occupancy_trigger"`); without
   this, every template that uses the new value fails at load time. Validation is mode-aware: `collision` validates
   only `stimulus_location` (no trigger zone); `occupancy_trigger` validates only the trigger zone (no boundary);
   `interaction`, `occupancy_disarm`, and `occupancy_arm` validate the zone, the boundary, and their ordering.
2. Add a new `if (trial.triggerType == "<new>")` branch in `BuildSegmentPrefabs` and a
   corresponding `Place<New>Zone` helper following the pattern of `PlaceInteractionZone` /
   `PlaceCollisionZone` / `PlaceOccupancyZone`. Add a matching `TriggerMode` enum member on
   `StimulusTriggerZone` and set it from `trigger_type` when a new mode needs a distinct runtime firing rule; reuse
   an existing base prefab where possible (the five current modes add **no** new prefab files — `collision` reuses
   `StimulusTriggerZone.prefab`, and `occupancy_arm` / `occupancy_trigger` reuse `OccupancyTriggerZone.prefab`).
3. Add the new prefab path to `McpBridge.DeleteProtectedPaths` — `BuildSegmentPrefabs` loads zone
   prefabs by hardcoded path, and an accidental `delete_asset_tool` would break subsequent
   generation runs.

Coordinate the prefab manufacturing through `/zone-prefabs` Step 7 and the Python registry parity
through assets `assets:library-extension` "Adding a new `TriggerType` member" so each skill bullets only
its own substeps.

The platform `TriggerType` enum carries all five members (`INTERACTION`, `COLLISION`, `OCCUPANCY_DISARM`,
`OCCUPANCY_ARM`, `OCCUPANCY_TRIGGER`); the C# `ConfigLoader` accepts all five literals. System support is a
**per-system subset**: a new `TriggerType` member does **not** require a `from_task_template` branch — each
acquisition system maps only the subset it supports and may leave a member unmapped. The Mesoscope-VR system's
`from_task_template` maps `INTERACTION` (→ `MesoscopeWaterRewardTrial`) and `OCCUPANCY_DISARM`
(→ `MesoscopeGasPuffTrial`), and does not map `collision`, `occupancy_arm`, or `occupancy_trigger`, so a
Mesoscope-VR config that uses one of those raises a clear "not mapped to a runtime trial class" error.
All five modes share one MQTT/wire contract:
every mode publishes the same `Stimulus{trialName}` event, adds no topics, and does not change
`require_interaction` / `require_wait`. `list_supported_trigger_types_tool` returns all five values.

### Adding a new cue or segment

No code changes are needed for cues, but **importing a new cue texture is a human hand-off** — you cannot author PNG
or other binary image assets. To add a new cue texture:

1. **Hand the texture off to the user.** If the referenced image is not already under
   `Assets/InfiniteCorridorTask/Textures/`, stop and ask the user to supply it — state the intended cue `name`,
   `code`, `length_cm`, and target filename. The user imports the `.png` (or compatible image) and then loops you back
   to continue. You MUST NOT let generation dead-end in a `Failed to load texture` error.
2. Reference the filename from the YAML template's `cues[].texture` field together with a unique `name`,
   `code`, and `lengthCm`.
3. Regenerate the task via `create_task_tool`; the cue prefab and matching material are created automatically
   under `Cues/Cue_<name>_<length>cm.prefab` and `Materials/Cue_<name>_<length>cm.mat`.

Segment prefabs are always generated from the template's `trial_structures` block; hand-authoring them is not
supported under the always-regenerate flow because `CleanGeneratedSegments` would delete the hand-authored
prefab on the next generation pass. Express new segment geometry by adding a trial structure to the YAML
template instead.

A template field is a **two-repo mirror** — the YAML deserializer maps each underscored YAML key to the camelCase C#
member, so the two class definitions must stay in lockstep or the field is silently dropped (or fails to parse) at
`create_task` time, far from the edit that caused it, in the other repo. There is no automated parity check; this is a
manual, verify-before-done step.

1. Add the field to the `TaskTemplate` (or nested) Python dataclass in `sollertia-shared-assets`, **and** add the
   matching `[Serializable]` field to the mirror C# class (`TaskTemplate.cs` / `Cue.cs` / `TrialStructure.cs` /
   `VREnvironment.cs`) in this repo. The C# member name MUST be the camelCase counterpart of the underscored YAML key
   (e.g. `cue_offset_cm` → `cueOffsetCm`), and its optionality / default MUST match the Python side.
2. Add a getter or conversion helper (e.g. a `*Unity` accessor) to `TaskTemplate.cs` if the field needs unit
   conversion.
3. Thread the field through `CreateFromTemplate` to the relevant sub-step.
4. If the field affects geometry, ordering, or asset count, update the
   [Cue prefab anatomy](#cue-prefab-anatomy), [Segment prefab anatomy](#segment-prefab-anatomy),
   or [Zone placement math](#zone-placement-math) section above so callers can see how the new
   field surfaces in the generated prefab.
5. **Verify the round-trip**: author a template that sets the new field, run `create_task_tool`, and confirm via
   `inspect_prefab_tool` (or the field's downstream effect) that the value actually arrived on the C# side — a missing
   or mistyped mirror field surfaces here as a dropped value, not a compile error.

---

## Failure modes

| Symptom                                                                           | Root cause                                                                                             | Resolution                                                                                                                                                        |
|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CreateFromTemplate` error: `Cross-template cue-texture conflict detected`        | Two templates under `Configurations/` declare the same `(cue name, length_cm)` with different textures | Reconcile the offending templates (rename the cue, change its length, or unify the textures), then re-run; the preflight aborts before any cue/segment is touched |
| `BuildCuePrefabs` error: `Failed to load texture`                                 | Cue's `texture` field references a file not in `Textures/`                                             | Import the texture, retry generation                                                                                                                              |
| `BuildSegmentPrefabs` error: `Missing Floor.mat`                                  | Shared materials deleted or renamed                                                                    | Restore from git                                                                                                                                                  |
| Segment length warning in Console                                                 | Cue lengths do not sum to measured prefab length                                                       | Either regenerate the segment or fix template cues                                                                                                                |
| `create_task_tool` error: `No segment found`                                      | `BuildSegmentPrefabs` errored: missing cue prefab, `Floor.mat`, or `Wall.mat`                          | Check Console for `BuildSegmentPrefabs:` errors; verify Floor/Wall materials and referenced cues (trigger-zone prefabs are non-fatal; zones are simply skipped)   |
| Zone geometry looks wrong in scene view                                           | Template's cm values or `cm_per_unity_unit` mismatch                                                   | Recheck YAML; regenerate                                                                                                                                          |
| Cue textures appear mirrored on Left or Right wall                                | Quad scale sign is flipped (Right uses negative X)                                                     | Intentional — each wall shows a correctly-oriented cue                                                                                                            |

---

## Related skills

| Skill                                    | Relationship                                                       |
|------------------------------------------|--------------------------------------------------------------------|
| `/task-prefabs` (this plugin)            | Consumer — invokes `create_task_tool` and validates output         |
| `/mqtt-contract` (this plugin)           | Zone scripts (authored here) own MQTT topics described there       |
| `/gimbl-framework` (this plugin)         | Segment prefabs place GIMBL-derived `Actor` coordinate frame usage |
| `assets:task-templates`          | Upstream — owns YAML authoring and schema evolution                |
| `ataraxis@automation:csharp-style`      | Enforced when editing `CreateTask.cs` or adding new generator code |
| `experiment:vr-driver-interface` | Host decomposes the cue sequence these generated prefabs render    |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change to `CreateTask.cs`, the zone
base prefabs, the hand-authored shared materials, or the `McpBridge` dispatch surface.

```text
Generator Pipeline Compliance:
- [ ] Any change to zone placement is reflected in PlaceInteractionZone, PlaceCollisionZone, and PlaceOccupancyZone
      if applicable
- [ ] New zone types appear in the BuildSegmentPrefabs trigger-type switch (all five literals: interaction,
      collision, occupancy_disarm, occupancy_arm, occupancy_trigger), set the StimulusTriggerZone.TriggerMode enum,
      and (if a new base prefab is added) appear in the zone base prefab set AND in McpBridge.DeleteProtectedPaths
- [ ] Cue prefab regeneration remains shared and skip-if-exists; segment prefab regeneration remains always-rebuilt
      via `CleanGeneratedSegments`
- [ ] Hardcoded asset paths (Prefabs/, Cues/, Materials/, Textures/) are not changed without updating every call site
- [ ] LoadReferenceCueShader still falls back through the documented chain when _CueShaderReference.mat is missing
- [ ] CreateTask → New Task Editor menu and McpBridge.GenerateTask produce identical assets for the same template
- [ ] CreateSceneFromTemplate runs MainWindow.EnsureControllers so the generated scene contains one GameObject
      per ControllerTypes enum value under the "Controllers" root
- [ ] After any generator change, regenerate a representative template via create_task_tool and spot-check the
      prefab via inspect_prefab_tool against the expected hierarchy
- [ ] Any new template field has a matching [Serializable] C# mirror field (camelCase of the underscored YAML key,
      with matching optionality/default) in TaskTemplate.cs / Cue.cs / TrialStructure.cs / VREnvironment.cs, verified
      by a create_task_tool + inspect_prefab_tool round-trip — the two-repo schema mirror has no automated parity check
```
