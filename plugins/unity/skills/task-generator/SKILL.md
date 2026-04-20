---
name: task-generator
description: >-
  Documents the CreateTask.cs editor pipeline that builds cue, segment, and task prefabs from YAML task templates,
  plus the hand-authored anatomy of segment and zone prefabs. Covers generation order, cue and segment prefab
  synthesis, zone placement math, and constraints on extending the pipeline. Use when modifying CreateTask.cs, adding
  a new zone type, hand-authoring a new segment prefab, introducing a new cue texture, or diagnosing why a generated
  prefab's geometry disagrees with expectations.
user-invocable: true
---

# Sollertia Unity task generator

Documents the `CreateTask.cs` editor pipeline and the prefab anatomy it assumes. This skill is the reference for
**extending** generation, not for invoking it — invocation is owned by `/task-prefabs`.

---

## Scope

**Covers:**
- The `CreateTask.CreateFromTemplate` pipeline (cue synthesis, segment synthesis, task assembly)
- Cue prefab internal layout (`Right`/`Left` quads) generated from templates
- Segment prefab internal layout (cue instances, `Floor`, `Walls`, `ResetZone`, trigger zone)
- Zone placement math (`PlaceLickZone` and `PlaceOccupancyZone`)
- Constraints on adding new zone types, cue shapes, or segment layouts
- The `CreateTask → New Task` Editor menu entry and its relationship to `generate_task_prefab_tool`

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
├── ConfigLoader.LoadTemplate                ← YAML → TaskTemplate
│
├── BuildCuePrefabs(template)                ← Cues/Cue_<name>.prefab  (idempotent)
│   │
│   ├── Load/create Materials/Cue_<name>.mat from Textures/<cue.texture>
│   ├── Build Right + Left quads scaled to cue.length_cm / cm_per_unity_unit
│   └── Save Cues/Cue_<name>.prefab
│
├── BuildSegmentPrefabs(template)            ← Prefabs/<segment-name>.prefab  (idempotent)
│   │
│   ├── Place cue instances sequentially along +Z
│   ├── Build Floor (plane) and Walls (LeftWall + RightWall quads)
│   ├── For each trial_structure[]:
│   │   ├── trigger_type == "lick"       → PlaceLickZone
│   │   └── trigger_type == "occupancy"  → PlaceOccupancyZone
│   └── Place ResetZone at segment start (local Z = 1)
│
└── Assemble task GameObject
    │
    ├── Iterate all (segment_count ^ depth) corridor permutations
    ├── Instantiate segment prefabs along +Z inside each Corridor<indices>
    ├── Strip StimulusTriggerZone and ResetZone from non-first segments in each corridor
    ├── Set StimulusTriggerZone.showBoundary on the first segment from template
    └── PrefabUtility.SaveAsPrefabAsset(task, savePath)
```

Keys:
- **Idempotent steps**: `BuildCuePrefabs` and `BuildSegmentPrefabs` check for existing prefabs and skip them. To
  regenerate a cue or segment prefab, delete it from the Editor first.
- **Non-idempotent step**: the final task prefab is always overwritten at `savePath`.
- **Validation warning** (not an error): if the measured segment-prefab length disagrees with
  `sum(cue.length_cm / cm_per_unity_unit)` by more than `0.01`, `CreateTask` logs a warning but proceeds using the
  template's computed length.

---

## Cue prefab anatomy

Generated under `Assets/InfiniteCorridorTask/Cues/Cue_<name>.prefab`.

```text
Cue_<name>
├── Right                     localPosition = (0.49, 0.5, lengthUnity/2)
│                             rotation      = (0, 90, 0)
│                             scale         = (-lengthUnity, 1, 1)   ← negative X flips the quad
│                             components    = [MeshFilter, MeshRenderer]
│                             material      = Materials/Cue_<name>.mat
│
└── Left                      localPosition = (-0.49, 0.5, lengthUnity/2)
                              rotation      = (0, -90, 0)
                              scale         = (lengthUnity, 1, 1)
                              components    = [MeshFilter, MeshRenderer]
                              material      = Materials/Cue_<name>.mat
