# Zone placement math

The per-mode geometry `CreateTask` bakes into a generated segment's trigger zone. Read this when auditing a
generated segment against its template's zone-cm fields, or when adding a `Place<New>Zone` helper.

`CreateTask` positions zones using the template's cm-valued fields, converted by `cm_per_unity_unit`. All math below
runs per trial structure — a single segment carries exactly one `StimulusTriggerZone` or `OccupancyTriggerZone`; a
`trigger_type` with no placement branch aborts the build rather than saving a zoneless segment.

Every placement helper writes `triggerMode`, `trialName`, and boundary visibility (via `ApplyBoundaryVisibility`,
which drives both `showBoundary` and the co-located `MeshRenderer.enabled`) onto the root `StimulusTriggerZone`. At
runtime the zone dispatches on `triggerMode` in `StimulusTriggerZone.Update` and applies the per-mode firing rule.

`PlaceCollisionZone` reuses `StimulusTriggerZone.prefab` (stripping its `GuidanceRegion` child and setting the root
collider as a thin boundary wall). `occupancy_arm` and `occupancy_trigger` reuse `OccupancyTriggerZone.prefab`
through `PlaceOccupancyZone`, which sets the occupancy sub-mode and the trial's `occupancy_duration_ms` on the
placed zone. No mode has its own prefab file. The occupancy sub-modes share a single `OccupancyZone.occupancyMet`
signal (the generic "occupancy requirement met" flag), and the parent `StimulusTriggerZone` applies the per-mode
firing rule.

Two constants recur below: `ZoneVerticalOffset = 0.505` (the Y offset of every zone root) and
`GuidanceColliderDepth = 0.4` (the Z depth of both guidance-region colliders **and** of the collision-mode
boundary wall).

---

## Interaction mode (`PlaceInteractionZone`)

```text
zoneStartUnity        = trial.stimulus_trigger_zone_start_cm / cm_per_unity_unit
zoneEndUnity          = trial.stimulus_trigger_zone_end_cm   / cm_per_unity_unit
zoneCenterUnity       = (zoneStartUnity + zoneEndUnity) / 2
zoneSizeUnity         = zoneEndUnity - zoneStartUnity
stimulusLocationUnity = trial.stimulus_location_cm          / cm_per_unity_unit

StimulusTriggerZone.localPosition      = (0, 0.505, zoneCenterUnity)
StimulusTriggerZone.BoxCollider.size   = (1, 1, zoneSizeUnity)
StimulusTriggerZone.BoxCollider.center = (0, 0, 0)

GuidanceRegion.BoxCollider.size   = (1, 1, 0.4)                                ← GuidanceColliderDepth
GuidanceRegion.BoxCollider.center = (0, 0, stimulusLocationUnity - zoneCenterUnity + 0.2)
```

- Y offset `0.505` is deliberate — raises the zone just above the floor to avoid collider overlap.
- GuidanceRegion depth is **not** template-driven (the hardcoded `0.4` unit collider depth is a design choice).
- The `+ 0.2` term is half that fixed collider depth. It anchors the guidance region's **leading edge** on
  `stimulus_location` and extends the region forward from there, so the region's center sits half a collider depth
  downstream of the declared stimulus location. The far edge carries no behavioral meaning.

## Collision mode (`PlaceCollisionZone`)

```text
stimulusLocationUnity = trial.stimulus_location_cm / cm_per_unity_unit
wallCenterUnity       = stimulusLocationUnity + 0.2                          ← + GuidanceColliderDepth / 2

StimulusTriggerZone.localPosition      = (0, 0.505, wallCenterUnity)
StimulusTriggerZone.BoxCollider.size   = (1, 1, 0.4)                         ← GuidanceColliderDepth
StimulusTriggerZone.BoxCollider.center = (0, 0, 0)

(GuidanceRegion child stripped — collision mode has no sensor / occupancy region)
```

- `ConfigureRootZoneCollider` centers the collider on the zone origin, so the origin carries the `+ 0.2` offset.
  The result is a 0.4-deep boundary wall whose **leading edge** lands exactly on `stimulus_location`, matching where
  the interaction guidance region and the occupancy root both begin — every mode fires at the location the template
  declares.
