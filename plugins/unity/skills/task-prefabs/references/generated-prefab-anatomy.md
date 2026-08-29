# Inspected zone anatomy

Per-mode hierarchy of the stimulus zone `CreateTask` places in each first segment, as `inspect_prefab_tool` reports it.
Load this when an `inspect_prefab_tool` result has to be judged against the template's `trigger_type`. The tool's
response shape and the general reading rules stay in `SKILL.md` "Reading inspect_prefab_tool output".

The five `trigger_type` modes are `interaction`, `collision`, `occupancy_disarm`, `occupancy_arm`, and
`occupancy_trigger`. No mode has its own prefab file: `interaction` and `collision` both instantiate
`StimulusTriggerZone.prefab`, the three occupancy modes all instantiate `OccupancyTriggerZone.prefab`, and every mode
publishes the same `Stimulus` event. `CreateTask` sets the `StimulusTriggerZone.triggerMode` enum field (`Interaction`,
`Collision`, `OccupancyDisarm`, `OccupancyArm`, `OccupancyTrigger`) from `trigger_type` and the zone dispatches on it at
runtime.

**`triggerMode` is not observable from `inspect_prefab_tool`.** The tool emits `name`, `active_self`, the three
transform vectors, `components` (type names only), `component_states` (a `type` and an `enabled` flag per component),
the `collider_*` keys, and `children`, and never a serialized MonoBehaviour field value. Distinguish the modes by the
root GameObject's **name**, its **child shape**, and its `collider_size.z`, then confirm the intended mode from the
template's `trigger_type`. `showBoundary` is the exception. `CreateTask` mirrors it onto the always-present
`MeshRenderer`, whose `component_states` entry reports the template's `show_stimulus_collision_boundary` as `enabled`.

Both base prefabs carry the same six root components, emitted in `GetComponents<Component>()` order so `Transform` is
always first:

```text
components: ["Transform", "MeshFilter", "MeshRenderer", "MeshCollider", "BoxCollider",
             "StimulusTriggerZone"]
```

`cm_per_unit` below is the template's `vr_environment.cm_per_unity_unit`, and every length in a tree is in Unity units.
`0.4` is the shared `CreateTask.GuidanceColliderDepth` constant.

---

## Interaction mode (trigger_type == "interaction")

```text
<SegmentName>
└── StimulusTriggerZone              ← localPosition.z = zone_center; collider_size.z = zone width,
    │                                  collider_center = (0, 0, 0)
    └── GuidanceRegion               ← collider_size.z = 0.4; collider_center.z =
                                       stimulus_location - zone_center + 0.2, so the region's
                                       leading edge sits on stimulus_location
                                       components: ["Transform", "BoxCollider", "GuidanceZone"]
```

Key markers:

- Root is named `StimulusTriggerZone`, carries `StimulusTriggerZone` in `components`, and has a `BoxCollider` of size.z
  ≈ `(zone_end - zone_start) / cm_per_unit`.
- Exactly one child named `GuidanceRegion` with `GuidanceZone` in components.
- The interaction sensor reports the animal inside the zone, and the guidance region supplies the guided fire path when
  the animal does not act on its own.

---

## Collision mode (trigger_type == "collision")

```text
<SegmentName>
└── StimulusTriggerZone              ← localPosition.z = stimulus_location + 0.2;
                                       collider_size.z = 0.4, collider_center = (0, 0, 0), so the
                                       thin wall's leading edge sits on stimulus_location
```

Key markers:

- Root is named `StimulusTriggerZone` and carries `StimulusTriggerZone` in `components`. Crossing the thin boundary wall
  fires the stimulus unconditionally, with no sensor and no occupancy.
- **No children.** `CreateTask.PlaceCollisionZone` reuses `StimulusTriggerZone.prefab` and destroys its `GuidanceRegion`
  child outright, so a `children` key on the root means the prefab is miswired.
- The wall depth is the same `0.4` the interaction guidance region uses, and the leading edge is anchored on
  `stimulus_location` in both, so every mode fires at the location the template declares.

---

## Occupancy modes (trigger_type == "occupancy_disarm" / "occupancy_arm" / "occupancy_trigger")

All three occupancy modes instantiate `OccupancyTriggerZone.prefab` unchanged and share one hierarchy, because
`CreateTask.PlaceOccupancyZone` only varies the occupancy sub-mode it writes to `triggerMode`. `OccupancyGuidanceRegion`
is a **grandchild**, nested inside `OccupancyRegion`:

```text
<SegmentName>
└── OccupancyTriggerZone             ← localPosition.z = stimulus_location + zone_size / 2;
    │                                  collider_size.z = zone_size, collider_center = (0, 0, 0), so
    │                                  the root spans forward from stimulus_location
    └── OccupancyRegion              ← collider_size.z = zone_size; collider_center.z =
        │                              zone_center - root_z, covering the wait range
        │                              components: ["Transform", "BoxCollider", "OccupancyZone"]
        └── OccupancyGuidanceRegion  ← collider_size.z = 0.4; collider_center.z sits at the
                                       downstream end of the occupancy range
                                       components: ["Transform", "OccupancyGuidanceZone",
                                                    "BoxCollider"]
```

Key markers shared by all three modes:

- Root is named `OccupancyTriggerZone`. The `StimulusTriggerZone` name appears only in its `components` list, never as
  the GameObject name.
- Exactly one child, `OccupancyRegion` (wait zone), which itself carries exactly one child, `OccupancyGuidanceRegion`
  (the occupancy-guidance brake, which publishes `Delay`).
- If the root has more than one child, or `OccupancyRegion` has no `OccupancyGuidanceRegion` child, the prefab is
  miswired. Regenerate from the template.
- `OccupancyZone` exposes the generic `occupancyMet` signal, and the parent `StimulusTriggerZone` applies the per-mode
  firing rule to it.

Per-mode firing rules (`StimulusTriggerZone.UpdateOccupancyMode`), none of them readable from the hierarchy, so confirm
them from the template's `trigger_type`:

| `trigger_type`      | Resolves on                | Delivers the stimulus when                        |
|---------------------|----------------------------|---------------------------------------------------|
| `occupancy_disarm`  | the boundary crossing      | occupancy is **not** met (a still-armed boundary) |
| `occupancy_arm`     | the boundary crossing      | occupancy **is** met (a newly-armed boundary)     |
| `occupancy_trigger` | the occupancy dwell itself | the required dwell elapses, firing immediately    |

`occupancy_trigger` never consults the boundary. The root collider still spans the zone, but the mode's branch ignores
it, so firing is occupancy-only.

---

## Disarmed segments and ignorable components

`CreateTask` strips zones from segments at corridor depth > 0 because they are visual-only, so an `inspect_prefab_tool`
result where every segment except the first under a corridor has no stimulus zone child is expected. When judging the
hierarchy, ignore `Transform`, `MeshFilter`, `MeshRenderer`, `MeshCollider`, and any `BoxCollider` not paired with a
zone script. Those are either visual geometry or standard Unity components every GameObject carries.
