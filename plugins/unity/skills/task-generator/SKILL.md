---
name: task-generator
description: >-
  Documents the `CreateTask.cs` editor pipeline that builds cue, segment, and task prefabs from YAML task templates.
  Covers its preflight validators, the anatomy of the generated prefabs, the placement of the hand-authored zone
  prefabs, and the runtime contract a generated task must satisfy. Use when modifying `CreateTask.cs`, adding a new zone
  type, hand-authoring a zone prefab, or diagnosing generated-prefab geometry mismatches.
user-invocable: false
---

# Sollertia Unity task generator

Documents the `CreateTask.cs` editor pipeline and the prefab anatomy it assumes. It is the reference for **extending**
generation rather than invoking it, and `/task-prefabs` owns invocation.

**Reference-only skill.** No upstream. Agents arrive here on demand from `/task-prefabs` (the pipeline
`create_task_tool` invokes), `/zone-prefabs` (Step 7 wiring), and assets plugin's `assets:library-extension` (when
extending the `TriggerType` enum).

Deep-reference material lives beside this file:
- [references/prefab-anatomy.md](references/prefab-anatomy.md) covers the cue, segment, and corridor hierarchy trees
  plus the `BuildCuePrefabs` internals.
- [references/zone-placement.md](references/zone-placement.md) covers the per-mode zone placement geometry.
- [references/template-validation.md](references/template-validation.md) covers the `ConfigLoader` validation surface
  and the two-repo template-field inventory.

---

## Scope

**Covers:**
- The `CreateTask.CreateFromTemplate` pipeline (preflight validation, cue and segment synthesis, task assembly)
- The `ConfigLoader` template-validation surface and the two-repo template-field mirror
- Cue prefab internal layout (`Right`/`Left` quads) and segment prefab internal layout (cue instances, `Floor`, `Walls`,
  trigger zone)
- Zone placement math (`PlaceInteractionZone`, `PlaceCollisionZone`, and `PlaceOccupancyZone`)
- The `Task.cs` startup contract a generated task prefab must satisfy to stay enabled
- Constraints on adding new zone types, cue shapes, or segment layouts
- The `CreateTask → New Task` Editor menu entry and its relationship to `create_task_tool`

**Does not cover:**
- Invoking prefab generation from MCP (see `/task-prefabs`)
- YAML template schema (see `assets:task-templates`)
- MQTT topic wiring inside zones (see `/mqtt-contract`)
- GIMBL actor / display / controller systems (see `/gimbl-framework`)
- Per-zone runtime firing rules and zone authoring (see `/zone-prefabs`)
- The Unity test suite and the assembly-definition layout (see `/unity-tests`)

---

## Namespaces and assemblies

The template data model lives in `namespace SL.Config`, covering `TaskTemplate`, `Cue`, `TrialStructure`,
`VREnvironment`, and `ConfigLoader`. The runtime and generator types live in `namespace SL.Tasks`, covering `Task`,
`StimulusTriggerZone`, `GuidanceZone`, `OccupancyZone`, `OccupancyGuidanceZone`, `IResettable`, `TriggerMode`,
`Utility`, `CreateTask`, and `McpBridge`. Runtime scripts under `Assets/InfiniteCorridorTask/Scripts/` compile into
`Sollertia.InfiniteCorridorTask`, editor scripts under `Scripts/Editor/` compile into
`Sollertia.InfiniteCorridorTask.Editor`, and the GIMBL layer compiles into `Sollertia.Gimbl` / `Sollertia.Gimbl.Editor`.
A new `Place<New>Zone` helper therefore joins `SL.Tasks` in the Editor assembly, while a new template field joins
`SL.Config` in the runtime assembly. See `/unity-tests` for the full assembly inventory.

---

## Pipeline architecture

