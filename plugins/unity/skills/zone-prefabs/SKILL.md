---
name: zone-prefabs
description: >-
  Manufactures new hand-authored trigger zone prefabs for sollertia-unity-tasks by copying one of
  the two canonical templates (`StimulusTriggerZone.prefab` for lick mode, `OccupancyTriggerZone.prefab`
  for occupancy mode) and rewriting the MonoBehaviour script GUIDs, region names, and field defaults.
  Use when adding a new `TriggerType` member or designing a new stimulus-zone variant that mixes
  existing modifier zones in a new configuration.
user-invocable: false
---

# Sollertia Unity zone prefabs

Authors new hand-authored trigger zone prefabs for `sollertia-unity-tasks` by copying one of the
two committed templates, swapping the modifier scripts and field defaults, and validating the
result with `inspect_prefab_tool` — instead of constructing prefab YAML from scratch.

---

## Scope

**Covers:**
- Manufacturing a new trigger zone prefab under `Assets/InfiniteCorridorTask/Prefabs/`
- Resolving MonoBehaviour script GUIDs from `.cs.meta` files
- Swapping modifier scripts (`GuidanceZone`, `OccupancyZone`, `OccupancyGuidanceZone`, or new
  `IResettable` implementations) on a copied prefab
- Renaming modifier regions and overriding their serialized field defaults
- Adding or removing nested modifier zones while preserving the `m_Children` ↔ `m_Father` pairing
- Validating the new prefab via `inspect_prefab_tool`
- Identifying every downstream wiring step required to make the new prefab usable

**Does not cover:**
- Task prefab generation from YAML templates (see `/task-prefabs`)
- Modifications to the `CreateTask` pipeline that consumes zone prefabs (see `/task-generator`)
- Adding a new `TriggerType` member to the shared-assets registry (see assets plugin's
  `/library-extension`)
- Authoring the new MonoBehaviour script itself (see `/csharp-style` in the automation plugin)
- Editing protected hand-authored assets — `/task-generator` "Required shared assets" enumerates
  the full set (zone base prefabs, shared materials, scene base template). They are source
  templates and shared assets that `CreateTask` and the generated prefabs reference, and they
  must remain untouched

---

## Why copy-and-edit, not generate-from-spec

Both zone prefabs share a fixed structural skeleton:

- A root GameObject carrying a `Transform`, `MeshFilter` (built-in Quad), `MeshRenderer`
  (`TargetMat.mat`), `MeshCollider` (non-trigger, same Quad), `BoxCollider` (trigger), and a
  `StimulusTriggerZone` script.
- One or more modifier-zone children, each carrying a `Transform`, a `BoxCollider` (trigger), and
  exactly one `MonoBehaviour` from `SL.Tasks`.
- A `Transform.localPosition` of `(0, 0.505, 0)` on the root (the `ZoneVerticalOffset` defined in
  `CreateTask.cs`) and `(0, 0, 0)` on every modifier.
- Placeholder `BoxCollider` sizes — `CreateTask.PlaceLickZone` and `CreateTask.PlaceOccupancyZone`
  overwrite them at task generation time, so the prefab's stored values are not authoritative.

The only fields that vary between trigger zone variants are the script GUIDs on each
`MonoBehaviour`, the GameObject `m_Name` values, and the serialized field defaults on each modifier
script. Everything else is identical. Manufacturing a new variant from a spec language would
require redescribing the entire skeleton on every call; copying the closest existing template and
patching the three fields above is faster, lower-risk, and lets the agent rely on standard
file-editing tools (`Read`, `Edit`, `Write`) instead of a custom MCP tool.

---

## Pre-flight checklist

You MUST verify all the following before manufacturing a new zone prefab:

1. **The new modifier script exists.** Author the new `MonoBehaviour` under
   `Assets/InfiniteCorridorTask/Scripts/` before producing the prefab — Unity creates the
   `.cs.meta` file (and the stable GUID inside it) on the next Editor refresh, and the prefab YAML
   needs that GUID to wire the script in.