- Collision mode fires the stimulus **unconditionally** when the actor crosses that boundary wall — no sensor, no
  occupancy. `PlaceCollisionZone` reuses `StimulusTriggerZone.prefab` and strips its `GuidanceRegion` child.
- Collision mode keeps the `showBoundary` visibility toggle (the per-trial `show_stimulus_collision_boundary`
  template field), applied at placement for every segment and re-applied to the first segment in corridor assembly.

## Occupancy modes (`PlaceOccupancyZone`)

`PlaceOccupancyZone` serves all three occupancy sub-modes (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`)
from the same `OccupancyTriggerZone.prefab`. `CreateTask.ResolveOccupancyTriggerMode` maps the literal to the
occupancy sub-mode, which `PlaceOccupancyZone` writes onto the placed zone alongside the trial's
`occupancy_duration_ms`. The geometry below is identical across the three sub-modes. They differ only in the runtime
firing rule the parent `StimulusTriggerZone` applies to the shared `OccupancyZone.occupancyMet` signal:

- **`occupancy_disarm`**: a collision with the boundary fires while occupancy is **not** met (occupancy "disarms" the
  boundary).
- **`occupancy_arm`**: occupying the zone **arms** the boundary; colliding with the now-armed boundary (occupancy
  **met**) fires. It is the inverse of `occupancy_disarm`.
- **`occupancy_trigger`**: occupying the zone for `occupancy_duration_ms` fires the stimulus **immediately**, with no
  boundary collision.

All three occupancy modes keep the occupancy-guidance brake (`OccupancyGuidanceZone` publishing `Delay`).

```text
zoneStartUnity, zoneEndUnity, zoneCenterUnity, zoneSizeUnity, stimulusLocationUnity   (same derivations)

rootZ = stimulusLocationUnity + zoneSizeUnity / 2
occupancyCenterOffset = zoneCenterUnity - rootZ                                ← negative: occupancy is upstream

StimulusTriggerZone.localPosition      = (0, 0.505, rootZ)
StimulusTriggerZone.BoxCollider.size   = (1, 1, zoneSizeUnity)
StimulusTriggerZone.BoxCollider.center = (0, 0, 0)

OccupancyZone.occupancyDurationMs  = trial.occupancy_duration_ms             ← baked from the template, in ms

OccupancyRegion.BoxCollider.size   = (1, 1, zoneSizeUnity)
OccupancyRegion.BoxCollider.center = (0, 0, occupancyCenterOffset)

OccupancyGuidanceRegion.BoxCollider.size   = (1, 1, 0.4)
OccupancyGuidanceRegion.BoxCollider.center = (0, 0, occupancyCenterOffset + zoneSizeUnity/2 - 0.2)
```

- The root is positioned **past** the waiting range at the stimulus boundary. This boundary collider is the
  "tripwire": for `occupancy_disarm` it fires when occupancy is **not** met, for `occupancy_arm` it fires when
  occupancy **is** met. For `occupancy_trigger` the boundary collider is unused — occupancy met fires immediately.
- `OccupancyGuidanceRegion` sits at the downstream end of the occupancy range, offset by half a collider depth to
  keep it inside the occupancy collider.
- `occupancyDurationMs` is the only non-geometric value the occupancy anatomy above records. It carries the trial's
  `occupancy_duration_ms` straight onto the child `OccupancyZone`, and `ConfigLoader.ValidateTemplate` requires that
  template field for all three occupancy literals, so the helper always has a value to bake.

---

## Critical invariant

For interaction-mode trials the generator places the zone root at the zone's center such that
`zone_z = zone.transform.localPosition.z` equals `(zone_end + zone_start) / (2 * cm_per_unity_unit)`,
and the `BoxCollider.size.z` equals `(zone_end - zone_start) / cm_per_unity_unit`. For the three
occupancy modes (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`) the generator places the
root at `rootZ` (past the waiting range) instead — the root collider marks the boundary, and the wait
region lives on the child `OccupancyRegion`. For collision mode the root is a 0.4-deep boundary wall centered at
`stimulus_location + 0.2`, with no occupancy region and no guidance child. Anyone auditing a generated segment via
`inspect_prefab_tool` should apply the appropriate formula per `trigger_type` when comparing the prefab
against the template's zone-cm fields.