```text
CreateTask.CreateFromTemplate(absoluteTemplatePath, relativeConfigPath, savePath)
│
├── ValidateCueDefinitionsAcrossTemplates    ← Scans every YAML in Configurations/ and aborts before any
│                                              mutation if two templates declare a cue with the same
│                                              (name, length_cm) identity but different textures
│
├── ConfigLoader.LoadTemplate                ← YAML → TaskTemplate; rejects a template filename outside
│                                              [A-Za-z0-9_]+ and runs the full ValidateTemplate surface
│
├── ValidateTrackLengthCoversCorridor        ← Rejects a non-positive longest segment, or one so long that
│                                              Task.DefaultTrackLength cannot fill segments_per_corridor
│
├── ValidateHandAuthoredAssets               ← Rejects the request, naming every missing path at once, when
│                                              Floor.mat, Wall.mat, either zone base prefab, the template's
│                                              padding prefab, or _CueShaderReference.mat is absent — before
│                                              any asset is written
│
├── BuildCuePrefabs(template)                ← Cues/Cue_<name>_<length>cm.prefab  (shared, skip-if-exists)
│   ├── LoadReferenceCueShader               ← Once per pass, before the cue loop; reads the shader from
│   │                                          Materials/_CueShaderReference.mat (canonical)
│   │
│   ├── Load Textures/<cue.texture> as a Texture2D  (aborts when it is not imported)
│   ├── Abort if the cached Cue_<name>_<length>cm.mat was built from a different texture
│   └── Load/create the .mat, build Right + Left quads, save Cues/Cue_<name>_<length>cm.prefab
│
├── CleanGeneratedSegments(template)         ← Deletes, by exact canonical name, every
│                                              Prefabs/<TemplateName>-<TrialName>.prefab the template owns
│
├── BuildSegmentPrefabs(template)            ← Prefabs/<TemplateName>-<TrialName>.prefab  (always rebuilt)
│   │
│   ├── Place cue instances sequentially along +Z
│   ├── Build Floor (plane) and Walls (LeftWall + RightWall quads)
│   └── For each trial_structure[] — exactly one zone per segment:
│       ├── trigger_type == "interaction"        → PlaceInteractionZone
│       ├── trigger_type == "collision"          → PlaceCollisionZone
│       ├── trigger_type == "occupancy_disarm"   → PlaceOccupancyZone (OccupancyDisarm sub-mode)
│       ├── trigger_type == "occupancy_arm"      → PlaceOccupancyZone (OccupancyArm sub-mode)
│       ├── trigger_type == "occupancy_trigger"  → PlaceOccupancyZone (OccupancyTrigger sub-mode)
│       └── anything else                        → abort; no zoneless segment is ever saved
│
└── Assemble task GameObject
    │
    ├── Resolve Prefabs/<padding_prefab_name>.prefab and load every segment prefab back
    ├── Iterate all (trial_count ^ segments_per_corridor) corridor permutations
    ├── Instantiate segment prefabs along +Z inside each Corridor<indices>
    ├── Strip the StimulusTriggerZone GameObject from non-first segments in each corridor
    ├── ApplyBoundaryVisibility on each corridor's first segment from its trial's declaration
    └── Append Padding at this corridor's own zShift - cueOffsetUnity, then SaveAsPrefabAsset


CreateTask.CreateSceneFromTemplate(sceneSavePath, taskPrefabPath, overwriteExisting)
│
├── Copy ExperimentTemplate.unity → sceneSavePath           ← refuses to clobber unless overwriteExisting
├── EditorSceneManager.OpenScene(sceneSavePath)
├── Optionally instantiate the task prefab (non-fatal if missing)
├── MainWindow.EnsureControllers                            ← one GameObject per ControllerTypes enum value under
│                                                             the scene's "Controllers" root (no-op if missing)
├── MainWindow.EnsureMqttDefaults                           ← EditorPrefs MQTT IP/port, 127.0.0.1:1883 fallback
├── MainWindow.SyncDisplayBrightnessToSettings              ← sets DisplayObject.currentBrightness = settings.brightness
├── MainWindow.RemoveDefaultMainCamera                      ← destroys every Camera tagged MainCamera or named
│                                                             "Main Camera", including inactive ones
└── Save the new scene + return SceneCreationResult         ← {Success, Message, SimulatedControllerAdded, TaskPrefabNotFound}
```

### Pipeline notes

- **Generation fails loudly, never silently.** All three preflight validators run before any asset is written, and the
  cue and segment builds abort on a missing input rather than degrading. Only three soft failures survive: the cue
  shader fallback (reachable only for a reference material that loads with no shader, because a missing one aborts at
  the preflight), the segment-length mismatch warning, and the non-fatal missing-task-prefab path.
- **Cross-template cue-texture preflight runs first**: `ValidateCueDefinitionsAcrossTemplates` enumerates every `*.yaml`
  / `*.yml` under `Assets/InfiniteCorridorTask/Configurations/`, loads each through `ConfigLoader`, and builds a `(cue
  name, length label) → list[(texture, template name)]` map. Any identity that resolves to more than one distinct
  texture aborts the request before any cue or segment is touched. The check exists because cue prefabs and materials
  are shared filesystem-keyed assets. Without it, the second template to generate would silently render the wrong
  texture on the cue prefab the first template owns. Because the preflight loads **every** template in the catalog, one
  template that fails to load blocks generation of all of them. Templates outside `Configurations/` are invisible to it,
  which is why both the Editor menu and the MCP surface reject them.
- **`ValidateTrackLengthCoversCorridor` is a generation-time template constraint.** It rejects a non-positive longest
  segment, and a longest segment for which `floor(15000 / max_segment_length_unity) < segments_per_corridor`, where
  `15000` is `Task.DefaultTrackLength` and `max_segment_length_unity` is the largest per-trial `sum(cue.length_cm) /
  cm_per_unity_unit`. Without it, generation would succeed and the task would disable itself on the first Play Mode
  entry. Resolve it template-side only, by shortening the longest `cue_sequence`, lowering `segments_per_corridor`, or
  raising `cm_per_unity_unit`. Raising Track Length cannot clear this gate, because the check divides the compile-time
  constant `Task.DefaultTrackLength` rather than the serialized `Task.trackLength` that `/task-parameters` writes, and
  the refusal returns before any scene exists to edit. The C# message's own "raise Track Length in Window > Task
  Parameters before generating" tail is misleading for that reason.