2. **The new script lives in the `SL.Tasks` namespace** (or a subnamespace), inherits from
   `MonoBehaviour`, and implements `IResettable` if it carries per-lap state. Follow the existing
   `OccupancyZone` / `GuidanceZone` pattern in `Assets/InfiniteCorridorTask/Scripts/`.
3. **Style compliance.** Invoke `/csharp-style` (automation plugin) before writing the new script,
   and run CSharpier on the modified C# files before committing.
4. **Unity Editor running** with `McpBridge` reachable. The validation step relies on
   `inspect_prefab_tool`, which fails without the bridge. Invoke `/unity-mcp-environment-setup`
   first if the bridge is offline.

If any of the above is unmet, stop and resolve it before continuing.

---

## Canonical templates

Both templates live under `Assets/InfiniteCorridorTask/Prefabs/` and are committed to source
control. Read whichever one matches the target zone shape, then write the modified contents to a
new path.

### Lick template (`StimulusTriggerZone.prefab`)

Hierarchy:

```text
StimulusTriggerZone               ← root, 6 components
└── GuidanceRegion                ← single child, 3 components
```

Use this template when the new zone needs exactly one modifier region that reports the animal's
arrival to `StimulusTriggerZone`. Examples: a new "approach detector" that triggers stimulus on
zone entry without occupancy timing.

### Occupancy template (`OccupancyTriggerZone.prefab`)

Hierarchy:

```text
OccupancyTriggerZone              ← root, 6 components
└── OccupancyRegion               ← child, 3 components
    └── OccupancyGuidanceRegion   ← grandchild, 3 components
```

Use this template when the new zone needs a nested modifier (a child of a modifier). Examples: a
new "must-wait-then-acknowledge" pattern where the inner zone reads state off the outer zone via
`GetComponentInParent`.

---

## Invariants you MUST preserve

Copying a template imports every invariant for free. You MUST NOT modify the following — they
are referenced by `CreateTask.PlaceLickZone` / `PlaceOccupancyZone` at task generation, by
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

## Workflow

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
registration in `ResetZone.cs` (see Step 7).

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

### Step 5: Add or remove modifier regions (optional)

Adding a new sibling or nested region requires appending three YAML blocks (GameObject, Transform,
BoxCollider, MonoBehaviour) and wiring the new fileIDs into the parent's `m_Children` list.

When adding a region:

1. Pick four fresh fileIDs that do not collide with any existing fileID in the prefab. Use random
   18–19-digit integers (e.g., `2839475610293847501`).
