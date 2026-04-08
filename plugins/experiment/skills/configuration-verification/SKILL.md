---
name: verifying-task-templates
description: >-
  Verifies YAML task templates against Unity prefab state. Checks zone positions, trigger types, and
  segment references match between configuration files and prefabs. Use when creating or modifying
  task templates, or when the user mentions template verification.
---

# Task Template Verification

Verifies YAML task templates against Unity prefab state using pre-baked expected values. Also defines naming and
formatting conventions for task template files.

---

## Template Conventions

### File Naming

```
ProjectAbbreviation_TaskDescription.yaml
```

| Abbreviation | Project Name      |
|--------------|-------------------|
| MF           | MaalstroomicFlow  |
| SSO          | StateSpaceOdyssey |

- Use `_Base` suffix for single-segment training configurations
- Capitalize each word in the task description
- Template name and Unity scene name are derived from filename (without `.yaml`)

### Template Header

Each template must begin with a YAML comment header:

```yaml
# Project: [Full project name]
# Purpose: [Single sentence describing the task structure]
# Layout:  [Segment names with cue letters and zone placements]
# Related: [Related template file (parenthetical explanation)]
```

**Multi-line Wrapping**: Align continuation text with the first character after the field name:

```yaml
# Project: MaalstroomicFlow
# Purpose: Extends the base MF_Reward trial structure to also include an aversive stimulus.
# Layout:  Segment ABCD with occupancy zone at cue C and aversive stimulus trigger zone in cue D.
#          Segment EFGH with the rewarding stimulus (water) trigger zone in cue H.
# Related: MF_Reward (the base version of this task that only includes the reward zone)
```

### Header Guidelines

- **Project**: Use the full project name, not the abbreviation
- **Purpose**: Single sentence; use verbs like "Defines", "Extends", "Teaches"
- **Layout**: Include segment names with cue letters, zone types, and stimulus clarifications
- **Related**: Explain the relationship in parentheses

### Inline YAML Comments

Add inline comments to clarify non-obvious values:

```yaml
cues:
  - name: "Gray" # This is a placeholder, the task does not use Gray cues.
    code: 0
    length_cm: 1.0
```

---

## Quick Reference

- **Expected values**: See [EXPECTED_VALUES.md](EXPECTED_VALUES.md) for all pre-baked template and prefab data
- **Templates location**: `Assets/InfiniteCorridorTask/Configurations/*.yaml`
- **Prefabs location**: `Assets/InfiniteCorridorTask/Prefabs/Segment_*.prefab`

---

## Verification Workflow

### Step 1: Compare against pre-baked values

Read [EXPECTED_VALUES.md](EXPECTED_VALUES.md) and compare against the template YAML:

1. Extract from template: `cm_per_unity_unit`, `cue_offset_cm`, segment names, trial zone ranges, `trigger_type`
2. Look up each segment in the Segment Prefabs table
3. Verify zone ranges: `range = (zone_z ± size/2) × cm_per_unity_unit`
4. Verify `trigger_type` matches segment zone type (Stimulus → lick, Occupancy → occupancy)
5. Note any discrepancies for verification in Step 2

### Step 2: Verify discrepancies against actual assets (MANDATORY)

**You MUST verify actual asset files before drawing conclusions when ANY discrepancy is detected.** Never assume
pre-baked values in EXPECTED_VALUES.md are correct without verification.

For each discrepancy:

1. **Read the actual prefab file** using the Read tool:
   - Path: `Assets/InfiniteCorridorTask/Prefabs/{segment_name}.prefab`

2. **Extract ground truth from prefab**:
   - Zone prefab GUID (determines zone type: Stimulus vs Occupancy)
   - Zone position (`m_LocalPosition.z`)
   - Zone collider size (`m_Size.z`)
   - Wall scale (`m_LocalScale.x` on LeftWall/RightWall)

