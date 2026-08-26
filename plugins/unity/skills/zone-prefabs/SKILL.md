---
name: zone-prefabs
description: >-
  Manufactures new trigger zone prefabs for sollertia-virtual-reality with the `clone_zone_prefab_tool` MCP
  tool. The tool copies one of the two canonical base prefabs, then swaps the modifier scripts, region names,
  and field defaults through Unity's serialization layer. Use when adding a new `TriggerType` member or
  designing a new stimulus-zone variant that mixes existing modifier zones in a new configuration.
user-invocable: false
---

# Sollertia Unity zone prefabs

Authors new trigger zone prefabs for `sollertia-virtual-reality` with the `clone_zone_prefab_tool` MCP tool: it copies
one of the two committed base prefabs, swaps the modifier scripts and field defaults through Unity's serialization
layer, and returns the resulting hierarchy for validation.

---

## Scope

**Covers:**
- Manufacturing a new trigger zone prefab under `Assets/InfiniteCorridorTask/Prefabs/`
- Resolving MonoBehaviour script GUIDs from `.cs.meta` files
- Swapping modifier scripts (`GuidanceZone`, `OccupancyZone`, `OccupancyGuidanceZone`, or new `IResettable`
  implementations) on a copied prefab
- Renaming modifier regions and overriding their serialized field defaults
- Adding or removing nested modifier zones while preserving the `m_Children` ↔ `m_Father` pairing
- Validating the new prefab via `inspect_prefab_tool`, a read-only **natural share** any skill may call
- Identifying every downstream wiring step required to make the new prefab usable

