# Worked examples: zone-prefab authoring

Concrete end-to-end walkthroughs for the two non-trivial zone-prefab authoring patterns. Each example assumes the reader
has already authored and compiled the new MonoBehaviour script(s) and just needs a worked illustration of the
script-side and prefab-side choices.

**Tool shortcut:** the prefab-side steps each example spells out by hand (copy the base prefab, rename regions, swap the
modifier scripts, set field defaults, and validate) are a single `clone_zone_prefab_tool` call (see `SKILL.md`
"Manufacturing a zone prefab"). The manual steps remain here as a reference for what the tool does and for the
add/remove-region case the tool leaves to a future version.

---

## Example A: A cumulative-occupancy variant by subclassing `OccupancyZone`

Goal: ship a `CumulativeOccupancyTriggerZone` whose occupancy timer **accumulates across multiple entries within a lap**
instead of restarting on every entry. The canonical `OccupancyZone` calls `_occupancyTimer.Restart()` on each
`OnTriggerEnter`, so the animal must occupy the zone for one **continuous** `occupancyDurationMs` window. The variant
lets the animal dip in and out and counts the **total** dwell time, which suits sampling and foraging paradigms. It
reuses the `occupancy_trigger` firing rule (fire immediately once the requirement is met) and the existing
`OccupancyGuidanceRegion` brake unchanged.

This is a genuinely new behavior, not one of the five built-in `trigger_type` modes, so it is a good fit for the
subclass-a-modifier-zone technique.

Three runtime systems read `OccupancyZone` by **typed** lookup, and every one of them picks up a
`CumulativeOccupancyZone` subclass for free:

- `StimulusTriggerZone.cs` calls `GetComponentInChildren<OccupancyZone>()` and reads that child zone's `occupancyMet`
  flag to decide when to fire. The subclass leaves that path unchanged.
- `OccupancyGuidanceZone.cs` calls `GetComponentInParent<OccupancyZone>()` to read `occupancyDurationMs`,
  `GetElapsedMilliseconds()`, and `occupancyMet` for the brake. Its remaining-duration math works against the
  accumulated elapsed time with no edit.
- `Task.FindResettableZones` (`Assets/InfiniteCorridorTask/Scripts/Task.cs`) discovers resettables via
  `FindObjectsByType<OccupancyZone>` (plus the other three known implementers), so the corridor advance resets the
  subclass each lap with no edit.

A fully standalone class would be invisible to all three and would force edits to each. Subclassing is the cheaper path.
Because only the timer-accrual policy changes, the variant needs **one** subclass and **one** small hook in
`OccupancyZone`, and the guidance zone is reused unchanged.

### Step 1: Expose a virtual hook in `OccupancyZone` (one-time edit)

`OccupancyZone.OnTriggerEnter` hard-codes `_occupancyTimer.Restart()`. Add a `protected virtual` policy hook so a
subclass can resume instead of restart, without touching the private timer field:

- Add `protected virtual bool RestartTimerOnEntry => true;`.
- In `OnTriggerEnter`, replace `_occupancyTimer.Restart();` with `if (RestartTimerOnEntry) { _occupancyTimer.Restart();
  } else { _occupancyTimer.Start(); }`. `Stopwatch.Start` resumes from the accumulated elapsed time, while `Restart`
  zeroes it first.

This is a behavior-preserving edit, because every existing occupancy mode keeps the default `RestartTimerOnEntry =>
true`. Run `automation:csharp-style` (ataraxis marketplace) and CSharpier before committing.

### Step 2: Write `CumulativeOccupancyZone : OccupancyZone`

Under `Assets/InfiniteCorridorTask/Scripts/CumulativeOccupancyZone.cs` (invoke `automation:csharp-style`):