```

- `lengthUnity = cue.length_cm / cm_per_unity_unit`.
- Both quads use the built-in `Quad.fbx` mesh and the `Standard` shader.
- No collider, no script — cues are purely visual.
- The material's `_MainTex` is assigned from `Assets/InfiniteCorridorTask/Textures/<cue.texture>`; texture must be
  imported into Unity before generation.

To add a new cue texture: import the `.png` into `Assets/InfiniteCorridorTask/Textures/`, reference it from the
YAML template's `cues[].texture` field, and generate — the cue prefab and material will be created automatically.

---

## Segment prefab anatomy

Generated under `Assets/InfiniteCorridorTask/Prefabs/<segment-name>.prefab`.

```text
<segment-name>
│   localPosition = (0, 0, -cueOffsetUnity)   ← cue_offset_cm / cm_per_unity_unit shifts the cues upstream
│
├── Cue<CueA>                 localPosition.z = 0
├── Cue<CueB>                 localPosition.z = length(Cue<CueA>)
├── ...                       (one per entry in segment.cue_sequence, placed end-to-end)
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
└── <ResetZone>               localPosition = (0, 0.5, 1)   ← placed only when a trial structure exists
```

### Required shared assets

`BuildSegmentPrefabs` aborts if any of these are missing:

| Asset                                       | Type      | Purpose                                  |
|---------------------------------------------|-----------|------------------------------------------|
| `Prefabs/StimulusTriggerZone.prefab`        | GameObject| Base prefab for lick-mode zones          |
| `Prefabs/OccupancyTriggerZone.prefab`       | GameObject| Base prefab for occupancy-mode zones     |
| `Prefabs/ResetZone.prefab`                  | GameObject| Placed at every segment's start          |
| `Materials/Floor.mat`                       | Material  | Shared floor material                    |
| `Materials/Wall.mat`                        | Material  | Shared wall material                     |

Do not rename these assets — the paths are hardcoded in `BuildSegmentPrefabs`.

### Padding prefab

The `vr_environment.padding_prefab_name` template field names a separate prefab under `Prefabs/`. `CreateTask` appends
one padding instance at `depth * min(segmentLengths) - 1` past every corridor to prevent the camera from seeing past
the last real segment. Padding prefabs are **hand-authored** and referenced by name; there is no automatic synthesis.

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

StimulusTriggerZone.localPosition   = (0, 0.505, zoneCenterUnity)
StimulusTriggerZone.BoxCollider.size = (1, 1, zoneSizeUnity)
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

`validate_prefab_against_template_tool` (`/task-prefabs`) re-derives `zone_z = zone.transform.localPosition.z` and
`zone_size = BoxCollider.size.z` from the prefab and compares them against `expected_center = (zone_end - zone_start)
/ (2 * cm_per_unity_unit)` and `expected_size = (zone_end - zone_start) / cm_per_unity_unit`. Note the validator
reconstructs the **lick-mode** formulas. In occupancy mode, the generator places the root at `rootZ` (past the
waiting range), but the validator still computes `expected_center` against the waiting range. **Occupancy segments
will report `zone_z_match: false` even when correctly generated.** This is a known drift between generator and
validator; treat occupancy `match: false` as informational unless the reported `zone_z` is also wrong by the offset
math above.

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
- The first segment's `StimulusTriggerZone.showBoundary` is set from
  `template.GetSegmentMarkerVisibility(segmentName)`, which maps trigger type → `show_stimulus_collision_boundary`.
- Corridor X-spacing is `vr_environment.corridor_spacing_cm / cm_per_unity_unit`.
- The padding Z-shift uses the **minimum** segment length, not the current segment's length. Segments longer than
  the minimum will overlap the padding by design (the overlap is outside the camera's view).

---

## Entry points

### Editor menu

`CreateTask.CreateNewTask` is registered at `CreateTask → New Task` in the Unity menu bar. It opens two file dialogs
(template selector, save target) and calls `CreateFromTemplate` with user input.

### MCP bridge

`McpBridge.GenerateTaskPrefab` calls the same `CreateFromTemplate` with:
- `absoluteTemplatePath = Application.dataPath/InfiniteCorridorTask/Configurations/<template_name>.yaml`
- `relativeConfigPath = /InfiniteCorridorTask/Configurations/<template_name>.yaml` (leading slash; stored on `Task`)
- `savePath = Assets/InfiniteCorridorTask/Tasks/<template_name>.prefab` (or the caller's override)

Both entry points converge on `CreateFromTemplate`, so any change to that method affects both. Test through the MCP
tool after any pipeline modification.

---

## Extending the pipeline

### Adding a new zone trigger type

1. Add a new `triggerType` enum value in `sollertia-shared-assets` via `/task-templates`.
2. Create a new zone base prefab under `Prefabs/<NewZone>TriggerZone.prefab` with the required script and colliders.
3. Add a new `if (trial.triggerType == "<new>")` branch in `BuildSegmentPrefabs` and a corresponding `Place<New>Zone`
   method following the pattern of `PlaceLickZone` / `PlaceOccupancyZone`.
4. Update `McpBridge.ValidatePrefabAgainstTemplate` if the new zone requires different geometry validation.
5. Update `/task-prefabs` zone behavior reference with the new trigger type.

### Adding a new cue or segment

No code changes needed for cues — new textures and YAML entries are enough.

For hand-authored segment prefabs (which `BuildSegmentPrefabs` does not generate because the prefab already exists),
create the prefab by **copying an existing segment prefab** and modifying the cue sequence, then update the YAML
template to reference it. Validate with `/task-prefabs`.

### Adding a new template-driven field

1. Add the field to the `TaskTemplate` (or nested) class in `sollertia-shared-assets`.
2. Add a getter or conversion helper (e.g. a `*Unity` accessor) to `TaskTemplate.cs` if the field needs unit
   conversion.
3. Thread the field through `CreateFromTemplate` to the relevant sub-step.
4. If the field affects zone geometry, extend `validate_prefab_against_template_tool` to check it.

---

## Failure modes

| Symptom                                               | Root cause                                                      | Resolution                                         |
|-------------------------------------------------------|-----------------------------------------------------------------|----------------------------------------------------|
| `BuildCuePrefabs` error: `Failed to load texture`     | Cue's `texture` field references a file not in `Textures/`      | Import the texture, retry generation               |
| `BuildSegmentPrefabs` error: `Missing Floor.mat`      | Shared materials deleted or renamed                             | Restore from git                                   |
| Segment length warning in Console                     | Cue lengths do not sum to measured prefab length                | Either regenerate the segment or fix template cues |
| `generate_task_prefab_tool` error: `No segment found` | Template references a segment prefab that was never authored    | Hand-author the segment prefab first               |
| Zone geometry looks wrong in scene view               | Template's cm values or `cm_per_unity_unit` mismatch             | Recheck YAML; regenerate                           |
| Occupancy validator reports `zone_z_match: false` on a correctly generated prefab | Known validator / generator drift (see above) | Use zone-collider visual inspection in Editor      |
| Cue textures appear mirrored on Left or Right wall    | Quad scale sign is flipped (Right uses negative X)              | Intentional — each wall shows a correctly-oriented cue |

---

## Verification checklist

```text
- [ ] Any change to zone placement is reflected in both PlaceLickZone and PlaceOccupancyZone if applicable
- [ ] New zone types appear in BuildSegmentPrefabs trigger-type switch AND in the zone base prefab set
- [ ] Cue and segment prefab regeneration paths remain idempotent (existing prefabs are skipped)
- [ ] Hardcoded asset paths (Prefabs/, Cues/, Materials/, Textures/) are not changed without updating every call site
- [ ] CreateTask → New Task Editor menu and McpBridge.GenerateTaskPrefab produce identical prefabs for the same template
- [ ] After any generator change, run validate_prefab_against_template_tool to confirm round-trip invariants hold
```

---

## Related skills

| Skill                                   | Relationship                                                          |
|-----------------------------------------|-----------------------------------------------------------------------|
| `/task-prefabs` (this plugin)           | Consumer — invokes `generate_task_prefab_tool` and validates output   |
| `/mqtt-contract` (this plugin)          | Zone scripts (authored here) own MQTT topics described there          |
| `/gimbl-framework` (this plugin)        | Segment prefabs place GIMBL-derived `Actor` coordinate frame usage    |
| assets plugin `/task-templates`         | Upstream — owns YAML authoring and schema evolution                   |
| `/csharp-style` (automation plugin)     | Enforced when editing `CreateTask.cs` or adding new generator code    |