- **`ValidateHandAuthoredAssets` is the single missing-asset gate.** It checks `Materials/Floor.mat`,
  `Materials/Wall.mat`, both zone base prefabs, `Prefabs/<padding_prefab_name>.prefab`, and
  `Materials/_CueShaderReference.mat`, reporting every missing path at once under `Unable to generate the task. Every
  hand-authored asset the pipeline references must exist, but these are missing from the project:`, and runs before
  `BuildCuePrefabs` and `CleanGeneratedSegments`.
- **Cues are built before the segment wipe.** `BuildCuePrefabs` precedes `CleanGeneratedSegments` because a missing
  texture or a conflicting cached material aborts there, and a wipe that ran first would strand the existing task prefab
  and scene on segments the aborted call is unable to rebuild.
- **Cue prefabs are shared across templates**: `BuildCuePrefabs` keys cue assets by `name + length_cm`, so two templates
  that declare an `A` at 30 cm reuse a single `Cue_A_30cm.prefab` / `Cue_A_30cm.mat`. The skip-if-exists test requires
  **both** assets to survive, so a material deleted from under a surviving prefab causes the prefab to be deleted and
  rebuilt with its renderers pointing at the new material. An edit to a cue's **texture** without renaming the cue is
  caught rather than rendered. A cached material whose `_MainTex` differs from the declared texture aborts the build and
  names the two resolutions, which are deleting both cue assets or re-keying the cue by name or length. Full internals
  live in [references/prefab-anatomy.md](references/prefab-anatomy.md).
- **Segment prefabs are template-owned and always rebuilt**: `CleanGeneratedSegments` deletes every
  `<TemplateName>-<TrialName>.prefab` declared by the template before `BuildSegmentPrefabs` runs, so trial-parameter
  edits in the YAML take effect on the next generation pass without any manual cleanup. Deletion is by exact canonical
  name, which the hyphen separator makes unambiguous even where one template basename nests another. No segment prefab
  is shared across templates.
- **The final task prefab** is always overwritten at `savePath`.
- **Validation warning** (not an error): a measured segment-prefab length that disagrees with `sum(cue.length_cm /
  cm_per_unity_unit)` by more than `0.01` warns but proceeds on the template's computed length.
- **Scene generation is bundled with prefab generation**: `CreateSceneFromTemplate` copies `ExperimentTemplate.unity`,
  instantiates the just-built task prefab, runs `MainWindow.EnsureControllers`, `MainWindow.EnsureMqttDefaults`,
  `MainWindow.SyncDisplayBrightnessToSettings`, and `MainWindow.RemoveDefaultMainCamera`, then saves the scene. The
  camera step leaves the generated scene free of the template's default camera without a human opening Window > Task
  Parameters, the window whose `OnEnable` otherwise runs that removal through `MainWindow.InitializeScene`. Both the
  `CreateTask → New Task` Editor menu and `create_task_tool` call `CreateFromTemplate` and `CreateSceneFromTemplate`
  back-to-back from one template selection, so the manual and agentic paths produce byte-equivalent assets.

---

## Required shared assets

Every entry below is in `McpBridge.DeleteProtectedPaths` and cannot be deleted via `delete_asset_tool`:

| Asset                                 | Type       | Purpose                                                          |
|---------------------------------------|------------|------------------------------------------------------------------|
| `Prefabs/StimulusTriggerZone.prefab`  | GameObject | Base prefab for interaction-mode and collision-mode zones        |
| `Prefabs/OccupancyTriggerZone.prefab` | GameObject | Base prefab for all three occupancy-mode zones                   |
| `Prefabs/Padding.prefab`              | GameObject | Appended past every corridor to cap the visible corridor depth   |
| `Materials/_CueShaderReference.mat`   | Material   | Canonical shader source for every generated cue material         |
| `Materials/Floor.mat`                 | Material   | Shared floor material baked into every generated segment         |
| `Materials/Wall.mat`                  | Material   | Shared wall material baked into every generated segment          |
| `Materials/TargetMat.mat`             | Material   | Renderer material referenced by both hand-authored trigger zones |

A missing or broken entry is handled at one of three severities:

- **Aborts before any mutation.** `ValidateHandAuthoredAssets` covers `Floor.mat`, `Wall.mat`, both zone prefabs, the
  template's padding prefab, and `_CueShaderReference.mat`, naming every missing path at once under `Unable to generate
  the task. Every hand-authored asset the pipeline references must exist, but these are missing from the project:`.
  `BuildSegmentPrefabs` keeps its own null guards (`Unable to build the segment prefabs. Floor.mat and Wall.mat must
  both exist under <folder>, but at least one of them is missing.` and `Unable to build the segment prefabs.
  StimulusTriggerZone.prefab and OccupancyTriggerZone.prefab must both exist under <folder>, but at least one of them
  is missing. Restore both hand-authored zone prefabs before generating.`) and `CreateFromTemplate` keeps its late
  `error: Unable to assemble the corridor. The padding prefab must exist at '<path>', but it is missing.` check, both
  as defense for direct callers and for a deletion racing the build. Every segment that reports success carries a
  trigger zone.
- **Aborts inside `BuildSegmentPrefabs`.** Beyond the table: a generated cue prefab that a trial's `cue_sequence`
  references but that is missing from `Cues/` ends the run with `Unable to build the segment prefab for trial '<name>'.
  The cue prefab must exist at '<path>', but it is missing.`
- **Degrades with a warning.** `LoadReferenceCueShader` logs a `Debug.LogWarning` (`Unable to load the canonical cue
  shader reference. The material at '<path>' must exist, but it is missing, so the shader falls back to a
  hand-authored Cue*.mat material or Shader.Find.`) and resolves the shader through the fallback chain documented in
  [references/prefab-anatomy.md](references/prefab-anatomy.md). Only a `Materials/_CueShaderReference.mat` that loads
  with a null shader reaches it from `CreateFromTemplate`, because a missing file is already refused by
  `ValidateHandAuthoredAssets`.

`Materials/TargetMat.mat` sits outside all three paths. The generator never loads it by path, and the table lists it
only because `DeleteProtectedPaths` protects it.

You MUST NOT rename these assets, because `BuildSegmentPrefabs` / `LoadReferenceCueShader` /
`ValidateHandAuthoredAssets` resolve them by hardcoded path, and the trigger zone prefabs bind `TargetMat.mat` by
serialized GUID. The scene base template `Assets/Scenes/ExperimentTemplate.unity` is also in
`McpBridge.DeleteProtectedPaths` but is consumed by `CreateSceneFromTemplate` rather than by the prefab build pass.
`Prefabs/Padding.prefab` is not synthesized either. It is hand-authored from Unity built-in primitives, a `Floor` plane
plus `Walls/LeftWall` and `Walls/RightWall` quads mirroring the generated segment layout, and its three renderers bind
the same shared `Materials/Floor.mat` and `Materials/Wall.mat` that `BuildSegmentPrefabs` bakes into every generated
segment. A material edit is therefore corridor-wide, only the geometry is padding-local, and a corridor-cap geometry
change is a prefab edit rather than a code change.

---

## Runtime contract

Generation succeeding is not the same as the task running. `Task.Start` re-loads the template from the `configPath`
baked into the prefab and disables itself rather than running on an inconsistent configuration. Every bailout below sets
`enabled = false` after logging a `Debug.LogError`, so the symptom is a Play Mode session in which nothing moves and no
stimulus publishes. `read_console_tool` retrieves that error, which names the bailout that fired. A change to
`CreateTask` MUST keep all of them unreachable for a fresh task.

- **Corridor-count ceiling.** `Task` allocates one corridor-map entry per permutation, so `len(trial_structures) **
  vr_environment.segments_per_corridor` must stay at or below `Task`'s private `MaximumCorridorCount` constant
  (`268435456` = 2^28, the point at which the map's `(float, float)` array reaches the two-gigabyte bound on a single
  managed array). Exceeding that ceiling logs `Unable to build the corridor map for template '<name>', which declares
  <n> trials over a corridor depth of <d>. The corridor combination count must stay at or below the 268435456 entries
  the map holds, but that combination needs more.` There is no generation-time counterpart, because `CreateTask`
  assembles the prefab first.
- **`configPath` must resolve.** An empty `configPath`, or one that does not resolve to a file under
  `Application.dataPath`, logs `Unable to load the task configuration. The configPath must resolve to an existing YAML
  file, but configPath='<p>' resolved to '<r>', which names no file.` The path is stored project-relative
  (`InfiniteCorridorTask/Configurations/<template>.yaml`) with leading separators stripped. This bailout fires for a
  task prefab generated from a template outside `Assets/`. A template that no longer loads fails the same way through
  the adjacent `Unable to load the task template from YAML file '<path>'. The file must hold a valid template, but
  deserialization failed with: <msg>.` bailout.
- **Track length must cover the corridor depth.** When maze generation yields fewer segments than
  `segments_per_corridor`, `Task` logs `Unable to start the task for template '<name>'. Maze generation must produce at
  least <d> segments to fill one corridor, but trackLength <n> produced <m>. The shortest segment measures <l> Unity
  units. Raise Track Length in Window > Task Parameters.` `ValidateTrackLengthCoversCorridor` is the generation-time
  gate that keeps this unreachable at the default track length, and a track length lowered by hand via
  `/task-parameters` can still trip it.
- **Sequence exhaustion ends the session.** When the animal runs past the last generated segment, `Task.Update` logs
  `Unable to advance to the next corridor. The generated segment sequence must extend past the current segment index,
  but the animal ran through every generated segment. Raise Track Length in Window > Task Parameters to cover a longer
  run.` and disables itself mid-run. This is a track-length budgeting concern, not a generation defect.
- **Corridor key out of range.** When the encoded corridor key falls outside the corridor map, `Task.Update` logs
  `Unable to read the current corridor. The corridor key must fall within [0, <n>), but it is <k>. The key stays out of
  range for every later frame, so the Task is disabled to prevent runtime errors.` and disables itself. A correctly
  assembled prefab keeps the key in range for every permutation, so this fires only on a hand-edited corridor map or a
  mismatched trial count.

Per-lap zone reset is driven from the same corridor advance rather than from any in-segment trigger volume.
`Task.ResetZoneStates()` walks the `IResettable[]` that `Task.FindResettableZones()` collected at `Start`.
`FindResettableZones` enumerates the four concrete implementers by type, which are `StimulusTriggerZone`,
`GuidanceZone`, `OccupancyZone`, and `OccupancyGuidanceZone`, because Unity's typed find helpers resolve components
rather than interfaces. **A new standalone `IResettable` implementer needs its own `FindObjectsByType` line there**, or
it silently never resets. `/zone-prefabs` owns the zone-authoring side of that contract.

---

## Entry points

### Editor menu

`CreateTask.CreateNewTask` is registered at `CreateTask → New Task` in the Unity menu bar. The menu opens a single
template-selector file dialog seeded at `Assets/InfiniteCorridorTask/Configurations/` **and rejects any selection
outside that directory**: it normalizes the chosen path, checks the `Configurations/` prefix, and logs an error before
any mutation when the prefix fails. This matches the MCP surface (hard-coded to the same folder) and ensures the
cross-template cue-texture preflight sees every template that can drive generation.

Output paths are **auto-resolved** from the template filename, so the prefab lands at
`Assets/InfiniteCorridorTask/Tasks/<template>.prefab` and the scene at `Assets/Scenes/<template>.unity`. The menu shows
a single overwrite-confirmation dialog if either target already exists, auto-creates
`Assets/InfiniteCorridorTask/Tasks/` when the folder is missing, and then runs `CreateFromTemplate` followed by
`CreateSceneFromTemplate` so one template selection produces both the prefab and a runnable scene. The scene step is
skipped entirely when `CreateFromTemplate` returns anything other than a `success:` result, so a failed preflight or
YAML-load error short-circuits before any scene work begins. When the prefab step succeeds, the scene step uses Unity's
native unsaved-changes dialog before opening, and a Cancel there leaves the already-generated prefab in place for a
follow-up run.

### MCP bridge

The `create_task_tool` dispatches to `McpBridge.GenerateTask`, which chains `CreateFromTemplate` and
`CreateSceneFromTemplate` in one call with:
- `absoluteTemplatePath = Application.dataPath/InfiniteCorridorTask/Configurations/<template_name>.yaml`
- `relativeConfigPath = InfiniteCorridorTask/Configurations/<template_name>.yaml` (relative, stored on `Task`)
- `savePath = Assets/InfiniteCorridorTask/Tasks/<template_name>.prefab` (auto-resolved from the template name)
- `sceneSavePath = Assets/Scenes/<template_name>.unity` (auto-resolved from the template name)

The bridge auto-creates `Assets/InfiniteCorridorTask/Tasks/` when missing and refuses to clobber an existing scene at
the resolved path (it calls `CreateSceneFromTemplate` with `overwriteExisting: false`), and regeneration is the two-step
`delete_task_tool` → `create_task_tool` cycle. The success payload carries `message`, `template_name`, `prefab_path`,
`scene_path`, and `simulated_controller_added` (plus the `success` key `Ok` adds). Only `message` and
`simulated_controller_added` come from `SceneCreationResult`, while `prefab_path` and `scene_path` are the bridge's own
auto-resolved locals. `SceneCreationResult.Success` selects the branch, and a false `Success` yields `Prefab generated
at <path> but scene creation failed: <message>`. `TaskPrefabNotFound` is never surfaced as its own key, because
`CreateSceneFromTemplate` folds it into `Message` as `Scene saved to <path> but task prefab was not found at: <path>`.
There is no separate scene-only MCP tool, and `create_task_tool` is the single entry point for both prefab and scene
generation, mirroring the Editor menu's single user-driven flow. The companion `delete_task_tool` removes the scene +
per-scene `savedFullScreenViews` companion + task prefab + every `<TemplateName>-` segment prefab in one call, refusing
outright when the template name resolves to a `DeleteProtectedPaths` entry.

All entry points converge on `CreateFromTemplate` and `CreateSceneFromTemplate`, so any change to either method affects
both flows. Test through the menu, the MCP tools, **and** the EditMode fixtures
(`Assets/Tests/EditMode/CreateTaskTests.cs` plus the `ConfigLoaderTests.cs` / `TaskTemplateTests.cs` schema fixtures,
see `/unity-tests`) after any pipeline modification. The automated suite is the surface that catches a regression the
round-trip spot-check will not.

---

## Extending the pipeline

### Adding a new zone trigger type

**Recipe boundary.** A new trigger mode is agent-doable even when its firing behavior is genuinely novel, because the
`/zone-prefabs` worked examples cover a speed-gated interaction reward and a cumulative-occupancy variant end to end.
The recipe holds as long as the new mode is a zone modifier on a copied zone prefab whose root subclasses
`StimulusTriggerZone` and publishes the standard `Stimulus` event. A zone modifier is either a subclass of an existing
zone or a standalone `IResettable` whose concrete type is added to `Task.FindResettableZones()`. Escalate to the human
supervisor only when the behavior needs a new MQTT topic, new `Task.cs` runtime mechanics, or geometry outside a single
corridor segment. Those are paradigm-level and have no author-derived recipe.

This skill owns the **`CreateTask` pipeline edits** for a new `TriggerType`. The full cross-cutting recipe is split four
ways:

| Slice                                   | Owning skill                    |
|-----------------------------------------|---------------------------------|
| Python registry + `TriggerType` enum    | `assets:library-extension`      |
| Hand-authored zone prefab manufacturing | `/zone-prefabs`                 |
| `CreateTask` pipeline edits             | this skill (steps 1 to 3 below) |
| EditMode / PlayMode test updates        | `/unity-tests`                  |

Apply all four skills' bullets in order. The pipeline-side touches owned here:

1. Extend the `trigger_type` literal check in `ConfigLoader.ValidateTemplate` (currently accepts `"interaction"`,
   `"collision"`, `"occupancy_disarm"`, `"occupancy_arm"`, and `"occupancy_trigger"`). Without this, every template that
   uses the new value fails at load time. The same method also gates `occupancy_duration_ms`, requiring it on all three
   occupancy literals and requiring it to be positive and finite whenever present. Extend that gate too when the new
   mode reads a duration. The rest of the validation surface is inventoried in
   [references/template-validation.md](references/template-validation.md), covering cue catalog integrity, the
   `^[A-Za-z0-9_]+$` name pattern, `ValidateVrEnvironment` bounds, duplicate cue-sequence rejection, and the
   `transitions` probability checks.
2. Add a new `string.Equals(trial.triggerType, "<new>", StringComparison.Ordinal)` branch in `BuildSegmentPrefabs` and a
   corresponding `Place<New>Zone` helper following the pattern of `PlaceInteractionZone` / `PlaceCollisionZone` /
   `PlaceOccupancyZone`. The branch chain ends in a fatal `else` (`Unable to place a trigger zone for trial '<name>'.
   The trigger_type must be one of interaction, collision, occupancy_disarm, occupancy_arm, or occupancy_trigger, but
   the template declares '<t>'.`), so a literal added to `ConfigLoader` without a matching branch here fails the build
   loudly rather than saving a zoneless segment. Add a matching member to the standalone `TriggerMode` enum in
   `Assets/InfiniteCorridorTask/Scripts/TriggerMode.cs` (namespace `SL.Tasks`), **appending** rather than inserting,
   because `Interaction` must stay ordinal 0 so an unconfigured prefab field defaults to it. Assign it to
   `StimulusTriggerZone.triggerMode` from the `Place<New>Zone` helper, and add a `case` to the `switch (triggerMode)`
   dispatch in `StimulusTriggerZone.Update`. A new **occupancy-family** literal must additionally be mapped in
   `CreateTask.ResolveOccupancyTriggerMode`, whose `_` default silently falls back to `TriggerMode.OccupancyDisarm`.
   Reuse an existing base prefab where possible. The five current modes add **no** new prefab files, because `collision`
   reuses `StimulusTriggerZone.prefab` and `occupancy_arm` / `occupancy_trigger` reuse `OccupancyTriggerZone.prefab`.
3. A genuinely new **base prefab** needs three registrations:
   1. `McpBridge.DeleteProtectedPaths`, because `BuildSegmentPrefabs` loads zone prefabs by hardcoded path and an
      accidental `delete_asset_tool` would break subsequent generation runs.
   2. The `requiredPaths` array in `CreateTask.ValidateHandAuthoredAssets`, plus a null guard in `BuildSegmentPrefabs`
      matching the existing `stimulusZonePrefab == null || occupancyZonePrefab == null` abort, so a missing base is
      caught before any mutation.
   3. `McpBridge.CloneSourcePrefabs`, if the new prefab is to serve as a sanctioned `clone_zone_prefab` source.

Coordinate the prefab manufacturing through `/zone-prefabs` Step 7, the Python registry parity through
`assets:library-extension` "Adding a new `TriggerType` member", and the fixture updates through `/unity-tests`, so each
skill bullets only its own substeps.

The platform `TriggerType` enum carries all five members (`INTERACTION`, `COLLISION`, `OCCUPANCY_DISARM`,
`OCCUPANCY_ARM`, `OCCUPANCY_TRIGGER`), and the C# `ConfigLoader` accepts all five literals. System support is a
**per-system subset**: a new `TriggerType` member does **not** require a per-system runtime-trial mapping. Each
acquisition system maps only the subset it can resolve to its own runtime trial classes and may leave the rest unmapped.
A configuration that uses an unmapped mode then raises a clear "not mapped to a runtime trial class" error. The concrete
per-system mapping table lives in that system's own plugin, and for Mesoscope-VR that is
`mesoscope:mesoscope-vr-experiment-schema`. All five modes share one MQTT/wire contract, so every mode publishes the
same `Stimulus` event, adds no topics, and does not change `require_interaction` / `require_wait`.
`assets:task-templates` owns `list_supported_trigger_types_tool`, which returns all five values.

### Adding a new cue or segment

No code changes are needed for cues, but **importing a new cue texture is a human hand-off**, because you cannot author
PNG or other binary image assets. To add a new cue texture:

1. **Hand the texture off to the user.** If the referenced image is not already under
   `Assets/InfiniteCorridorTask/Textures/`, stop and ask the user to supply it, stating the intended cue `name`, `code`,
   `length_cm`, and target filename. The user imports the `.png` (or compatible image) and then loops you back to
   continue. You MUST NOT let generation dead-end in `Cue '<name>' references texture '<t>' but no file found at
   <path>.`, the `ConfigLoader` rejection that fires at template load. Because the preflight loads every template, that
   rejection blocks generation of all of them.
2. Reference the filename from the YAML template's `cues[].texture` field together with a unique `name`, `code`, and
   `length_cm`. `ConfigLoader` deserializes with `UnderscoredNamingConvention` and `IgnoreUnmatchedProperties`, so an
   unrecognized key is dropped in silence and the cue then fails validation on whichever field went missing. A dropped
   `length_cm` yields `Cue '<name>' has invalid length 0. Must be positive and finite.`, a dropped `name` yields `A cue
   entry is missing the required 'name' field.`, and a dropped `texture` yields `Cue '<name>' is missing required
   'texture' field.`
3. Regenerate the task via `create_task_tool`, and the cue prefab and matching material are created automatically under
   `Cues/Cue_<name>_<length>cm.prefab` and `Materials/Cue_<name>_<length>cm.mat`.

Segment prefabs are always generated from the template's `trial_structures` block, and hand-authoring them is not
supported under the always-regenerate flow because `CleanGeneratedSegments` would delete the hand-authored prefab on the
next generation pass. Express new segment geometry by adding a trial structure to the YAML template instead.

### Adding a new template-driven field

A template field is a **two-repo mirror**. The YAML deserializer maps each underscored YAML key to the camelCase C#
member, so the two class definitions must stay in lockstep. Otherwise the field is silently dropped, or fails to parse,
at `create_task` time, far from the edit that caused it, in the other repo. There is no automated parity check, so
verify before done against the inventory in [references/template-validation.md](references/template-validation.md).

1. Add the field to the `TaskTemplate` (or nested) Python dataclass in `sollertia-shared-assets`, **and** add the
   matching `[Serializable]` field to the mirror C# class (`TaskTemplate.cs` / `Cue.cs` / `TrialStructure.cs` /
   `VREnvironment.cs`, all in `namespace SL.Config`) in this repo. The C# member name MUST be the camelCase counterpart
   of the underscored YAML key (e.g. `cue_offset_cm` → `cueOffsetCm`), and its optionality / default MUST match the
   Python side.
2. Add a getter or conversion helper (e.g. a `*Unity` accessor) to `TaskTemplate.cs` if the field needs unit conversion.
3. Thread the field through `CreateFromTemplate` to the relevant sub-step.
4. If the field affects geometry, ordering, or asset count, update
   [references/prefab-anatomy.md](references/prefab-anatomy.md) or
   [references/zone-placement.md](references/zone-placement.md) so callers can see how the new field surfaces in the
   generated prefab, and add its row to [references/template-validation.md](references/template-validation.md).
5. **Verify the round-trip**: author a template that sets the new field, run `create_task_tool`, and confirm via
   `inspect_prefab_tool` (or the field's downstream effect) that the value actually arrived on the C# side. A missing or
   mistyped mirror field surfaces here as a dropped value rather than a compile error.

---

## Failure modes

Every symptom `CreateTask` can produce is cataloged in
[references/generation-errors.md](references/generation-errors.md), covering the raw `error: ` prefixed form the
`CreateTask → New Task` menu path returns and the unprefixed text `create_task_tool` callers match.

## Related skills

`automation:csharp-style` (ataraxis marketplace) is the only external-marketplace skill referenced below.

| Skill                                      | Relationship                                                       |
|--------------------------------------------|--------------------------------------------------------------------|
| `/task-prefabs` (this plugin)              | Consumer, invokes `create_task_tool` and validates output          |
| `/zone-prefabs` (this plugin)              | Owns zone authoring and the `IResettable` registration it requires |
| `/mqtt-contract` (this plugin)             | Zone scripts (authored here) own MQTT topics described there       |
| `/gimbl-framework` (this plugin)           | Segment prefabs place GIMBL-derived `Actor` coordinate frame usage |
| `/task-parameters` (this plugin)           | Owns Track Length, the scene-side half of the runtime contract     |
| `/unity-tests` (this plugin)               | Owns the test suite that gates every `CreateTask.cs` change        |
| `assets:task-templates`                    | Upstream, owns YAML authoring and schema evolution                 |
| `assets:library-extension`                 | Owns the Python registry half of a new `TriggerType`               |
| `mesoscope:mesoscope-vr-experiment-schema` | Owns the Mesoscope-VR trigger-type-to-trial-class mapping          |
| `automation:csharp-style`                  | Enforced when editing `CreateTask.cs` or adding new generator code |
| `experiment:vr-driver-interface`           | Host decomposes the cue sequence these generated prefabs render    |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change to `CreateTask.cs`, the zone base prefabs,
the hand-authored shared materials, or the `McpBridge` dispatch surface.

```text
Generator Pipeline Compliance:
- [ ] Any change to zone placement is reflected in PlaceInteractionZone, PlaceCollisionZone, and PlaceOccupancyZone
      if applicable