- Declare the class in `namespace SL.Tasks`, the namespace every script in that folder uses. The folder's
  `Sollertia.InfiniteCorridorTask.asmdef` (`"name": "Sollertia.InfiniteCorridorTask"`, `"rootNamespace": "SL"`) compiles
  the new file into the `Sollertia.InfiniteCorridorTask` assembly, and no script in the project compiles into Unity's
  predefined `Assembly-CSharp`.
- Override `RestartTimerOnEntry => false` so re-entries resume the stopwatch and the dwell time accumulates across the
  lap.

Everything else is unchanged, which covers the `occupancyMet` signal, the `Stopwatch`, the per-lap `ResetState` reset,
and every inherited Unity callback. Save the file, call `refresh_assets_tool` so the Editor imports it, then confirm
`CumulativeOccupancyZone.cs.meta` exists with a fresh `guid:`.

### Step 3: Copy and rename the prefab

1. Copy `OccupancyTriggerZone.prefab` → `Prefabs/CumulativeOccupancyTriggerZone.prefab`.
2. Rename the root: `m_Name: OccupancyTriggerZone` → `m_Name: CumulativeOccupancyTriggerZone`.
3. Rename the occupancy region: `m_Name: OccupancyRegion` → `m_Name: CumulativeOccupancyRegion`.

### Step 4: Swap the modifier script GUID

For the `CumulativeOccupancyRegion` MonoBehaviour block: swap `OccupancyZone`'s GUID for `CumulativeOccupancyZone`'s
GUID (from its `.cs.meta`). `m_Script.guid` is the load-bearing field. `m_EditorClassIdentifier:` is informational and
Unity rewrites it on reimport, so either leave it empty or write
`Sollertia.InfiniteCorridorTask::SL.Tasks.CumulativeOccupancyZone`, because Unity's format is
`<assembly>::<fully-qualified type>`. The `Assembly-CSharp::OccupancyZone` string the committed
`OccupancyTriggerZone.prefab` still carries on its `OccupancyRegion` MonoBehaviour block is pre-asmdef residue and MUST
NOT be copied into a newly authored block.

The `OccupancyGuidanceRegion` grandchild and the root `StimulusTriggerZone` MonoBehaviour are **unchanged**, because
both reach the polymorphic `CumulativeOccupancyZone` through their inherited typed lookups.

### Step 5: Validate via `inspect_prefab_tool`

Expect a `CumulativeOccupancyTriggerZone` root with `StimulusTriggerZone` in its `components` list, a
`CumulativeOccupancyRegion` child with `CumulativeOccupancyZone` in `components`, and an `OccupancyGuidanceRegion`
grandchild with `OccupancyGuidanceZone` in `components`. The hierarchy depth and parent-child fileID pairings match the
canonical occupancy prefab exactly.

### Step 6: Wire downstream per `SKILL.md` Step 7

- New `TriggerType` member `OCCUPANCY_TRIGGER_CUMULATIVE` (`occupancy_trigger_cumulative`) via
  `assets:library-extension`. The matching literal also goes into **both** `ConfigLoader.ValidateTemplate` gates: the
  accepted `trigger_type` literal set, along with the `InvalidDataException` message beside it, and the separate
  `isOccupancy` predicate that makes `occupancy_duration_ms` mandatory. The `BuildSegmentPrefabs` dispatch branch
  passes `trial.occupancyDurationMs.Value` unguarded (`CreateTask.cs`), so a literal missing from the second gate lets
  a template omit the duration and throws at generation time.
- A `PlaceCumulativeOccupancyZone` helper in `CreateTask.cs` (clone `PlaceOccupancyZone`) that instantiates
  `CumulativeOccupancyTriggerZone.prefab` and sets the root `StimulusTriggerZone.triggerMode =
  TriggerMode.OccupancyTrigger`. The variant reuses the occupancy-trigger **firing** rule and only customizes the timer
  accrual on the zone. Dispatch to it from `BuildSegmentPrefabs` on the new literal (via `/task-generator`).
