# Generated prefab anatomy

The exact hierarchy, transform, and material layout `CreateTask` writes for cue prefabs, segment prefabs,
and the assembled task prefab. Read this when auditing a generated asset against its template, or when a
change to `CreateTask.cs` alters the shape of what it writes.

Zone geometry is deliberately excluded — the per-mode placement math lives in
[zone-placement.md](zone-placement.md).

---

## Cue prefab anatomy

Generated under `Assets/InfiniteCorridorTask/Cues/Cue_<name>_<length>cm.prefab` — the length suffix lets a single
cue name resolve to distinct prefabs when different templates declare different lengths (e.g. `Cue_A_30cm.prefab`
and `Cue_A_50cm.prefab`). `CreateTask.FormatCueLengthLabel` renders the suffix culture-invariantly with up to two
decimals and no trailing zeros, so `30` and `37.5` are both well-formed labels.

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
- The material's `_MainTex` is assigned from `Assets/InfiniteCorridorTask/Textures/<cue.texture>`; the texture file
  must exist (`ConfigLoader` enforces that at load) **and** must already be imported into the AssetDatabase as a
  `Texture2D` (`BuildCuePrefabs` enforces that).

## Cue build internals (`BuildCuePrefabs`)

Shader resolution through `LoadReferenceCueShader` runs once for the whole pass, before the first cue is
examined, so its missing-reference warning and folder scan fire once per invocation rather than once per cue.
The per-cue sequence then runs, in this order:

1. **Resolve the asset stem** `Cue_<name>_<lengthLabel>cm` and derive both the prefab and the material path from it.
   Cue assets are keyed by `(name, length_cm)` only, so every template that declares a matching cue shares them.
2. **Load the texture** as a `Texture2D`. A file present on disk but not imported aborts the build with
   `BuildCuePrefabs: Failed to load texture '<texture>'.` The load happens before the cached-asset checks so a
   template that changed a cue's texture is caught rather than served a stale asset.
3. **Reject a cached material built from a different texture.** When `Cue_<stem>.mat` exists and its `_MainTex` is
   not the texture the template declares, the build aborts:
   `BuildCuePrefabs: Cue '<name>' at <n> cm declares texture '<t>', but the cached material '<stem>.mat' was built
   from a different texture.` The message names the two resolutions — delete both cue assets to rebuild them for
   every template that shares the identity, or give the cue a distinct name or length so it occupies its own slot.
   Rebuilding in place is refused because it would silently alter every other template's rendering.
4. **Skip only when BOTH assets survive.** The skip-if-exists test requires the prefab *and* its material. A material
   deleted from under a surviving prefab leaves that prefab rendering untextured, so `BuildCuePrefabs` deletes the
   surviving prefab and rebuilds both — `SaveAsPrefabAsset` merges into a same-named asset already at the path, and
   that merge would otherwise keep both wall renderers on the deleted material. The two quads are then built
   from the already-resolved shader and saved.

### Cue shader resolution

`LoadReferenceCueShader` reads the shader from `Assets/InfiniteCorridorTask/Materials/_CueShaderReference.mat` — a
hand-authored material protected from deletion via `McpBridge.DeleteProtectedPaths`. The shader is Unity's built-in
`Legacy Shaders/Diffuse`, chosen because it renders both walls of a cue correctly even when the Right wall uses a
negative geometry scale to mirror its texture; the Standard shader breaks under negative scales and Unlit shaders
drop lighting altogether. When the reference material is missing, the method logs a `Debug.LogWarning` and falls
back through a hand-authored `Cue*.mat` heuristic (a `Materials/` material whose filename starts with `Cue` but not
`Cue_`), then `Shader.Find("Legacy Shaders/Diffuse")`, then `Standard`. The reference material is the canonical
source and must be restored from version control when missing.

---

## Segment prefab anatomy

Generated under `Assets/InfiniteCorridorTask/Prefabs/<TemplateName>-<TrialName>.prefab`, the name
`CreateTask.CanonicalSegmentName` builds from the template filename (without extension) and the trial key under
`trial_structures`. `ConfigLoader.SegmentNameComponentPattern` (`^[A-Za-z0-9_]+$`) excludes the hyphen from both
halves, so the joined filename carries exactly one hyphen and splits back into exactly one owning template even
where one template basename nests another. The prefab root's `m_Name` matches the filename.

