# Worked examples: zone-prefab authoring

Concrete end-to-end walkthroughs for the two non-trivial zone-prefab authoring patterns. Each
example assumes the reader has already completed every step in `SKILL.md` ("Workflow", Steps 1–7)
and just needs a worked illustration of the script-side and YAML-side choices.

---

## Example A: A cumulative-occupancy variant by subclassing `OccupancyZone`

Goal: ship a `CumulativeOccupancyTriggerZone` whose occupancy timer **accumulates across multiple
entries within a lap** instead of restarting on every entry. The canonical `OccupancyZone` calls
`_occupancyTimer.Restart()` on each `OnTriggerEnter`, so the animal must occupy the zone for one
**continuous** `occupancyDurationMs` window. The variant lets the animal dip in and out and counts
the **total** dwell time — useful for sampling/foraging paradigms — while reusing the
`occupancy_trigger` firing rule (fire immediately once the requirement is met) and the existing
`OccupancyGuidanceRegion` brake unchanged.

This is a genuinely new behavior, not one of the five built-in `trigger_type` modes, so it is a
good fit for the subclass-a-modifier-zone technique.

Three runtime systems read `OccupancyZone` by **typed** lookup, and every one of them picks up a
`CumulativeOccupancyZone` subclass for free:

- `StimulusTriggerZone.cs` calls `GetComponentInChildren<OccupancyZone>()` and reads the parent's
  `occupancyMet` flag to decide when to fire — unchanged by the subclass.
- `OccupancyGuidanceZone.cs` calls `GetComponentInParent<OccupancyZone>()` to read
  `occupancyDurationMs`, `GetElapsedMilliseconds()`, and `occupancyMet` for the brake — its
  remaining-duration math works against the accumulated elapsed time with no edit.
- `ResetZone.cs` discovers resettables via `FindObjectsByType<OccupancyZone>` (plus the other two
  known types), so it resets the subclass each lap with no edit.

A fully standalone class would be invisible to all three and would force edits to each. Subclassing
is the cheaper path, and because only the timer-accrual policy changes, the variant needs **one**
subclass and **one** small hook in `OccupancyZone` — the guidance zone is reused unchanged.

### Step 1: Expose a virtual hook in `OccupancyZone` (one-time edit)

`OccupancyZone.OnTriggerEnter` hard-codes `_occupancyTimer.Restart()`. Add a `protected virtual`
policy hook so a subclass can resume instead of restart, without touching the private timer field:

- Add `protected virtual bool RestartTimerOnEntry => true;`.
- In `OnTriggerEnter`, replace `_occupancyTimer.Restart();` with
  `if (RestartTimerOnEntry) { _occupancyTimer.Restart(); } else { _occupancyTimer.Start(); }`
  (`Stopwatch.Start` resumes from the accumulated elapsed time; `Restart` zeroes it first).

This is a behavior-preserving edit — every existing occupancy mode keeps the default
`RestartTimerOnEntry => true`. Run `/csharp-style` and CSharpier before committing.

### Step 2: Write `CumulativeOccupancyZone : OccupancyZone`

Under `Assets/InfiniteCorridorTask/Scripts/CumulativeOccupancyZone.cs` (invoke `/csharp-style`):

- Override `RestartTimerOnEntry => false` so re-entries resume the stopwatch and the dwell time
  accumulates across the lap.

Everything else — the `occupancyMet` signal, the `Stopwatch`, the per-lap `ResetState` reset, and
every inherited Unity callback — is unchanged. Save and confirm `CumulativeOccupancyZone.cs.meta`
exists with a fresh `guid:`.

### Step 3: Copy and rename the prefab

1. Copy `OccupancyTriggerZone.prefab` → `Prefabs/CumulativeOccupancyTriggerZone.prefab`.
2. Rename the root: `m_Name: OccupancyTriggerZone` → `m_Name: CumulativeOccupancyTriggerZone`.
3. Rename the occupancy region: `m_Name: OccupancyRegion` → `m_Name: CumulativeOccupancyRegion`.

### Step 4: Swap the modifier script GUID

For the `CumulativeOccupancyRegion` MonoBehaviour block: swap `OccupancyZone`'s GUID for
`CumulativeOccupancyZone`'s GUID (from its `.cs.meta`), and update `m_EditorClassIdentifier:` from
`Assembly-CSharp::OccupancyZone` to `Assembly-CSharp::CumulativeOccupancyZone`.

