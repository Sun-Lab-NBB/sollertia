---
name: task-generator
description: >-
  Documents the `CreateTask.cs` editor pipeline that builds cue, segment, and task prefabs from
  YAML task templates, the anatomy of the generated cue and segment prefabs, and the placement of
  the hand-authored zone prefabs. Use when modifying `CreateTask.cs`, adding a new zone type,
  hand-authoring a zone prefab, or diagnosing generated-prefab geometry mismatches.
user-invocable: true
---

# Sollertia Unity task generator

Documents the `CreateTask.cs` editor pipeline and the prefab anatomy it assumes — the reference for
**extending** generation rather than invoking it (invocation is owned by `/task-prefabs`).

---

## Scope

**Covers:**
- The `CreateTask.CreateFromTemplate` pipeline (cue synthesis, segment synthesis, task assembly)
- Cue prefab internal layout (`Right`/`Left` quads) generated from templates
- Segment prefab internal layout (cue instances, `Floor`, `Walls`, `ResetZone`, trigger zone)
- Zone placement math (`PlaceLickZone` and `PlaceOccupancyZone`)
- Constraints on adding new zone types, cue shapes, or segment layouts
- The `CreateTask → New Task` Editor menu entry and its relationship to `create_task_tool`

**Does not cover:**
- Invoking prefab generation from MCP (see `/task-prefabs`)
- YAML template schema (see assets plugin `/task-templates`)
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
│   │   ├── trigger_type == "lick"       → PlaceLickZone
│   │   └── trigger_type == "occupancy"  → PlaceOccupancyZone
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
| `Prefabs/StimulusTriggerZone.prefab`  | GameObject | Base prefab for lick-mode zones                                  |
| `Prefabs/OccupancyTriggerZone.prefab` | GameObject | Base prefab for occupancy-mode zones                             |
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

### Lick mode (`PlaceLickZone`)

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

### Occupancy mode (`PlaceOccupancyZone`)

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

- The root is positioned **past** the waiting range at the stimulus boundary. This is intentional: the collider is
  the "tripwire" that fires when occupancy fails.
- `OccupancyGuidanceRegion` sits at the downstream end of the occupancy range, offset by `-0.2` to keep it inside the
  occupancy collider.

### Critical invariant

For lick-mode trials the generator places the zone root at the segment's center such that
`zone_z = zone.transform.localPosition.z` equals `(zone_end + zone_start) / (2 * cm_per_unity_unit)`,
and the `BoxCollider.size.z` equals `(zone_end - zone_start) / cm_per_unity_unit`. For occupancy
mode the generator places the root at `rootZ` (past the waiting range) instead — the root collider
marks the boundary, and the wait region lives on the child `OccupancyRegion`. Anyone auditing a
generated segment via `inspect_prefab_tool` should apply the appropriate formula per `trigger_type`
when comparing the prefab against the template's zone-cm fields.

---

## Corridor assembly

After `BuildSegmentPrefabs` finishes, `CreateTask` builds the top-level task hierarchy:

```text
<TaskName>
│   components: [Task]
│   Task.configPath    = relativeConfigPath (stored for runtime load)
│   Task.requireLick   = true                (default; overridden at runtime by MQTT)
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

1. Add a new `triggerType` enum value in `sollertia-shared-assets` via `/task-templates` and run `/library-extension`
   so the Python registry parity check still passes at import time.
2. Extend the `trigger_type` literal check in `ConfigLoader.ValidateTemplate` (currently accepts `"lick"` and
   `"occupancy"` only); without this, every template that uses the new value fails at load time.
3. Create a new zone base prefab under `Prefabs/<NewZone>TriggerZone.prefab` with the required script and colliders
   (use `/zone-prefabs` to copy and rewrite the closest canonical template).
4. Add the new prefab path to `McpBridge.DeleteProtectedPaths` — `BuildSegmentPrefabs` loads zone prefabs by hardcoded
   path, and an accidental `delete_asset_tool` would break subsequent generation runs.
5. Add a new `if (trial.triggerType == "<new>")` branch in `BuildSegmentPrefabs` and a corresponding `Place<New>Zone`
   helper following the pattern of `PlaceLickZone` / `PlaceOccupancyZone`.
6. Update `/task-prefabs` zone behavior reference with the new trigger type so callers see how to drive it from YAML.

### Adding a new cue or segment

No code changes are needed for cues. To add a new cue texture:

1. Import the `.png` (or compatible image) into `Assets/InfiniteCorridorTask/Textures/`.
2. Reference the filename from the YAML template's `cues[].texture` field together with a unique `name`,
   `code`, and `lengthCm`.
3. Regenerate the task via `create_task_tool`; the cue prefab and matching material are created automatically
   under `Cues/Cue_<name>_<length>cm.prefab` and `Materials/Cue_<name>_<length>cm.mat`.

Segment prefabs are always generated from the template's `trial_structures` block; hand-authoring them is not
supported under the always-regenerate flow because `CleanGeneratedSegments` would delete the hand-authored
prefab on the next generation pass. Express new segment geometry by adding a trial structure to the YAML
template instead.

### Adding a new template-driven field

1. Add the field to the `TaskTemplate` (or nested) class in `sollertia-shared-assets`.
2. Add a getter or conversion helper (e.g. a `*Unity` accessor) to `TaskTemplate.cs` if the field needs unit
   conversion.
3. Thread the field through `CreateFromTemplate` to the relevant sub-step.
4. If the field affects geometry, ordering, or asset count, update the
   Template → prefab field mapping table in `/task-prefabs` so callers can see how the new field surfaces in the 
   generated prefab.

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

| Skill                               | Relationship                                                       |
|-------------------------------------|--------------------------------------------------------------------|
| `/task-prefabs` (this plugin)       | Consumer — invokes `create_task_tool` and validates output         |
| `/mqtt-contract` (this plugin)      | Zone scripts (authored here) own MQTT topics described there       |
| `/gimbl-framework` (this plugin)    | Segment prefabs place GIMBL-derived `Actor` coordinate frame usage |
| assets plugin `/task-templates`     | Upstream — owns YAML authoring and schema evolution                |
| `/csharp-style` (automation plugin) | Enforced when editing `CreateTask.cs` or adding new generator code |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change to `CreateTask.cs`, the zone
base prefabs, the hand-authored shared materials, or the `McpBridge` dispatch surface.

```text
Generator Pipeline Compliance:
- [ ] Any change to zone placement is reflected in both PlaceLickZone and PlaceOccupancyZone if applicable
- [ ] New zone types appear in BuildSegmentPrefabs trigger-type switch AND in the zone base prefab set AND in
      McpBridge.DeleteProtectedPaths
- [ ] Cue prefab regeneration remains shared and skip-if-exists; segment prefab regeneration remains always-rebuilt
      via `CleanGeneratedSegments`
- [ ] Hardcoded asset paths (Prefabs/, Cues/, Materials/, Textures/) are not changed without updating every call site
- [ ] LoadReferenceCueShader still falls back through the documented chain when _CueShaderReference.mat is missing
- [ ] CreateTask → New Task Editor menu and McpBridge.GenerateTask produce identical assets for the same template
- [ ] CreateSceneFromTemplate runs MainWindow.EnsureControllers so the generated scene contains one GameObject
      per ControllerTypes enum value under the "Controllers" root
- [ ] After any generator change, regenerate a representative template via create_task_tool and spot-check the
      prefab via inspect_prefab_tool against the expected hierarchy
```