2. Append the GameObject, Transform, MonoBehaviour, and BoxCollider blocks at the end of the file,
   following the exact structure shown in the canonical templates (read the lick template for the
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

### Step 7: Hand off the remaining wiring

The new prefab is unreferenced after Step 6. The full cross-cutting recipe is split three ways:

| Slice                                                | Owning skill                                                    |
|------------------------------------------------------|-----------------------------------------------------------------|
| Python `TriggerType` enum + registry parity          | assets plugin `/library-extension` (Adding a new `TriggerType`) |
| `CreateTask` pipeline edits + `DeleteProtectedPaths` | `/task-generator` (Adding a new zone trigger type)              |
| Hand-authored prefab (this skill)                    | Steps 1–6 above                                                 |

The only `ResetZone` consideration that lives in this skill (because it depends on the new
modifier script's class identity) is:

- **Register new `IResettable` classes in `ResetZone`.** `ResetZone.Start` discovers resettables by
  calling `FindObjectsByType<T>` on `StimulusTriggerZone`, `OccupancyZone`, and
  `OccupancyGuidanceZone` only — subclasses of those three are covered polymorphically, but a
  standalone `IResettable` needs an explicit `FindObjectsByType<NewZone>` line added to
  `ResetZone.cs`, or per-lap state for the new zone is never reset.

Once `/library-extension`, `/task-generator`, and this skill's bullets have all landed, the new
prefab is usable by any YAML template that declares the new `trigger_type`.

---

## Worked examples

End-to-end walkthroughs of the two non-trivial zone-prefab authoring patterns live in
[references/worked-examples.md](references/worked-examples.md):

- **Example A:** Inverting an `OccupancyTriggerZone` from "disable trigger" (aversive) to "enable
  trigger" (rewarding) by writing a new `RewardOccupancyZone` script that preserves the
  `boundaryDisarmed` field name with flipped polarity, so `StimulusTriggerZone` and
  `OccupancyGuidanceZone` keep working unchanged.
- **Example B:** Building a brand-new compound `SpeedLickTriggerZone` that gates lick-triggered
  stimulus on the animal's traversal speed through an upstream speed-test region — covers a new
  parent script, a new sibling-region script, and a new `PlaceSpeedLickZone` placement helper.

Load that file when you actually need to extend the zone vocabulary; the workflow above (Steps
1–7) is enough for routine variants.

---

## Anti-patterns

- **Editing the canonical templates in place.** `StimulusTriggerZone.prefab` and
  `OccupancyTriggerZone.prefab` are protected via `McpBridge.DeleteProtectedPaths` for a reason:
  every generated task prefab references them by exact filename. Always copy to a new path.
- **Constructing prefab YAML from scratch.** Unity prefabs have implicit serialization rules
  (component ordering, fileID scoping, default field handling) that are easy to violate when
  authoring from a blank file. The two committed templates already satisfy every rule; copy them.
- **Reusing fileIDs across prefabs.** fileIDs are scoped per-asset; reusing the source template's
  IDs in the copy is fine. But never reuse an ID that already appears in the same prefab — Unity
  will silently merge the components.
- **Modifying root invariants.** Changing the `MeshFilter` mesh, the `MeshRenderer` material, the
  `MeshCollider` setup, or `Transform.localPosition.y` on the root breaks the visual boundary or
  the trigger detection geometry. Override fields on the modifier scripts instead.
- **Hand-sizing the root BoxCollider.** `ConfigureRootZoneCollider` in `CreateTask.cs` overwrites
  the root collider's size and center at task generation time. Spending edit cycles tuning these
  values is wasted effort.
- **Setting `isActive: true` on the root.** The root `StimulusTriggerZone` script must start
  inactive; `ResetZone.ResetState` activates it at the start of each lap. Setting it to `true` at
  authoring time causes the zone to fire on the first frame before the actor is in position.
- **Skipping `inspect_prefab_tool`.** Unity will silently load broken prefabs into the Editor but
  produce import errors that are easy to miss. `inspect_prefab_tool` returns a structured failure
  the agent can act on.
- **Forgetting `DeleteProtectedPaths`.** A new hand-authored prefab without protection will be
  removed by a future `delete_asset_tool` cleanup pass. Always update `McpBridge.cs` in the
  same PR that introduces the prefab.

---

## Failure modes

| Symptom                                                                              | Cause                                                                         | Resolution                                                                                                                                                               |
|--------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `inspect_prefab_tool` returns "Prefab not found at: …"                               | Prefab was not saved, or path is wrong                                        | Re-run Step 3; confirm the `Write` call returned success                                                                                                                 |
| `inspect_prefab_tool` returns success but no `StimulusTriggerZone` component on root | Root MonoBehaviour was accidentally removed or its script GUID is invalid     | Re-read the source template; restore the root MonoBehaviour block verbatim                                                                                               |
| `inspect_prefab_tool` returns success but the modifier region is missing             | Child block was deleted but parent's `m_Children` still references its fileID | Remove the orphan fileID from `m_Children` or restore the deleted blocks                                                                                                 |
| Hierarchy returned by `inspect_prefab_tool` is flat (no children)                    | `m_Father` ↔ `m_Children` symmetry was broken                                 | Verify every child's Transform `m_Father` matches the parent Transform's fileID, and that the parent's `m_Children` list contains the child's Transform fileID           |
| Modifier script defaults look correct but the runtime behavior is wrong              | `m_Script.guid` references the wrong script                                   | Re-read the target script's `.cs.meta` and confirm the GUID; the `m_EditorClassIdentifier` line is informational and may lag the real script class until Unity reimports |
| Unity Editor reports "missing script" when opening the new prefab                    | Either the script does not exist yet, or the GUID is malformed                | Confirm the `.cs` and `.cs.meta` files exist under `Scripts/`; ensure the GUID is 32 hex characters with no whitespace                                                   |
| New prefab disappears after a cleanup pass                                           | Prefab not added to `McpBridge.DeleteProtectedPaths`                          | Add the path to the protected set and recover the prefab from git                                                                                                        |
| Task generation succeeds but the new zone never triggers                             | `BuildSegmentPrefabs` does not route to the new prefab                        | Hand off to `/task-generator` — add a `trigger_type` branch and a `Place...Zone` helper                                                                                  |

---

## Related skills

| Skill                                        | Relationship                                                               |
|----------------------------------------------|----------------------------------------------------------------------------|
| `/task-prefabs` (this plugin)                | Provides `inspect_prefab_tool` used for Step 6 validation                  |
| `/task-generator` (this plugin)              | Reference for `BuildSegmentPrefabs`, `Place...Zone`, and validator updates |
| `/task-parameters` (this plugin)             | Reference if the new zone exposes Inspector-driven fields                  |
| `/task-scenes` (this plugin)                 | Consumer — places the regenerated task prefab into a scene                 |
| `/play-mode` (this plugin)                   | Consumer — exercises the new zone at runtime                               |
| `/mqtt-contract` (this plugin)               | Reference if the new modifier publishes or subscribes to MQTT topics       |
| `/unity-mcp-environment-setup` (this plugin) | Run first if `inspect_prefab_tool` cannot reach the Unity Editor           |
| assets plugin `/library-extension`           | Required for new `TriggerType` member and registry parity check            |
| automation plugin `/csharp-style`            | Required when authoring the new modifier script and editing C# wiring      |
| automation plugin `/commit`                  | Run after the prefab, script, and wiring changes are ready to commit       |
| experiment plugin `/vr-driver-interface`     | Host pairs `DecomposedTrials.trigger_types` with these trigger zones       |

---

## Verification checklist

You MUST verify this checklist before submitting any new or modified hand-authored zone prefab.

```text
Zone Prefabs Compliance:
- [ ] The new modifier script exists under Assets/InfiniteCorridorTask/Scripts/ with a valid
      .cs.meta and a stable GUID
- [ ] The new prefab path is under Assets/InfiniteCorridorTask/Prefabs/ and the filename matches
      the intended TriggerType naming
- [ ] The new prefab's root m_Name matches the prefab filename basename
- [ ] The new prefab's root Transform.localPosition is exactly (0, 0.505, 0)
- [ ] The new prefab's root retains MeshFilter (built-in Quad), MeshRenderer (TargetMat.mat),
      MeshCollider (non-trigger), BoxCollider (trigger), and StimulusTriggerZone
- [ ] StimulusTriggerZone.showBoundary is 0 and StimulusTriggerZone.isActive is 0 on the root
- [ ] Every modifier MonoBehaviour's m_Script.guid matches the corresponding .cs.meta guid
- [ ] Every modifier region's Transform.localPosition is (0, 0, 0)
- [ ] Every modifier region's BoxCollider is a trigger
- [ ] m_Children ↔ m_Father pairs match for every parent-child relationship in the prefab
- [ ] No fileID appears more than once in the same prefab
- [ ] inspect_prefab_tool returns success and the reported hierarchy matches the intended
      template-derived shape
- [ ] McpBridge.DeleteProtectedPaths includes the new prefab path
- [ ] BuildSegmentPrefabs has a branch that instantiates the new prefab for the new trigger_type
- [ ] ConfigLoader.ValidateTemplate accepts the new trigger_type literal
- [ ] ResetZone.Start finds the new IResettable (subclasses of the three known types are covered
      polymorphically; a standalone class needs an explicit FindObjectsByType registration)
- [ ] /library-extension was invoked on the assets-plugin side to register the new TriggerType
      member and run the import-time parity check
- [ ] CSharpier ran cleanly on the modified C# files (the new script, McpBridge.cs, ConfigLoader.cs,
      CreateTask.cs)
```