- [ ] New zone types appear in the BuildSegmentPrefabs trigger-type chain (all five literals: interaction,
      collision, occupancy_disarm, occupancy_arm, occupancy_trigger), set StimulusTriggerZone.triggerMode from the
      appended TriggerMode member, and (for a new base prefab) appear in McpBridge.DeleteProtectedPaths,
      CreateTask.ValidateHandAuthoredAssets, and McpBridge.CloneSourcePrefabs
- [ ] Cue prefab regeneration remains shared and skip-if-exists on BOTH the prefab and its material
- [ ] Segment prefab regeneration remains always-rebuilt via CleanGeneratedSegments over exact
      TemplateName-TrialName names
- [ ] Every missing hand-authored input still aborts before any asset is written (ValidateHandAuthoredAssets), and
      no code path can save a segment without a trigger zone
- [ ] Hardcoded asset paths (Prefabs/, Cues/, Materials/, Textures/) are unchanged, or every call site is updated
- [ ] ValidateHandAuthoredAssets still carries _CueShaderReference.mat in its requiredPaths, so a missing file
      aborts before any mutation, and LoadReferenceCueShader still falls back through the documented chain for a
      reference material that loads with a null shader
- [ ] CreateTask → New Task Editor menu and McpBridge.GenerateTask produce identical assets for the same template
- [ ] CreateSceneFromTemplate runs MainWindow.EnsureControllers so the generated scene contains one GameObject
      per ControllerTypes enum value under the "Controllers" root, and MainWindow.RemoveDefaultMainCamera so it
      retains no Camera tagged MainCamera or named "Main Camera"
