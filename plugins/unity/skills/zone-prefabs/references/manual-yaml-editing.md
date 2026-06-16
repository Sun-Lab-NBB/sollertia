# Manual zone-prefab YAML editing

Reference for hand-editing a zone prefab's YAML. Needed only when `clone_zone_prefab_tool` cannot express the
operation it leaves to a future version (adding or removing a region), or when inspecting a surprising clone
result. For rename, script swaps, and field overrides, use the tool (see `SKILL.md` "Manufacturing a zone
prefab"). After editing the prefab per the steps below, hand off the downstream wiring per `SKILL.md`
"Step 7: Hand off the remaining wiring".

---

## Invariants you MUST preserve

Copying a template imports every invariant for free. You MUST NOT modify the following — they
are referenced by `CreateTask.PlaceInteractionZone` / `PlaceOccupancyZone` at task generation, by
`StimulusTriggerZone.cs` at runtime, or by both.

### On the root GameObject

| Field                              | Required value                                                                |
|------------------------------------|-------------------------------------------------------------------------------|
| `Transform.m_LocalPosition`        | `{x: 0, y: 0.505, z: 0}` (the `ZoneVerticalOffset`)                           |
| `MeshFilter.m_Mesh`                | `{fileID: 10210, guid: 0000000000000000e000000000000000}` (built-in Quad)     |
| `MeshRenderer.m_Materials[0]`      | `{fileID: 2100000, guid: 0517064f81d2dc54fac8fa8c97538189}` (`TargetMat.mat`) |
| `MeshCollider.m_Mesh`              | Same built-in Quad as the MeshFilter                                          |
| `MeshCollider.m_IsTrigger`         | `0` (false — this is the physical boundary)                                   |
| `BoxCollider.m_IsTrigger`          | `1` (true — this is the StimulusTriggerZone detection)                        |
| `BoxCollider.m_Size` / `m_Center`  | Placeholder — `ConfigureRootZoneCollider` overwrites                          |
| `StimulusTriggerZone.showBoundary` | `0` (false; `CreateTask` sets it per trial at generation)                     |
| `StimulusTriggerZone.isActive`     | `0` (false; `ResetZone.ResetState` activates at lap start)                    |

### On every modifier-zone child

| Field                             | Required value                                      |
|-----------------------------------|-----------------------------------------------------|
| `Transform.m_LocalPosition`       | `{x: 0, y: 0, z: 0}`                                |
| `BoxCollider.m_IsTrigger`         | `1` (true)                                          |
| `BoxCollider.m_Size` / `m_Center` | Placeholder — `Place*Zone` overwrites at generation |
| Exactly one `MonoBehaviour`       | The modifier script (e.g., `GuidanceZone`)          |

### Hierarchy integrity

- Every entry in a parent's `m_Children: [{fileID: X}]` must match the `m_Father: {fileID: Y}` on
  the referenced child's Transform.
- fileIDs are scoped per-prefab — renaming a region needs no fileID change. Adding a new region
  requires picking a fresh 18–19-digit integer that does not collide with any existing fileID in
  the same prefab.
- The script GUID on every `MonoBehaviour` (`m_Script: {fileID: 11500000, guid: ...}`) must match
  the `guid:` value in the script's `.cs.meta` file.

---

## Modifier script GUID lookup

Every script GUID lives in the script's `.cs.meta` companion file. To resolve a script's GUID:

```text
Read("Assets/InfiniteCorridorTask/Scripts/<ScriptName>.cs.meta")
```

Find the line `guid: <32-character hex string>`. That value goes into the prefab's
`m_Script.guid` field.

The four scripts currently used by the two committed templates have the following GUIDs (verify
against the `.cs.meta` files before each edit — Unity GUIDs are sticky, but verification catches
copy-paste errors in advance):

| Script class            | `.cs.meta` path                                                     | GUID                               |
|-------------------------|---------------------------------------------------------------------|------------------------------------|
| `StimulusTriggerZone`   | `Assets/InfiniteCorridorTask/Scripts/StimulusTriggerZone.cs.meta`   | `72389065db4262222b18469cd7662432` |
| `GuidanceZone`          | `Assets/InfiniteCorridorTask/Scripts/GuidanceZone.cs.meta`          | `d99710621d4dc286b93af3d3946e3440` |
| `OccupancyZone`         | `Assets/InfiniteCorridorTask/Scripts/OccupancyZone.cs.meta`         | `5ac4de8c500fd94d192243204f3a2a99` |
| `OccupancyGuidanceZone` | `Assets/InfiniteCorridorTask/Scripts/OccupancyGuidanceZone.cs.meta` | `dcab3a92479672720b736c7ef24fcacf` |

For a newly authored script, read its `.cs.meta` to extract the freshly minted GUID.

---

## Step-by-step YAML edits

### Step 1: Pick the template

| New zone has...                                     | Copy this template                                            |
|-----------------------------------------------------|---------------------------------------------------------------|
| One modifier region, flat                           | `StimulusTriggerZone.prefab`                                  |
| One modifier region with a nested guidance modifier | `OccupancyTriggerZone.prefab`                                 |
| Two or more sibling modifier regions                | `StimulusTriggerZone.prefab` (then add siblings — see Step 5) |

### Step 2: Read the source prefab

Use `Read` to load the entire source prefab YAML:

```text
Read("Assets/InfiniteCorridorTask/Prefabs/<Source>.prefab")
```

Keep the file contents in working memory — every subsequent edit operates on this exact text.

### Step 3: Write to the new path

Use `Write` to copy the source contents to the target path. Target paths must live under
`Assets/InfiniteCorridorTask/Prefabs/` and the filename should match the new `TriggerType` (e.g.,
`MyNewTriggerZone.prefab`):

```text
Write("Assets/InfiniteCorridorTask/Prefabs/MyNewTriggerZone.prefab", <contents>)
```

The `m_Name` values inside the copied YAML still reference the source's GameObject names
(`OccupancyTriggerZone`, `OccupancyRegion`, etc.) — patching them is part of Step 4.

### Step 4: Edit the new prefab

Apply these `Edit` calls in order. Each operates on the new prefab file you just wrote.

#### 4a. Rename the root

Replace the root GameObject's `m_Name` to match the new prefab filename basename:

```text
Edit(
  file_path="Assets/InfiniteCorridorTask/Prefabs/MyNewTriggerZone.prefab",
  old_string="  m_Name: OccupancyTriggerZone",
  new_string="  m_Name: MyNewTriggerZone"
)
```

The root name is cosmetic (`BuildSegmentPrefabs` resolves the prefab by filename), but match the
filename for hierarchy clarity.

#### 4b. Swap modifier script GUIDs

For each `MonoBehaviour` whose script you are replacing, edit the surrounding `m_Script` block.
Include enough context to make the edit unique (the `m_GameObject` fileID above the
`m_EditorHideFlags`, or the field values below the GUID line, both work):

```text
Edit(
  file_path="Assets/InfiniteCorridorTask/Prefabs/MyNewTriggerZone.prefab",
  old_string=|
    m_Script: {fileID: 11500000, guid: 5ac4de8c500fd94d192243204f3a2a99, type: 3}
    m_Name: 
    m_EditorClassIdentifier: Assembly-CSharp::OccupancyZone
    occupancyDurationMs: 1000
  ,
  new_string=|
    m_Script: {fileID: 11500000, guid: <new-script-guid>, type: 3}
    m_Name: 
    m_EditorClassIdentifier: Assembly-CSharp::MyNewZone
    myDurationMs: 2000
)
```

The `m_EditorClassIdentifier` is informational, but updating it preserves diff readability when
the prefab is opened in the Editor later. The committed templates leave the field empty on the
root MonoBehaviour in both prefabs and populate it on modifier MonoBehaviours
(`Assembly-CSharp::OccupancyZone`, `Assembly-CSharp::OccupancyGuidanceZone`) in
`OccupancyTriggerZone.prefab`; preserve the source's polarity when editing.

Repeat for every modifier `MonoBehaviour` you are replacing. A root or modifier script may be
replaced with a **subclass** of the original (`StimulusTriggerZone`, `OccupancyZone`,
`OccupancyGuidanceZone`) without breaking `ResetZone.Start`'s typed `FindObjectsByType<T>`
discovery; a fully unrelated `IResettable` class is invisible to it and needs an explicit
registration in `ResetZone.cs` (see `SKILL.md` "Step 7: Hand off the remaining wiring").

#### 4c. Rename modifier regions

For each region whose `m_Name` you want to change, edit the GameObject block. Region names are
cosmetic but should match the new zone's terminology:

```text
Edit(
  file_path="Assets/InfiniteCorridorTask/Prefabs/MyNewTriggerZone.prefab",
  old_string="  m_Name: OccupancyRegion",
  new_string="  m_Name: MyNewRegion"
)
```

#### 4d. Override field defaults

Field defaults live inside each `MonoBehaviour` block under the `m_EditorClassIdentifier` line.
Adjust them in place. Adding a new field that the new script declares is also fine — the YAML
deserializer reads field names directly:

```text
Edit(
  file_path="Assets/InfiniteCorridorTask/Prefabs/MyNewTriggerZone.prefab",
  old_string="  occupancyDurationMs: 1000",
  new_string="  myDurationMs: 2000"
)
```

Removing a field is also fine — Unity will fall back to the script's declared default value.

### Step 5: Add or remove modifier regions

Adding a new sibling or nested region requires appending three YAML blocks (GameObject, Transform,
BoxCollider, MonoBehaviour) and wiring the new fileIDs into the parent's `m_Children` list.

When adding a region:

1. Pick four fresh fileIDs that do not collide with any existing fileID in the prefab. Use random
   18–19-digit integers (e.g., `2839475610293847501`).
2. Append the GameObject, Transform, MonoBehaviour, and BoxCollider blocks at the end of the file,
   following the exact structure shown in the canonical templates (read the interaction template for the
   `GuidanceRegion` shape).
3. Add the new GameObject's Transform fileID to the parent's `m_Children` list:

   ```yaml
   m_Children:
   - {fileID: 3732166563701906224}    # existing
   - {fileID: 2839475610293847501}    # new
   ```

4. Set the new Transform's `m_Father` to the parent's Transform fileID:

   ```yaml
   m_Father: {fileID: 1872135180607641772}
   ```

The simpler path is to **start from the closest structural match** and only add or remove regions
when neither template's hierarchy fits.

When removing a region: delete its four YAML blocks (GameObject + Transform + MonoBehaviour +
BoxCollider) and remove its Transform fileID from the parent's `m_Children` list.

### Step 6: Validate via inspect_prefab_tool

Run `inspect_prefab_tool` (owned by `/task-prefabs`) against the new prefab. The tool returns the
hierarchy Unity actually loads — if the YAML is malformed, the tool errors out before producing a
hierarchy, which is a stronger signal than visual inspection.

```text
inspect_prefab_tool(prefab_path="Assets/InfiniteCorridorTask/Prefabs/MyNewTriggerZone.prefab")
```

Verify:
- The root has a `StimulusTriggerZone` component (in the `components` list) and a
  `BoxCollider`.
- Each modifier region has the expected new script in `components`.
- `collider_size` and `collider_center` on the root are non-zero (size is a placeholder; presence
  matters more than the value).
- The hierarchy depth and child ordering match the template you copied.

If `inspect_prefab_tool` fails, the YAML is broken. Re-read the file, compare to the source
template via `git diff`, and fix the structural divergence before continuing.
