# Expected Values Reference

Pre-baked values extracted from actual template and prefab files. Use these for verification without reading files.

---

## Templates

| Template               | cm_per_unit | cue_offset | Segments Used                                                              |
|------------------------|-------------|------------|----------------------------------------------------------------------------|
| MF_Aversion_Reward     | 10.0        | 10.0       | Segment_airPuff1, Segment_airPuff2                                         |
| MF_Reward              | 10.0        | 10.0       | Segment_abcdefgh                                                           |
| SSO_Connection         | 10.0        | 10.0       | Segment_abc_40cm, Segment_abca_40cm, Segment_defg_40cm, Segment_defgd_40cm |
| SSO_Connection_Base    | 10.0        | 10.0       | Segment_defg_40cm                                                          |
| SSO_Extension_Shortcut | 10.0        | 10.0       | Segment_abc_40cm, Segment_abdc_40cm                                        |
| SSO_Merging            | 10.0        | 10.0       | Segment_abc_40cm, Segment_agfe_40cm                                        |
| SSO_Merging_Base       | 10.0        | 10.0       | Segment_agfe_40cm                                                          |
| SSO_Shared_Base        | 10.0        | 10.0       | Segment_abc_40cm                                                           |
| SSO_Shortcut_Base      | 10.0        | 10.0       | Segment_abdc_40cm                                                          |

---

## Template Trial Structures

| Template               | Trial        | trigger_type |
|------------------------|--------------|--------------|
| MF_Aversion_Reward     | ABCD         | occupancy    |
| MF_Aversion_Reward     | EFGH         | lick         |
| MF_Reward              | cyclic_8_cue | lick         |
| SSO_Connection         | ABC          | lick         |
| SSO_Connection         | ABCA         | lick         |
| SSO_Connection         | DEFG         | lick         |
| SSO_Connection         | DEFGD        | lick         |
| SSO_Connection_Base    | DEFG         | lick         |
| SSO_Extension_Shortcut | ABC          | lick         |
| SSO_Extension_Shortcut | ABDC         | lick         |
| SSO_Merging            | ABC          | lick         |
| SSO_Merging            | AGFE         | lick         |
| SSO_Merging_Base       | AGFE         | lick         |
| SSO_Shared_Base        | ABC          | lick         |
| SSO_Shortcut_Base      | ABDC         | lick         |

---

## Segment Prefabs

| Segment            | Wall Scale | Zone Type | Zone Z | Reset Z |
|--------------------|------------|-----------|--------|---------|
| Segment_abc_40cm   | 24         | Stimulus  | 18     | 1       |
| Segment_abca_40cm  | 32         | Stimulus  | 18     | 1       |
| Segment_abdc_40cm  | 32         | Stimulus  | 26     | 1       |
| Segment_agfe_40cm  | 32         | Stimulus  | 18     | 1       |
| Segment_defg_40cm  | 32         | Stimulus  | 26     | 1       |
| Segment_defgd_40cm | 40         | Stimulus  | 26     | 1       |
| Segment_abcdefgh   | 40         | Stimulus  | 37.5   | 1       |
| Segment_airPuff1   | 20         | Occupancy | 17.5   | 1       |
| Segment_airPuff2   | 20         | Stimulus  | 17.5   | 1       |

---

## Zone Collider Values

Collider values are read from the segment prefab's PrefabInstance modifications, or inherited from the base zone prefab.

### StimulusTriggerZone Segments

| Segment            | Trigger Size | Trigger Center | Guidance Size | Guidance Center |
|--------------------|--------------|----------------|---------------|-----------------|
| Segment_abc_40cm   | 2.4          | 0              | 0.4           | 1.0             |
| Segment_abca_40cm  | 2.4          | 0              | 0.4           | 1.0             |
| Segment_abdc_40cm  | 2.4          | 0              | 0.4           | 1.0             |
| Segment_agfe_40cm  | 2.4          | 0              | 0.4           | 1.0             |
| Segment_defg_40cm  | 2.4          | 0              | 0.4           | 1.0             |
| Segment_defgd_40cm | 2.4          | 0              | 0.4           | 1.0             |
| Segment_abcdefgh   | 3.5          | 0              | 0.4           | 1.55            |
| Segment_airPuff2   | 3.5          | 0              | 0.4           | 1.55            |

### OccupancyTriggerZone Segments

| Segment          | Boundary Size | Boundary Center | Occupancy Size | Occupancy Center |
|------------------|---------------|-----------------|----------------|------------------|
| Segment_airPuff1 | 3.5           | 0               | 3.5            | -5               |

**Note**: OccupancyTriggerZone has two colliders:
- **Boundary collider** (root): Where stimulus triggers if armed
- **Occupancy collider** (child): Where animal must wait to disarm boundary

---

## Pre-Computed Zone Ranges

### StimulusTriggerZone Segments

Formula: `(zone_z + trigger_center ± trigger_size/2) × cm_per_unity_unit`

| Segment            | Zone Range (cm) | Calculation            |
|--------------------|-----------------|------------------------|
| Segment_abc_40cm   | 168.0 - 192.0   | (18 + 0 ± 1.2) × 10    |
| Segment_abca_40cm  | 168.0 - 192.0   | (18 + 0 ± 1.2) × 10    |
| Segment_abdc_40cm  | 248.0 - 272.0   | (26 + 0 ± 1.2) × 10    |
| Segment_agfe_40cm  | 168.0 - 192.0   | (18 + 0 ± 1.2) × 10    |
| Segment_defg_40cm  | 248.0 - 272.0   | (26 + 0 ± 1.2) × 10    |
| Segment_defgd_40cm | 248.0 - 272.0   | (26 + 0 ± 1.2) × 10    |
| Segment_abcdefgh   | 357.5 - 392.5   | (37.5 + 0 ± 1.75) × 10 |
| Segment_airPuff2   | 157.5 - 192.5   | (17.5 + 0 ± 1.75) × 10 |

### OccupancyTriggerZone Segments

Occupancy range formula: `(zone_z + occupancy_center ± occupancy_size/2) × cm_per_unity_unit`
Boundary range formula: `(zone_z + boundary_center ± boundary_size/2) × cm_per_unity_unit`

| Segment          | Occupancy Range (cm) | Boundary Range (cm) | Calculation                           |
|------------------|----------------------|---------------------|---------------------------------------|
| Segment_airPuff1 | 107.5 - 142.5        | 157.5 - 192.5       | (17.5 + (-5) ± 1.75) × 10 / (17.5 + 0 ± 1.75) × 10 |

**Template mapping for OccupancyTriggerZone:**
- `stimulus_trigger_zone_start/end_cm` → Occupancy range (where animal waits)
- `stimulus_location_cm` → Within boundary range (where stimulus triggers)

---

## Zone Prefab GUIDs

| Prefab                      | GUID                               |
|-----------------------------|------------------------------------|
| ResetZone.prefab            | `78e4c512d0af3c44cbfbb233f81d345f` |
| StimulusTriggerZone.prefab  | `e502aa673cd52774593125318db2aeb3` |
| OccupancyTriggerZone.prefab | `3d9e6b3219444f94e85ebcb948ade18a` |

---

## Cue Offset Verification

All templates use `cue_offset_cm: 10.0`. All prefabs have `reset_z: 1`.
Verification: `1 × 10.0 = 10.0` cm.