The `OccupancyGuidanceRegion` grandchild and the root `StimulusTriggerZone` MonoBehaviour are
**unchanged** — both reach the polymorphic `CumulativeOccupancyZone` through their inherited typed
lookups.

### Step 5: Validate via `inspect_prefab_tool`

Expect a `CumulativeOccupancyTriggerZone` root with `StimulusTriggerZone` in its `components` list,
a `CumulativeOccupancyRegion` child with `CumulativeOccupancyZone` in `components`, and an
`OccupancyGuidanceRegion` grandchild with `OccupancyGuidanceZone` in `components`. The hierarchy
depth and parent-child fileID pairings match the canonical occupancy prefab exactly.

### Step 6: Wire downstream per `SKILL.md` Step 7

- New `TriggerType` member `OCCUPANCY_TRIGGER_CUMULATIVE` (`occupancy_trigger_cumulative`) via
  `/library-extension`, plus the matching literal in `ConfigLoader.ValidateTemplate`.
- A `PlaceCumulativeOccupancyZone` helper in `CreateTask.cs` (clone `PlaceOccupancyZone`) that
  instantiates `CumulativeOccupancyTriggerZone.prefab` and sets the root
  `StimulusTriggerZone.triggerMode = TriggerMode.OccupancyTrigger` — the variant reuses the
  occupancy-trigger **firing** rule and only customizes the timer accrual on the zone. Dispatch to
  it from `BuildSegmentPrefabs` on the new literal (via `/task-generator`).
- `DeleteProtectedPaths` entry for `CumulativeOccupancyTriggerZone.prefab`.
- **No** `ResetZone.cs` edit needed — `CumulativeOccupancyZone` inherits from `OccupancyZone`, which
  `ResetZone.Start` already discovers polymorphically.

---

## Example B: New compound trigger — speed-gated interaction reward

Goal: ship a `SpeedInteractionTriggerZone` where the animal has to traverse an upstream **speed-test
region** within a target running-speed range AND then engage the interaction sensor inside a
downstream **reward-interaction region** to receive the stimulus. The two criteria are independent
geometrically (separate colliders at separate cm offsets) and conjunctive at fire time (both must
succeed in the same lap).

`StimulusTriggerZone`'s built-in interaction gate covers the interaction half but has no notion of
speed. To keep `ResetZone.cs`'s typed `FindObjectsByType<StimulusTriggerZone>` working without an explicit
edit, the new root parent script **subclasses** `StimulusTriggerZone` rather than replacing it.
The new sibling-region script (`SpeedZone`) is a standalone `IResettable` — it does not subclass
any of the three known types, so it requires an explicit `ResetZone.cs` registration.

### Step 1: Author `SpeedZone.cs` (standalone `IResettable`)

Under `Assets/InfiniteCorridorTask/Scripts/SpeedZone.cs` (invoke `/csharp-style`):

- `[Serializable]` fields: `targetSpeedCmPerSec`, `toleranceCmPerSec`, plus an `[HideInInspector]`
  `cmPerUnit` populated by `PlaceSpeedInteractionZone` at task generation time so the script does not
  need to discover the conversion at runtime.
- `IResettable` state: `inZone`, `speedMet`, `_entryPosZ`, `_entryTime`.
- `OnTriggerEnter(Collider other)`: stamp `_entryPosZ = other.transform.position.z` and
  `_entryTime = Time.time`.
- `OnTriggerExit(Collider other)`: compute
  `avgSpeedCmPerSec = (other.transform.position.z - _entryPosZ) * cmPerUnit / (Time.time - _entryTime)`
  and set `speedMet = Mathf.Abs(avg - targetSpeedCmPerSec) <= toleranceCmPerSec`.
- `ResetState()`: clear `speedMet` and `inZone` so each lap is judged independently.

`cmPerUnit` is the only piece of cross-script state. Injecting it from `PlaceSpeedInteractionZone`
(which already reads `template.vrEnvironment.cmPerUnityUnit`) keeps `SpeedZone` self-contained
and avoids a `FindAnyObjectByType<Task>()` round-trip every frame.

### Step 2: Author `SpeedInteractionTriggerZone : StimulusTriggerZone`

Under `Assets/InfiniteCorridorTask/Scripts/SpeedInteractionTriggerZone.cs` (invoke `/csharp-style`):