```text
<TemplateName>-<TrialName>
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
├── Walls                     localPosition = (0, 0, 0)
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
└── <StimulusTriggerZone or OccupancyTriggerZone>   ← always exactly one, selected by the trial's trigger_type
```

- There is **no** in-segment reset volume. The segment root's authored `(0, 0, -cueOffsetUnity)` is the only
  cue-offset mechanism, and per-lap zone reset is driven at runtime by `Task.ResetZoneStates()` over the
  `IResettable[]` that `Task.FindResettableZones()` collects — see the `## Runtime contract` section of `SKILL.md`.
- Every segment prefab corresponds one-to-one to a trial structure and always carries exactly one trigger zone. A
  `trigger_type` with no placement branch aborts the build rather than saving a zoneless segment.

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
│   ├── <segment0>       localPosition = (0, 0, -cueOffsetUnity)
│   ├── <segment0>       localPosition = (0, 0, segmentLengths[0] - cueOffsetUnity)   ← same segment, re-instantiated
│   ├── ... (segments_per_corridor instances total)
│   └── Padding          localPosition = (0, 0, zShift - cueOffsetUnity)
│
├── Corridor001          localPosition = (corridorSpacingUnity, 0, 0)
│   └── ...
│
└── Corridor<N>          (one per permutation of segment indices, segments_per_corridor deep)
```

Rules enforced by the assembly loop:
- Every segment and padding instance is reparented with `worldPositionStays: false` and then shifted by a
  `localPosition +=` addition, so each instance keeps its prefab root's authored `localPosition` and the Z shift
  stacks on top of it. Each segment adds `(0, 0, zShift)`, the running total of the segment lengths placed before
  it. Segment roots are authored at `(0, 0, -cueOffsetUnity)`, which is why the corridor's first segment sits at
  `-cueOffsetUnity` rather than at the origin. `Padding.prefab`'s root sits at the origin, so its shift resolves to
  the value shown above.
  Each `Corridor<indices>` root is parented with the default `worldPositionStays` and receives its X spacing through a
  direct `localPosition` assignment.
- Only the **first** segment in each corridor retains its trigger zone. Later segments are visual-only: the single
  `StimulusTriggerZone` GameObject is removed with `DestroyImmediate`, and nothing else is stripped.
- Boundary visibility is written through `CreateTask.ApplyBoundaryVisibility`, which sets both
  `StimulusTriggerZone.showBoundary` and the co-located `MeshRenderer.enabled` from the trial's
  `show_stimulus_collision_boundary`. Both hand-authored zone prefabs ship the renderer enabled and
  `StimulusTriggerZone` only reconciles it from `Start`, so leaving the renderer alone would draw the boundary
  across the whole corridor cross-section in the Scene view. It is applied once per segment inside the `Place*Zone`
  helpers and re-applied to the first segment of each corridor during assembly.
- Corridor X-spacing is `vr_environment.corridor_spacing_cm / cm_per_unity_unit`.
- Each corridor's padding is anchored on that corridor's own accumulated segment length, so corridors mixing trials
  of different lengths each butt their padding against their true end. There is no cross-corridor minimum and no
  designed overlap.

### Padding prefab

The `vr_environment.padding_prefab_name` template field names the padding prefab under `Prefabs/`. `CreateTask`
appends one padding instance at `zShift - cueOffsetUnity` past every corridor — where `zShift` is the running sum
of that corridor's own segment lengths — to prevent the camera from seeing past the last real segment. The padding
prefab is **hand-authored** and referenced by name; there is no automatic synthesis. It is in the
required-shared-assets table in `SKILL.md` and is protected by `McpBridge.DeleteProtectedPaths`.

Its geometry is built from Unity built-in primitive meshes — a `Floor` plane plus `Walls/LeftWall` and
`Walls/RightWall` quads mirroring the generated segment layout — with two hand-authored materials, so a
corridor-cap geometry change is a prefab edit rather than a code change.
`Assets/InfiniteCorridorTask/Models/blender_TunnelSegment.blend` is the Blender 4.5.0 LTS source mesh for the
corridor tunnel segment, but `Padding.prefab` references it only through a leftover `Animator` avatar, never for
its geometry — re-exporting the model changes nothing about the padding cap.