- `DeleteProtectedPaths` entry for `CumulativeOccupancyTriggerZone.prefab`.
- **Test suite.** Add `CumulativeOccupancyZone` to the `expected` array in
  `IResettable_RuntimeAssembly_DeclaresExactlyTheRegisteredImplementers`
  (`Assets/Tests/EditMode/StimulusTriggerZoneTests.cs`), which asserts the runtime assembly declares exactly the
  registered `IResettable` implementers, and extend the occupancy-zone tests to cover the accumulating timer (see
  `/unity-tests`).
- **No** `Task.FindResettableZones` edit is needed, because `CumulativeOccupancyZone` inherits from `OccupancyZone`,
  which `Task.FindResettableZones` (called from `Task.Start`) already discovers polymorphically.

---

## Example B: A speed-gated interaction reward compound trigger

Goal: ship a `SpeedInteractionTriggerZone` where the animal has to traverse an upstream **speed-test region** within a
target running-speed range AND then engage the interaction sensor inside a downstream **reward-interaction region** to
receive the stimulus. The two criteria are independent geometrically (separate colliders at separate cm offsets) and
conjunctive at fire time (both must succeed in the same lap).

`StimulusTriggerZone`'s built-in interaction gate covers the interaction half but has no notion of speed. To keep
`Task.FindResettableZones`'s typed `FindObjectsByType<StimulusTriggerZone>` working without an explicit edit, the new
root parent script **subclasses** `StimulusTriggerZone` rather than replacing it. The new sibling-region script
(`SpeedZone`) is a standalone `IResettable`. It does not subclass any of the four known implementers, so it requires an
explicit registration in `Task.FindResettableZones`.

### Step 1: Author `SpeedZone.cs` (standalone `IResettable`)

Under `Assets/InfiniteCorridorTask/Scripts/SpeedZone.cs` (invoke `automation:csharp-style`):

- Declare `SpeedZone` in `namespace SL.Tasks` so the folder's `Sollertia.InfiniteCorridorTask.asmdef` compiles it into
  the `Sollertia.InfiniteCorridorTask` assembly alongside the zones that share that folder.
- `[Serializable]` fields: `targetSpeedCmPerSec`, `toleranceCmPerSec`, plus an `[HideInInspector]` `cmPerUnit` populated
  by `PlaceSpeedInteractionZone` at task generation time so the script does not need to discover the conversion at
  runtime.
- `IResettable` state: `inZone`, `speedMet`, `_entryPosZ`, `_entryTime`.
- `OnTriggerEnter(Collider other)`: stamp `_entryPosZ = other.transform.position.z` and `_entryTime = Time.time`.
- `OnTriggerExit(Collider other)`: compute `avgSpeedCmPerSec = (other.transform.position.z - _entryPosZ) * cmPerUnit /
  (Time.time - _entryTime)` and set `speedMet = Mathf.Abs(avg - targetSpeedCmPerSec) <= toleranceCmPerSec`.
- `ResetState()`: clear `speedMet` and `inZone` so each lap is judged independently.

`cmPerUnit` is the only piece of cross-script state. Injecting it from `PlaceSpeedInteractionZone` (which already reads
`template.vrEnvironment.cmPerUnityUnit`) keeps `SpeedZone` self-contained and avoids a `FindAnyObjectByType<Task>()`
round-trip every frame.

### Step 2: Author `SpeedInteractionTriggerZone : StimulusTriggerZone`

Under `Assets/InfiniteCorridorTask/Scripts/SpeedInteractionTriggerZone.cs` (invoke `automation:csharp-style`):

- Promote `StimulusTriggerZone.Start`, `UpdateInteractionMode`, `OnTriggerExit`, and `TriggerStimulus` to `protected
  virtual` first, parallel to Example A's `OccupancyZone` edit. Promote the `BehaviorCause` and `GuidanceCause`
  constants alongside them (both `private const` in `StimulusTriggerZone.cs`), because a subclass has to name them
  when it calls `TriggerStimulus`. Promote the `_inZone`, `_interactionDetectedInZone`, `_guidanceZone`, and `_task`
  fields too (all `private` in `StimulusTriggerZone.cs`), because an outright `UpdateInteractionMode` or
  `OnTriggerExit` override has to read them. The base declares all four methods `private`, and Unity dispatches
  `Start` and `OnTriggerExit` as messages rather than through virtual dispatch, so a subclass reaches the base
  implementations only after this promotion.