- Cache the sibling `SpeedZone` via `GetComponentInChildren<SpeedZone>()` in `Start` (after
  `base.Start()`).
- Override the interaction-mode fire condition. The simplest path is to make
  `StimulusTriggerZone.UpdateInteractionMode` and `TriggerStimulus` `protected virtual` first (parallel
  to Example A's `OccupancyZone` edits), then override `UpdateInteractionMode` to gate on
  `_speedZone != null && _speedZone.speedMet` in addition to the existing interaction-detection check.
- Inherit `ResetState` from `StimulusTriggerZone`; `SpeedZone.ResetState` handles its own state.

Subclassing means:

- `ResetZone.Start`'s `FindObjectsByType<StimulusTriggerZone>` finds the new parent
  polymorphically — no edit needed there for the parent.
- `SpeedZone` is **not** a subclass of the three known types, so `ResetZone.cs` does need
  a `FindObjectsByType<SpeedZone>` registration. This is the one mandatory `ResetZone` edit per
  `SKILL.md` Step 7.

### Step 3: Copy and rename the prefab

1. Copy `StimulusTriggerZone.prefab` → `Prefabs/SpeedInteractionTriggerZone.prefab`.
2. Rename the root: `m_Name: StimulusTriggerZone` → `m_Name: SpeedInteractionTriggerZone`.
3. Rename the existing `GuidanceRegion` child: `m_Name: GuidanceRegion` → `m_Name: SpeedTestRegion`.

### Step 4: Swap script GUIDs

- Root MonoBehaviour: swap `StimulusTriggerZone`'s GUID for `SpeedInteractionTriggerZone`'s GUID. The
  root invariants (MeshFilter Quad, MeshRenderer TargetMat, MeshCollider Quad, BoxCollider
  trigger, `showBoundary: 0`, `isActive: 0`) all stay because the subclass inherits them.
- `SpeedTestRegion` MonoBehaviour: swap `GuidanceZone`'s GUID for `SpeedZone`'s GUID. Replace
  the `GuidanceZone` field block (`inZone: 0`) with `targetSpeedCmPerSec`,
  `toleranceCmPerSec`, and a placeholder `cmPerUnit: 10` (overwritten at task generation).
- Update `m_EditorClassIdentifier:` lines to match (root populated or empty per the source
  template's polarity; modifier set to `Assembly-CSharp::SpeedZone`).

### Step 5: Validate via `inspect_prefab_tool`

```text
SpeedInteractionTriggerZone (SpeedInteractionTriggerZone + StimulusTriggerZone in components)
└── SpeedTestRegion         (SpeedZone in components, BoxCollider over speed-test range)
```

The reward-interaction collider lives on the root; it is not a separate child. `inspect_prefab_tool`'s
`components` list for the root should include both the subclass name (`SpeedInteractionTriggerZone`)
and — through inheritance polling — should at minimum confirm `BoxCollider`, `MeshFilter`,
`MeshRenderer`, and `MeshCollider` are still present.

### Step 6: Wire downstream per `SKILL.md` Step 7

- New `TriggerType` member (via `/library-extension`).
- New YAML fields on `TrialStructure` for speed-test cm bounds, target speed, and tolerance
  (mirror the existing `stimulus_trigger_zone_*_cm` pattern; pass them through
  `PlaceSpeedInteractionZone` in `CreateTask.cs`).
- A `PlaceSpeedInteractionZone` helper that positions the root collider over the reward range and the
  `SpeedTestRegion` collider over the speed-test range, writes `targetSpeedCmPerSec`,
  `toleranceCmPerSec`, and `cmPerUnit` onto the `SpeedZone` instance, and sets `showBoundary`
  on the root.
- `BuildSegmentPrefabs` dispatch entry that selects the new prefab for the new `trigger_type`.
- `DeleteProtectedPaths` entry for `SpeedInteractionTriggerZone.prefab`.
- `ConfigLoader.ValidateTemplate` accepts the new `trigger_type` literal.
- **`ResetZone.cs`** — add `resettables.AddRange(FindObjectsByType<SpeedZone>(FindObjectsSortMode.None));`
  next to the existing three `FindObjectsByType` calls. `SpeedInteractionTriggerZone` is covered by the
  existing `FindObjectsByType<StimulusTriggerZone>` line via polymorphism.