**Does not cover:**
- Task prefab generation from YAML templates (see `/task-prefabs`)
- Modifications to the `CreateTask` pipeline that consumes zone prefabs (see `/task-generator`)
- Adding a new `TriggerType` member to the shared-assets registry (see assets plugin's `assets:library-extension`)
- Authoring the new MonoBehaviour script itself, covered by `automation:csharp-style` (ataraxis marketplace)
- Editing protected hand-authored assets, where `/task-generator` "Required shared assets" enumerates the full set (the
  two zone base prefabs, the padding prefab, the four shared materials, and the scene base template). They are source
  templates and shared assets that `CreateTask` and the generated prefabs reference, and they must remain untouched

---

## Manufacturing a zone prefab

The `clone_zone_prefab_tool` MCP tool (owned **exclusively** by this skill, relayed to `McpBridge.CloneZonePrefab`)
performs the whole prefab-authoring step in one call. It copies a canonical base prefab, renames regions, swaps the root
and region modifier scripts for new compiled `MonoBehaviour` types, applies serialized field overrides, and returns the
resulting hierarchy in the same shape as `inspect_prefab_tool`. Unity assigns the fileIDs, script references, and
`m_Children` / `m_Father` wiring through its serialization layer, so the result is correct by construction and validates
in the same call.

A successful call returns three keys:

| Key                  | Contents                                                                        |
|----------------------|---------------------------------------------------------------------------------|
| `destination_prefab` | The project-relative path of the new prefab                                     |
| `hierarchy`          | The new prefab's recursive GameObject tree, same shape as `inspect_prefab_tool` |
| `warning`            | The remaining wiring steps, always present on success                           |

The `warning` text is authored by `McpBridge.CloneZonePrefab` and names the four code edits [Step
7](#step-7-hand-off-the-remaining-wiring) hands off. They are the `DeleteProtectedPaths` entry, the `Place...Zone`
branch in `CreateTask`, the `trigger_type` literal in `ConfigLoader`, and the `TriggerType` member in
`sollertia-shared-assets`. Treat it as an authoritative to-do list rather than incidental output, and read Step 7 for
the rest (the per-lap-reset registration and the test-suite updates).

```text
clone_zone_prefab_tool(
    source_prefab="Assets/InfiniteCorridorTask/Prefabs/StimulusTriggerZone.prefab",
    destination_prefab="Assets/InfiniteCorridorTask/Prefabs/SpeedInteractionTriggerZone.prefab",
    root_script="SpeedInteractionTriggerZone",
    regions=[{"match": "GuidanceRegion", "rename": "SpeedTestRegion", "script": "SpeedZone",
              "fields": {"targetSpeedCmPerSec": 20, "toleranceCmPerSec": 5}}],
)
```

The tool resolves every script name before it writes anything, so a typo or an uncompiled script fails before any asset
is created, and it rolls the asset back if a later edit fails. The new prefab's root takes the destination filename. The
tool enforces the same guarantees this skill applies by hand:

- The source must be one of the two canonical base prefabs.
- `root_script` must name a concrete `MonoBehaviour` deriving from `StimulusTriggerZone`. Region scripts may be any
  concrete `MonoBehaviour`.
- The destination must sit under `Assets/InfiniteCorridorTask/Prefabs/`, end with `.prefab`, carry no `..` segment, be
  project-relative rather than absolute, and may not name a protected base.
- Each region `match` must resolve to exactly one descendant.
- The destination path must be free unless the call opts into overwriting.
- Each `fields` value must land on an int, bool, float, string, or enum property.

**`fields` overrides are region-only.** The root swap is invoked with no field map, and a `regions[].match` cannot
select the prefab root, because the descendant matcher excludes the root Transform. A root swap instead carries the
shared serialized values over from the script it replaces. The root's runtime-relevant values are written elsewhere
anyway: `CreateTask` sets `triggerMode`, `trialName`, and boundary visibility on the root `StimulusTriggerZone` at task
generation time, and `isActive` is established at runtime by `StimulusTriggerZone.ResetState` (see
[Anti-patterns](#anti-patterns)).

The keyword-only `overwrite` argument defaults to `false`, so an existing destination fails with `A prefab already
exists at '<path>'. Pass overwrite=true to replace it`. Pass `overwrite=True` to re-author a prefab in place, which
deletes the existing asset before the clone runs. Every requested script name is resolved *before* that delete, so a
missing, ambiguous, or abstract script name fails with the existing prefab intact. Only a failure during the edit phase
rolls back by deleting the destination asset, which leaves the path empty, so recover the original prefab from git
before retrying.

**Prerequisites:** author and compile the new `MonoBehaviour` script(s) first (see the [preflight
checklist](#preflight-checklist)), then run the tool. Hand off the downstream wiring afterward (see [Step
7](#step-7-hand-off-the-remaining-wiring)).

### Compose behavior with trials, not regions

`clone_zone_prefab_tool` edits a base prefab's existing region slots and stops there by design. A task is built from
one-zone-per-trial segments strung into a corridor, so extra behavior comes from adding a trial to the template, and
cues can reuse a texture so trials that look identical still differ in code. Reach for a richer single zone only when
one trial genuinely needs two coupled sensors, and otherwise express the new behavior as another trial. This is why the
tool targets a single zone's slots and leaves multi-region composition to the task template.

### When to drop to the manual workflow

The tool covers rename, root and region script swaps, and region field overrides, which are the operations every shipped
variant and both [worked examples](#worked-examples) need. Use the [manual fallback workflow](#manual-fallback-workflow)
to add or remove a region (the one operation the tool leaves to a future version) or to inspect the raw YAML when a
clone result is surprising.

---

## Why clone a base prefab

Both zone prefabs share a fixed structural skeleton:

- A root GameObject carrying a `Transform`, `MeshFilter` (built-in Quad), `MeshRenderer` (`TargetMat.mat`),
  `MeshCollider` (non-trigger, same Quad), `BoxCollider` (trigger), and a `StimulusTriggerZone` script.
- One or more modifier-zone children, each carrying a `Transform`, a `BoxCollider` (trigger), and exactly one
  `MonoBehaviour` from `SL.Tasks`.
- A `Transform.localPosition` of `(0, 0.505, 0)` on the root (the `ZoneVerticalOffset` defined in `CreateTask.cs`) and
  `(0, 0, 0)` on every modifier.
- Placeholder `BoxCollider` sizes, because `CreateTask.ConfigureRootZoneCollider`, called from `PlaceInteractionZone`,
  `PlaceCollisionZone`, and `PlaceOccupancyZone`, overwrites the root collider's size and center at task generation
  time. The child colliders are rewritten too: `PlaceInteractionZone` resizes and recenters the `GuidanceRegion`
  collider, `PlaceOccupancyZone` resizes the `OccupancyRegion` and `OccupancyGuidanceRegion` colliders, and
  `PlaceCollisionZone` deletes the `GuidanceRegion` child outright. The prefab's stored collider values are therefore
  not authoritative.

The only fields that vary between trigger zone variants are the modifier scripts on each `MonoBehaviour`, the GameObject
`m_Name` values, and the serialized field defaults on each modifier script. Everything else is identical.
`clone_zone_prefab_tool` copies the closest base prefab and patches exactly those three fields through Unity's
serialization layer, so the new variant inherits the skeleton and every invariant for free.

---

## Preflight checklist

You MUST verify all the following before manufacturing a new zone prefab:

1. **The new modifier script exists.** Author the new `MonoBehaviour` under `Assets/InfiniteCorridorTask/Scripts/`
   before producing the prefab, because Unity creates the `.cs.meta` file (and the stable GUID inside it) on the next
   Editor refresh, and the prefab YAML needs that GUID to wire the script in.
2. **The new script lives in the `SL.Tasks` namespace** (or a subnamespace), inherits from `MonoBehaviour`, and
   implements `IResettable` if it carries per-lap state. Follow the existing `OccupancyZone` / `GuidanceZone` pattern in
   `Assets/InfiniteCorridorTask/Scripts/`.
3. **The new script compiles into a named assembly.** A script placed under `Assets/InfiniteCorridorTask/Scripts/` joins
   `Sollertia.InfiniteCorridorTask` by sitting inside that subtree. A brand-new folder needs its own `.asmdef` plus a
   `references` entry from every consuming assembly, including the test assemblies, or the tests cannot see the type.
   See `/unity-tests`.
4. **The new script has test coverage.** Author an EditMode fixture at `Assets/Tests/EditMode/<NewZone>Tests.cs` on the
   shared `SL.Tests.ZoneRig` / `ZoneRigOptions` harness (extend `ZoneRig` when the new zone adds a component slot), and
   add a case to `Assets/Tests/PlayMode/ZonePlayModeTests.cs` when the behavior depends on real physics callbacks or
   wall-clock timing. See `/unity-tests`.
5. **Style compliance.** Invoke `automation:csharp-style` before writing the new script, and run CSharpier on the
   modified C# files before committing.
6. **Unity Editor running** with `McpBridge` reachable. The validation step relies on `inspect_prefab_tool`, which fails
   without the bridge. Invoke `/unity-mcp-environment-setup` first if the bridge is offline.

If any of the above is unmet, stop and resolve it before continuing.

---

## Canonical templates

Both templates live under `Assets/InfiniteCorridorTask/Prefabs/` and are committed to source control. The tool copies
whichever one matches the target zone shape, so pick the source by the table below.

The five `trigger_type` modes are backed by these two prefabs, and no mode has its own prefab file. `CreateTask` reuses
one template for each and selects the behavior at generation time:

| `trigger_type`      | Backing prefab                | Fires when…                                                                    |
|---------------------|-------------------------------|--------------------------------------------------------------------------------|
| `interaction`       | `StimulusTriggerZone.prefab`  | the interaction sensor reports the animal inside the zone                      |
| `collision`         | `StimulusTriggerZone.prefab`  | the animal crosses an invisible boundary wall, with no sensor and no occupancy |
| `occupancy_disarm`  | `OccupancyTriggerZone.prefab` | the animal collides with the boundary while occupancy is NOT met               |
| `occupancy_arm`     | `OccupancyTriggerZone.prefab` | the animal collides with the now-armed boundary after occupancy IS met         |
| `occupancy_trigger` | `OccupancyTriggerZone.prefab` | the required occupancy duration elapses, firing immediately with no boundary   |

`StimulusTriggerZone` dispatches on a `TriggerMode` enum field (`Interaction`, `Collision`, `OccupancyDisarm`,
`OccupancyArm`, `OccupancyTrigger`) that `CreateTask` sets from `trigger_type`. For the `collision` mode,
`CreateTask.PlaceCollisionZone` reuses `StimulusTriggerZone.prefab` with its `GuidanceRegion` child stripped and the
root collider set as a thin boundary wall at `stimulus_location`. The `occupancy_arm` and `occupancy_trigger` modes
reuse `OccupancyTriggerZone.prefab` unchanged, because `CreateTask` only selects the occupancy sub-mode. All three
occupancy modes keep the occupancy-guidance brake (`OccupancyGuidanceZone` publishing `Delay`).

### Interaction template (`StimulusTriggerZone.prefab`)

Hierarchy:

```text
StimulusTriggerZone               ← root, 6 components
└── GuidanceRegion                ← single child, 3 components
```

Use this template when the new zone needs exactly one modifier region that reports the animal's arrival to
`StimulusTriggerZone`. Examples: a new "approach detector" that triggers stimulus on zone entry without occupancy
timing. The shipped `interaction` and `collision` modes both back onto this prefab, and `collision` reuses it with the
`GuidanceRegion` child stripped and the root collider sized as a thin boundary wall.

### Occupancy template (`OccupancyTriggerZone.prefab`)

Hierarchy:

```text
OccupancyTriggerZone              ← root, 6 components
└── OccupancyRegion               ← child, 3 components
    └── OccupancyGuidanceRegion   ← grandchild, 3 components
```

Use this template when the new zone needs a nested modifier (a child of a modifier). Examples: a new
"must-wait-then-acknowledge" pattern where the inner zone reads state off the outer zone via `GetComponentInParent`. The
shipped `occupancy_disarm`, `occupancy_arm`, and `occupancy_trigger` modes all back onto this prefab, and `CreateTask`
reuses it unchanged and only selects the occupancy sub-mode on the parent `StimulusTriggerZone`.

---

## Manual fallback workflow

Use this workflow to add or remove a region (the one operation `clone_zone_prefab_tool` leaves to a future version), or
to inspect raw YAML when a clone result is surprising. For rename, script swaps, and region field overrides, the tool in
"Manufacturing a zone prefab" is the primary path.

The template invariants, the script-GUID lookup, and the step-by-step YAML edits (Steps 1 through 6: pick, read, write,
rename, swap script GUIDs, add/remove regions, validate) live in
[references/manual-yaml-editing.md](references/manual-yaml-editing.md). Author and compile the new script first (see the
[preflight checklist](#preflight-checklist)), apply those edits, then hand off the wiring below.

### Step 7: Hand off the remaining wiring

The new prefab is unreferenced once it is validated. The full cross-cutting recipe is split four ways:

| Slice                                                       | Owning skill                                            |
|-------------------------------------------------------------|---------------------------------------------------------|
| Python `TriggerType` enum + per-system `from_task_template` | `assets:library-extension` (Adding a new `TriggerType`) |
| `CreateTask` pipeline edits + `DeleteProtectedPaths`        | `/task-generator` (Adding a new zone trigger type)      |
| EditMode / PlayMode test updates                            | `/unity-tests` (test-suite layout and Test Runner)      |
| Prefab authoring (this skill)                               | `clone_zone_prefab_tool`, or the manual reference above |

The only per-lap-reset consideration that lives in this skill (because it depends on the new modifier script's class
identity) is:

- **Register new `IResettable` classes in `Task.FindResettableZones`.** Per-lap reset is driven by the corridor advance.
  `Task.ResetZoneStates` calls `ResetState()` on every cached resettable each time the actor teleports into a new
  corridor, and the cache is built once at `Start` by `Task.FindResettableZones`
  (`Assets/InfiniteCorridorTask/Scripts/Task.cs`). That method enumerates the **four** concrete `IResettable`
  implementers by type, calling `FindObjectsByType<T>` on `StimulusTriggerZone`, `GuidanceZone`, `OccupancyZone`, and
  `OccupancyGuidanceZone`. A **subclass** of any of those four is discovered polymorphically and needs no edit. A
  **standalone** `IResettable` implementer is invisible to all four lines and needs its own
  `resettables.AddRange(FindObjectsByType<NewZone>(FindObjectsSortMode.None));` added to `Task.FindResettableZones`, or
  per-lap state for the new zone is never reset.

Adding a `TriggerMode` member or a new protected prefab also breaks tests that pin the enum and the protected-asset
inventory, and `/unity-tests` owns which fixtures to update.

Once `assets:library-extension`, `/task-generator`, `/unity-tests`, and this skill's bullets have all landed, the new
prefab is usable by any YAML template that declares the new `trigger_type`.

---

## Worked examples

End-to-end walkthroughs of the two non-trivial zone-prefab authoring patterns live in
[references/worked-examples.md](references/worked-examples.md):

- **Example A:** Building a `CumulativeOccupancyTriggerZone` whose occupancy timer accumulates across multiple zone
  entries within a lap (instead of restarting each entry) by adding one `protected virtual` hook to `OccupancyZone`
  itself and overriding it in a subclass. `StimulusTriggerZone`, `OccupancyGuidanceZone`, and `Task.FindResettableZones`
  then pick the subclass up polymorphically with no edit. `OccupancyZone` declares no `protected virtual` member today,
  so the hook is a new, behavior-preserving edit to that file. The "no edit" clause covers only the collaborator classes
  that resolve the zone polymorphically. For the disarm-vs-arm polarity, prefer the built-in `occupancy_disarm` /
  `occupancy_arm` modes over authoring a subclass, because `StimulusTriggerZone`'s `TriggerMode` enum already covers it.
- **Example B:** Building a brand-new compound `SpeedInteractionTriggerZone` that gates interaction-triggered stimulus
  on the animal's traversal speed through an upstream speed-test region. It covers a new parent script, a new
  sibling-region script, and a new `PlaceSpeedInteractionZone` placement helper.

Load that file when you actually need to extend the zone vocabulary, because the `clone_zone_prefab_tool` flow above
handles routine variants in one call.

---

## Anti-patterns

- **Editing the canonical templates in place.** `StimulusTriggerZone.prefab` and `OccupancyTriggerZone.prefab` are
  protected via `McpBridge.DeleteProtectedPaths` for a reason: every generated task prefab references them by exact
  filename. Always copy to a new path.
- **Constructing prefab YAML from scratch.** Unity prefabs have implicit serialization rules (component ordering, fileID
  scoping, default field handling) that are easy to violate when authoring from a blank file. The two committed
  templates already satisfy every rule, so copy them.
- **Reusing fileIDs across prefabs.** fileIDs are scoped per-asset, so reusing the source template's IDs in the copy is
  fine. Never reuse an ID that already appears in the same prefab, because Unity will silently merge the components.
- **Modifying root invariants.** Changing the `MeshFilter` mesh, the `MeshRenderer` material, the `MeshCollider` setup,
  or `Transform.localPosition.y` on the root breaks the visual boundary or the trigger detection geometry. Override
  fields on the modifier scripts instead.
- **Hand-sizing the root BoxCollider.** `ConfigureRootZoneCollider` in `CreateTask.cs` overwrites the root collider's
  size and center at task generation time. Spending edit cycles tuning these values is wasted effort.
- **Setting `isActive: true` on the root.** Both base prefabs serialize `isActive: 0`, and a clone MUST preserve that
  structural convention. It is a convention rather than a gate. `StimulusTriggerZone.ResetState` sets `isActive = true`
  unconditionally, and `Start` calls it before the first `Update`, so the first lap runs through the same path every
  later lap uses. `Task.ResetZoneStates` then re-invokes it at every corridor advance. The serialized value is therefore
  not what gates first-frame firing, so do not "fix" behavior by flipping it.
- **Skipping `inspect_prefab_tool`.** Unity will silently load broken prefabs into the Editor but produce import errors
  that are easy to miss. `inspect_prefab_tool` returns a structured failure on which the agent can act.
- **Forgetting `DeleteProtectedPaths`.** A new hand-authored prefab without protection will be removed by a future
  `delete_asset_tool` cleanup pass. Always update `McpBridge.cs` in the same PR that introduces the prefab.

---

## Failure modes

| Symptom                                                                                         | Cause                                                                                                                                                                | Resolution                                                                                                                                                                            |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `clone_zone_prefab_tool` returns "Script type '…' not found"                                    | The named root or region script is not authored or the project is not compiled                                                                                       | Author the script, let Unity compile, then re-run the tool                                                                                                                            |
| `clone_zone_prefab_tool` returns "Script type '…' is ambiguous (N matches)"                     | Two or more compiled `MonoBehaviour` types share that simple class name, and resolution matches on `type.Name` only                                                  | Rename one of them so the class name is unique project-wide, then re-run, because the call fails before any asset is written                                                          |
| `clone_zone_prefab_tool` returns "Script type '…' is abstract"                                  | A `root_script` or `regions[].script` names an abstract `MonoBehaviour`                                                                                              | Name a concrete subclass, since the tool rejects abstract types up front because `AddComponent` would return null and strand a half-authored prefab                                   |
| `clone_zone_prefab_tool` returns "Root script '…' must derive from StimulusTriggerZone"         | `root_script` names a `MonoBehaviour` outside the `StimulusTriggerZone` hierarchy, and region scripts carry no such constraint                                       | Subclass `StimulusTriggerZone` for the root, or move the behavior into a region script                                                                                                |
| `clone_zone_prefab_tool` returns "Expected exactly one modifier script on '…', but found N"     | The targeted root or region GameObject does not carry exactly one `MonoBehaviour`                                                                                    | Target a GameObject with a single modifier script, or use the manual workflow to reshape the component set first                                                                      |
| `clone_zone_prefab_tool` returns "Field '…' does not exist on …"                                | A `fields` override names a member the modifier script does not declare                                                                                              | Correct the field name to one the script declares, and the tool rolled the asset back, so no partial prefab remains                                                                   |
| `clone_zone_prefab_tool` returns "Field '…' has unsupported type …"                             | A `fields` override targets a serialized property outside the supported set of int, bool, float, string, and enum, such as a `Vector3` or an object reference        | Set the value in the Editor or via the manual YAML workflow, and the tool rolled the asset back, so no partial prefab remains                                                         |
| `clone_zone_prefab_tool` returns "A prefab already exists at '…'"                               | The destination prefab already exists and `overwrite` defaults to false                                                                                              | Re-run with `overwrite=True` to replace the existing prefab, or pick a destination path that no prefab occupies                                                                       |
| `inspect_prefab_tool` returns success but the root `components` list omits the root zone script | Root MonoBehaviour was accidentally removed or its script GUID is invalid                                                                                            | Re-read the source template, then restore the root MonoBehaviour block verbatim (manual workflow only)                                                                                |
| Hierarchy returned by `inspect_prefab_tool` is flat (no children)                               | `m_Father` ↔ `m_Children` symmetry was broken                                                                                                                        | Verify every child's Transform `m_Father` matches the parent Transform's fileID, and that the parent's `m_Children` list contains the child's Transform fileID                        |
| Modifier script defaults look correct but the runtime behavior is wrong                         | `m_Script.guid` references the wrong script                                                                                                                          | Re-read the target script's `.cs.meta` and confirm the GUID, because the `m_EditorClassIdentifier` line is informational and may lag the real script class until Unity reimports      |
| Unity Editor reports "missing script" when opening the new prefab                               | Either the script does not exist yet, or the GUID is malformed                                                                                                       | Confirm the `.cs` and `.cs.meta` files exist under `Scripts/`, and ensure the GUID is 32 hex characters with no whitespace                                                            |
| New prefab disappears after a cleanup pass                                                      | Prefab not added to `McpBridge.DeleteProtectedPaths`                                                                                                                 | Add the path to the protected set and recover the prefab from git                                                                                                                     |
| `create_task_tool` fails with "Unable to place a trigger zone for trial '…'"                    | `ConfigLoader.ValidateTemplate` accepts the new `trigger_type` literal but `BuildSegmentPrefabs` has no matching branch, so the dispatch chain hits its fatal `else` | Hand off to `/task-generator` and add the `trigger_type` branch plus the `Place...Zone` helper, since the run aborts and writes no segment prefab, so no zoneless task is left behind |

---

## Related skills

| Skill                                        | Relationship                                                                                          |
|----------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `/task-prefabs` (this plugin)                | Regenerates task prefabs (`delete_task_tool` + `create_task_tool`) and provides `inspect_prefab_tool` |
| `/task-generator` (this plugin)              | Reference for `BuildSegmentPrefabs`, `Place...Zone`, and validator updates                            |
| `/task-parameters` (this plugin)             | Reference if the new zone exposes Inspector-driven fields                                             |
| `/task-scenes` (this plugin)                 | Consumer, opens and inspects the scene holding the regenerated task                                   |
| `/play-mode` (this plugin)                   | Consumer, exercises the new zone at runtime                                                           |
| `/mqtt-contract` (this plugin)               | Reference if the new modifier publishes or subscribes to MQTT topics                                  |
| `/gimbl-framework` (this plugin)             | Reference for the GIMBL vs `SL.*` split a new modifier script must respect                            |
| `/unity-mcp-environment-setup` (this plugin) | Run first if the Unity Editor bridge is unreachable                                                   |
| `/unity-tests` (this plugin)                 | Owns the EditMode / PlayMode suites and the assembly layout a new zone script joins                   |
| `assets:library-extension`                   | Required for the Python `TriggerType` enum member and the per-system `from_task_template` decision    |
| `automation:csharp-style`                    | Required when authoring the new modifier script and editing C# wiring                                 |
| `automation:commit`                          | Run after the prefab, script, and wiring changes are ready to commit                                  |
| `experiment:vr-driver-interface`             | Host consumes the `Stimulus` events these zones emit, joined via `DecomposedTrials.trial_names`       |

---

## Verification checklist

You MUST verify this checklist before submitting any new or modified zone prefab.

```text
Zone Prefabs Compliance:
- [ ] clone_zone_prefab_tool was used for rename, script-swap, and region field-override authoring, and the
      manual YAML workflow was used only to add or remove a region
- [ ] The new modifier script exists under Assets/InfiniteCorridorTask/Scripts/ with a valid
      .cs.meta and a stable GUID, and compiles into a named assembly (Sollertia.InfiniteCorridorTask
      for that folder, and a new folder needs its own .asmdef plus references from every consumer)
- [ ] The new prefab path is under Assets/InfiniteCorridorTask/Prefabs/ and the filename matches
      the intended TriggerType naming
- [ ] The new prefab's root m_Name matches the prefab filename basename
- [ ] The new prefab's root Transform.localPosition is exactly (0, 0.505, 0)
- [ ] The new prefab's root retains MeshFilter (built-in Quad), MeshRenderer (TargetMat.mat),
      MeshCollider (non-trigger), BoxCollider (trigger), and the root zone script (StimulusTriggerZone,
      or the subclass name when the root script was swapped)
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
- [ ] Task.FindResettableZones finds the new IResettable (subclasses of the four known types, namely
      StimulusTriggerZone, GuidanceZone, OccupancyZone, and OccupancyGuidanceZone, are covered
      polymorphically, and a standalone class needs an explicit FindObjectsByType registration in Task.cs)
- [ ] assets:library-extension was invoked on the assets-plugin side to register the new TriggerType
      member, and every acquisition system's from_task_template was explicitly confirmed to either map
      the member or deliberately leave it unmapped (no import-time parity check covers TriggerType)
- [ ] An EditMode fixture for the new zone script exists under Assets/Tests/EditMode/ on the shared
      ZoneRig harness, plus a PlayMode case in Assets/Tests/PlayMode/ZonePlayModeTests.cs when the
      behavior depends on real physics callbacks or wall-clock timing (see /unity-tests)
- [ ] The fixtures pinning the TriggerMode enum and the protected-asset inventory were updated for the
      new member and the new prefab path (see /unity-tests)
- [ ] Both test platforms pass in the Unity Test Runner (see /unity-tests)
- [ ] CSharpier ran cleanly on the modified C# files (the new script, McpBridge.cs, ConfigLoader.cs,
      CreateTask.cs, Task.cs)
```