- [ ] A freshly generated task prefab reaches no Task.cs enabled = false bailout (corridor-count ceiling, configPath
      lookup, track-length-covers-depth) at the default Track Length, and any new standalone IResettable implementer
      is registered in Task.FindResettableZones()
- [ ] Assets/Tests/EditMode/CreateTaskTests.cs covers the new or changed generator branch, and a new trigger_type
      literal is covered in ConfigLoaderTests.cs (test templates use the ZZTest_ prefix, test cues the ZZ prefix)
- [ ] A new McpBridge.Dispatch case joins the [TestCase] list on
      McpBridgeTests.Dispatch_DeclaredToolName_DoesNotFallThroughToUnknownTool, or on McpBridgePlayModeTests /
      McpBridgeTaskParametersTests when its handler needs Play Mode or the FullScreenViewManager fixture
- [ ] Both Unity test platforms pass (see /unity-tests)
- [ ] After any generator change, regenerate a representative template via create_task_tool and spot-check the
      prefab via inspect_prefab_tool against the expected hierarchy
- [ ] Any new template field has a matching [Serializable] C# mirror field (camelCase of the underscored YAML key,
      with matching optionality/default) in TaskTemplate.cs / Cue.cs / TrialStructure.cs / VREnvironment.cs, verified
      by a create_task_tool + inspect_prefab_tool round-trip and recorded in references/template-validation.md,
      because the two-repo schema mirror has no automated parity check
```
