# Required shared assets

The hand-authored prefabs and materials `CreateTask` resolves by hardcoded path, the severity a missing entry is handled
at, and the rename prohibition that keeps every one of those paths resolvable.

---

## Protected assets

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

---

## Missing-asset severities

A missing or broken entry is handled at one of three severities:

- **Aborts before any mutation.** The `ValidateHandAuthoredAssets` preflight is the primary gate. It checks
  `Materials/Floor.mat`, `Materials/Wall.mat`, both zone base prefabs, `Prefabs/<padding_prefab_name>.prefab`, and
  `Materials/_CueShaderReference.mat`, and it runs before `BuildCuePrefabs` and `CleanGeneratedSegments`. It reports
  every missing path at once under `Unable to generate the task. Every hand-authored asset the pipeline references must
  exist, but these are missing from the project:`. `BuildSegmentPrefabs` keeps its own null guards (`Unable to build the
  segment prefabs. Floor.mat and Wall.mat must both exist under <folder>, but at least one of them is missing.` and
  `Unable to build the segment prefabs. StimulusTriggerZone.prefab and OccupancyTriggerZone.prefab must both exist under
  <folder>, but at least one of them is missing. Restore both hand-authored zone prefabs before generating.`) and
  `CreateFromTemplate` keeps its late `error: Unable to assemble the corridor. The padding prefab must exist at
  '<path>', but it is missing.` check, both as defense for direct callers and for a deletion racing the build. Every
  segment that reports success carries a trigger zone.
- **Aborts inside `BuildSegmentPrefabs`.** Beyond the table: a generated cue prefab that a trial's `cue_sequence`
  references but that is missing from `Cues/` ends the run with `Unable to build the segment prefab for trial '<name>'.
  The cue prefab must exist at '<path>', but it is missing.`
- **Degrades with a warning.** `LoadReferenceCueShader` logs a `Debug.LogWarning` (`Unable to load the canonical cue
  shader reference. The material at '<path>' must exist, but it is missing, so the shader falls back to a
  hand-authored Cue*.mat material or Shader.Find.`) and resolves the shader through its fallback chain, a hand-authored
  `Cue*.mat` heuristic (a `Materials/` material whose filename starts with `Cue` but not `Cue_`), then
  `Shader.Find("Legacy Shaders/Diffuse")`, then `Standard`. Only a `Materials/_CueShaderReference.mat` that loads with a
  null shader reaches it from `CreateFromTemplate`, because a missing file is already refused by
  `ValidateHandAuthoredAssets`.

`Materials/TargetMat.mat` sits outside all three paths. The generator never loads it by path, and the table lists it
only because `DeleteProtectedPaths` protects it.

---

## Rename prohibition

You MUST NOT rename these assets, because `BuildSegmentPrefabs` / `LoadReferenceCueShader` /
`ValidateHandAuthoredAssets` resolve them by hardcoded path, and the trigger zone prefabs bind `TargetMat.mat` by
serialized GUID. The scene base template `Assets/Scenes/ExperimentTemplate.unity` is also in
`McpBridge.DeleteProtectedPaths` but is consumed by `CreateSceneFromTemplate` rather than by the prefab build pass.
`Prefabs/Padding.prefab` is not synthesized either. It is hand-authored from Unity built-in primitives, a `Floor` plane
plus `Walls/LeftWall` and `Walls/RightWall` quads mirroring the generated segment layout, and its three renderers bind
the same shared `Materials/Floor.mat` and `Materials/Wall.mat` that `BuildSegmentPrefabs` bakes into every generated
segment. A material edit is therefore corridor-wide, only the geometry is padding-local, and a corridor-cap geometry
change is a prefab edit rather than a code change.