- Cache the sibling `SpeedZone` via `GetComponentInChildren<SpeedZone>()` in an overridden `Start`, calling
  `base.Start()` first so the base MQTT and `Task` wiring runs.
- Override `UpdateInteractionMode` outright rather than layering a `_speedZone.speedMet` check onto the base. The base
  `UpdateInteractionMode` (`StimulusTriggerZone.cs`) branches three ways on `_task.requireInteraction` and
  `_guidanceZone != null`. Because Step 4 replaces this prefab's `GuidanceZone` with `SpeedZone`, `_guidanceZone` is
  null, so the base's final `else` branch is the one that would run whenever `requireInteraction` is false, firing on
  bare `_inZone` with no sensor involvement at all.
- Override `OnTriggerExit` as well. `StimulusTriggerZone.OnTriggerExit` (`StimulusTriggerZone.cs`) resolves an
  interaction trial on the boundary crossing via
  `TriggerStimulus(delivered: _interactionDetectedInZone, cause: BehaviorCause)`. A subclass that gates only
  `UpdateInteractionMode` therefore still delivers on exit whenever the animal engaged the sensor inside the zone,
  however slowly it traversed the speed-test region. The speed gate has to apply on both paths or the conjunction the
  example promises is not real.
- Inherit `ResetState` from `StimulusTriggerZone`, because `SpeedZone.ResetState` handles its own state.

Subclassing means:

- `Task.FindResettableZones`'s `FindObjectsByType<StimulusTriggerZone>` finds the new parent polymorphically, so no edit
  is needed there for the parent.
- `SpeedZone` is **not** a subclass of the four known implementers, so `Task.FindResettableZones` does need a
  `FindObjectsByType<SpeedZone>` registration. This is the one mandatory `Task.cs` edit per `SKILL.md` Step 7.

### Step 3: Copy and rename the prefab

1. Copy `StimulusTriggerZone.prefab` → `Prefabs/SpeedInteractionTriggerZone.prefab`.
2. Rename the root: `m_Name: StimulusTriggerZone` → `m_Name: SpeedInteractionTriggerZone`.
3. Rename the existing `GuidanceRegion` child: `m_Name: GuidanceRegion` → `m_Name: SpeedTestRegion`.

### Step 4: Swap script GUIDs

- Root MonoBehaviour: swap `StimulusTriggerZone`'s GUID for `SpeedInteractionTriggerZone`'s GUID. The root invariants
  (MeshFilter Quad, MeshRenderer TargetMat, MeshCollider Quad, BoxCollider trigger, `showBoundary: 0`, `isActive: 0`)
  all stay because the subclass inherits them.
- `SpeedTestRegion` MonoBehaviour: swap `GuidanceZone`'s GUID for `SpeedZone`'s GUID. Replace the `GuidanceZone` field
  block (`inZone: 0`) with `targetSpeedCmPerSec`, `toleranceCmPerSec`, and a placeholder `cmPerUnit: 10` (overwritten at
  task generation).
- Leave `m_EditorClassIdentifier:` empty on both the root and the `SpeedTestRegion` block. `StimulusTriggerZone.prefab`
  ships the field blank on its root `StimulusTriggerZone` block and on its `GuidanceZone` block, and Unity repopulates
  the field on reimport. If a value is written by hand, the format is `<assembly>::<fully-qualified type>`, giving
  `Sollertia.InfiniteCorridorTask::SL.Tasks.SpeedZone` rather than `Assembly-CSharp::`.