3. **Compare against zone prefab GUIDs** (both use StimulusTriggerZone.cs as root, differentiated by child zone):
   - `e502aa673cd52774593125318db2aeb3` = StimulusTriggerZone.prefab (with GuidanceZone child) → `trigger_type: "lick"`
   - `3d9e6b3219444f94e85ebcb948ade18a` = OccupancyTriggerZone.prefab (with OccupancyZone child) → `trigger_type: "occupancy"`

4. **Determine the source of error**:
   - If prefab matches expected values → template is wrong
   - If prefab differs from expected values → EXPECTED_VALUES.md is outdated
   - If prefab matches template but not expected values → update EXPECTED_VALUES.md

5. **Update the incorrect source**:
   - Fix template if it does not match prefab
   - Update EXPECTED_VALUES.md if pre-baked data is stale

### Step 3: Report results

Report PASS/FAIL only after completing Step 2 verification for any discrepancies:

```
| Template | Status | Notes                            |
|----------|--------|----------------------------------|
| Name     | PASS   |                                  |
| Name     | FAIL   | Reason (verified against prefab) |
```

Include verification evidence when reporting failures:

- Which prefab file was checked
- What GUID or values were found in the prefab
- Whether the template or expected values were incorrect

---

## Validation Checklist

### Schema Compliance
- Cue codes unique (0-255) and names unique
- Segment cue_sequence references valid cues
- Transition probabilities sum to 1.0 (±0.001)
- Trial segment_names reference valid segments
- Trial trigger_type is "lick" or "occupancy"

### Prefab Compliance
- `Prefabs/{segment_name}.prefab` exists for each segment
- Wall scale matches segment length: `sum(cue.length_cm) / cm_per_unit`
- ResetZone z × cm_per_unity_unit = cue_offset_cm
- Zone ranges match template values

### Trigger Type Compliance
- Each trial_structure has a trigger_type field
- trigger_type matches segment zone prefab (both use StimulusTriggerZone.cs as root, differentiated by child zone):
  - StimulusTriggerZone.prefab with GuidanceZone child (GUID: `e502aa673cd52774593125318db2aeb3`) → "lick"
  - OccupancyTriggerZone.prefab with OccupancyZone child (GUID: `3d9e6b3219444f94e85ebcb948ade18a`) → "occupancy"

---

## Extraction Patterns

When reading Unity `.prefab` files to update pre-baked values:

```yaml
# Wall scale (segment length in Unity units)
m_Name: LeftWall
m_LocalScale: {x: 24, y: 2.5, z: 1}  # x value is wall scale

# Zone position (after zone prefab GUID reference)
m_LocalPosition: {x: 0, y: 0, z: 18}  # z value is zone position

# Zone collider size
m_Size: {x: 2, y: 2, z: 2.4}  # z value is zone size
```

**Zone range formulas:**
- StimulusTriggerZone: `(zone_z + collider_center ± collider_size/2) × cm_per_unity_unit`
- OccupancyTriggerZone occupancy range: `(zone_z + occupancy_center ± occupancy_size/2) × cm_per_unity_unit`
- OccupancyTriggerZone boundary range: `(zone_z + boundary_center ± boundary_size/2) × cm_per_unity_unit`

Where collider values are read from the prefab's BoxCollider `m_Center.z` and `m_Size.z` fields.

---

## Zone Behavior Reference

Both zone prefabs use `StimulusTriggerZone.cs` as the root script. The behavior mode is determined by the child zone:

### Lick Mode (StimulusTriggerZone.prefab)

**Prefab structure:**
- Root: StimulusTriggerZone (trigger collider)
  - Child: GuidanceRegion (GuidanceZone.cs)

**Runtime behavior:**
- `requireLick=true` (guidance disabled): Animal must lick inside the trigger zone collider to receive stimulus
- `requireLick=false` (guidance enabled): Stimulus delivered when animal reaches GuidanceZone OR licks in trigger zone

**Template fields:**
- `stimulus_trigger_zone_start/end_cm`: Range where licking triggers stimulus
- `stimulus_location_cm`: Position within the trigger zone (should be inside the zone range)

### Occupancy Mode (OccupancyTriggerZone.prefab)