Replacing the region's `GuidanceZone` with `SpeedZone` removes the only `GuidanceZone` from every scene generated from
this prefab, and `GuidanceZone` is the marker on which the Task Parameters surface keys.
`McpBridge.AcquireSceneComponents` (`McpBridge.cs`) sets `HasInteractionZone = false`, so `read_task_parameters` reports
`visibility.task.require_interaction: false` and the Parameters-window control is hidden.
`state.task.require_interaction` keeps reporting the live `Task.requireInteraction` field, which `CreateTask` writes as
`true` on every generated task and which is stuck there, because `write_task_parameters` rejects the key in
`ValidateTaskSectionWrites`. The overridden `UpdateInteractionMode` still reads `_task.requireInteraction`
(`StimulusTriggerZone.cs`), so it sees `true` and takes the strict branch that demands the interaction sensor. Either
keep a `GuidanceZone` on a second region or extend `AcquireSceneComponents` to detect `SpeedZone` as well (see
`/task-parameters`).

### Step 5: Validate via `inspect_prefab_tool`

```text
SpeedInteractionTriggerZone (SpeedInteractionTriggerZone in components)
└── SpeedTestRegion         (SpeedZone in components, BoxCollider over speed-test range)
```

The reward-interaction collider lives on the root. It is not a separate child. `inspect_prefab_tool` reports each
attached component once under its concrete runtime type name, so the root's `components` list carries the script name
`SpeedInteractionTriggerZone` alongside `MeshFilter`, `MeshRenderer`, `MeshCollider`, and `BoxCollider`. Confirm the
script name and each of those four Unity components appear in the list.

### Step 6: Wire downstream per `SKILL.md` Step 7

- New `TriggerType` member (via `assets:library-extension`).
- New YAML fields on `TrialStructure` for speed-test cm bounds, target speed, and tolerance (mirror the existing
  `stimulus_trigger_zone_*_cm` pattern, and pass them through `PlaceSpeedInteractionZone` in `CreateTask.cs`).
- A `PlaceSpeedInteractionZone` helper that positions the root collider over the reward range and the `SpeedTestRegion`
  collider over the speed-test range, writes `targetSpeedCmPerSec`, `toleranceCmPerSec`, and `cmPerUnit` onto the
  `SpeedZone` instance, and sets `showBoundary` on the root.
- `BuildSegmentPrefabs` dispatch entry that selects the new prefab for the new `trigger_type`.
- `DeleteProtectedPaths` entry for `SpeedInteractionTriggerZone.prefab`.
- `ConfigLoader.ValidateTemplate` accepts the new `trigger_type` literal in the accepted-literal set
  and in its `InvalidDataException` message (`ConfigLoader.cs`). The new mode is not an occupancy mode, so it stays
  out of the `isOccupancy` predicate in the same method.
- **`Task.cs`.** Add `resettables.AddRange(FindObjectsByType<SpeedZone>(FindObjectsSortMode.None));` inside
  `Task.FindResettableZones`, next to the existing four `FindObjectsByType` calls. `SpeedInteractionTriggerZone` is
  covered by the existing `FindObjectsByType<StimulusTriggerZone>` line via polymorphism.
- **`McpBridge.AcquireSceneComponents`.** Settle the `HasInteractionZone` consequence raised in Step 4 before the prefab
  ships, either by keeping a `GuidanceZone` in the prefab or by teaching the scan about `SpeedZone`.
- **Test suite.** Add both `SpeedZone` and `SpeedInteractionTriggerZone` to the `expected` array in
  `IResettable_RuntimeAssembly_DeclaresExactlyTheRegisteredImplementers`
  (`Assets/Tests/EditMode/StimulusTriggerZoneTests.cs`), which asserts the runtime assembly declares exactly the
  registered `IResettable` implementers. Then add coverage for the conjunctive gate on both the `Update` and the
  `OnTriggerExit` delivery paths (see `/unity-tests`).