**Prefab structure:**
- Root: StimulusTriggerZone (boundary collider)
  - Child: OccupancyRegion (OccupancyZone.cs, collider offset by `m_Center.z`)
    - Child: OccupancyGuidanceRegion (OccupancyGuidanceZone.cs)

**Runtime behavior:**
1. Animal enters OccupancyRegion (offset from boundary by collider center) → occupancy timer starts
2. If animal stays for `occupancyDurationMs` → boundary **disarmed**, animal passes safely
3. If animal reaches boundary while **armed** (didn't wait long enough) → stimulus triggers

**Template fields:**
- `stimulus_trigger_zone_start/end_cm`: Occupancy waiting range (calculated from OccupancyRegion collider)
- `stimulus_location_cm`: Boundary position where stimulus triggers (calculated from root collider)

**IMPORTANT**: For occupancy zones, `stimulus_location_cm` is at the boundary position, NOT within
the occupancy waiting range. The waiting range and trigger location are intentionally different—the
animal waits in one area, then the stimulus triggers if they reach the boundary without meeting the
occupancy requirement.

---

## Troubleshooting

### Zone Range Mismatch

**Symptom**: Template zone range does not match calculated prefab range.

**Common causes**:
1. Wrong `cm_per_unity_unit` value in template
2. Zone prefab position changed in Unity Editor
3. Zone collider size modified
4. Pre-baked values in EXPECTED_VALUES.md are outdated

**Resolution**:
1. Read the actual prefab file using `Read` tool
2. Extract `m_LocalPosition.z` and `m_Size.z` values
3. Recalculate range using appropriate formula
4. Update either template or EXPECTED_VALUES.md

### Missing Segment Prefab

**Symptom**: Template references segment that does not exist in Prefabs directory.

**Resolution**:
1. Verify segment name spelling in template matches prefab filename exactly
2. Check if prefab was renamed or moved
3. Create missing prefab using CreateTask editor tool if needed

### Trigger Type Mismatch

**Symptom**: Template `trigger_type` does not match expected zone prefab type.

**You MUST verify the actual prefab file before concluding which value is wrong.**

**Resolution**:

1. Read the actual segment prefab file (e.g., `Segment_airPuff1.prefab`)
2. Find the zone prefab GUID in the PrefabInstance section (look for `m_SourcePrefab` or `guid:` references)
3. Compare GUID against known zone prefabs (both use StimulusTriggerZone.cs as root, differentiated by child zone):
   - `e502aa673cd52774593125318db2aeb3` = StimulusTriggerZone.prefab (GuidanceZone child) → `"lick"`
   - `3d9e6b3219444f94e85ebcb948ade18a` = OccupancyTriggerZone.prefab (OccupancyZone child) → `"occupancy"`
4. Determine what needs updating:
   - If prefab GUID matches EXPECTED_VALUES.md but not template → fix template
   - If prefab GUID differs from EXPECTED_VALUES.md → update EXPECTED_VALUES.md first, then verify template

### Pre-Baked Data Outdated

**Symptom**: Multiple verification failures after prefab or template changes.

**Resolution**:
1. Re-extract values from source files using Glob and Read tools
2. Update all affected tables in EXPECTED_VALUES.md
3. Re-run verification to confirm fixes

---

## Template Verification Checklist

Copy this checklist when reviewing YAML task templates:

```
Task Template Compliance:
- [ ] Filename follows `ProjectAbbreviation_TaskDescription.yaml` format
- [ ] Header includes Project, Purpose, Layout, Related fields
- [ ] Project uses full name (not abbreviation)
- [ ] Purpose is single sentence with verb (Defines, Extends, Teaches)
- [ ] Multi-line values align with first character after field name
- [ ] Inline comments explain non-obvious values
- [ ] Cue codes unique (0-255) and names unique
- [ ] Segment cue_sequence references valid cues
- [ ] Transition probabilities sum to 1.0 (±0.001)
- [ ] Trial segment_names reference valid segments
- [ ] Trial trigger_type is "lick" or "occupancy"
- [ ] All referenced segment prefabs exist
- [ ] Zone positions verified against prefab state
```
