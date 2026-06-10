# Worked examples: zone-prefab authoring

Concrete end-to-end walkthroughs for the two non-trivial zone-prefab authoring patterns. Each
example assumes the reader has already completed every step in `SKILL.md` ("Workflow", Steps 1–7)
and just needs a worked illustration of the script-side and YAML-side choices.

---

## Example A: Inverting an occupancy zone from "disable trigger" to "enable trigger"

Goal: convert the canonical `OccupancyTriggerZone` (aversive — the stimulus boundary starts
**armed**, occupying for `occupancyDurationMs` **disarms** it, and a failure to occupy leaves the
boundary armed so the animal triggers the aversive stimulus on cross) into a rewarding variant
where the boundary starts **dormant** and meeting the occupancy requirement **arms** it, so
crossing the boundary delivers a reward — and where the existing `OccupancyGuidanceRegion`
brake mechanism still helps the animal complete the occupancy window (now to earn the reward
rather than avoid punishment).

Three runtime systems read `OccupancyZone` by **typed** lookup:

- `StimulusTriggerZone.cs` calls `GetComponentInChildren<OccupancyZone>()` to enter occupancy
  mode and gates its fire condition on `!_occupancyZone.boundaryDisarmed`.
- `OccupancyGuidanceZone.cs` calls `GetComponentInParent<OccupancyZone>()` to read the parent's
  `occupancyDurationMs` and `GetElapsedMilliseconds()` for the brake duration, and gates the
  brake on `!_parentOccupancyZone.boundaryDisarmed`.
- `ResetZone.cs:24-31` discovers resettables via `FindObjectsByType<OccupancyZone>` (plus the
  other two known types).

All three rely on polymorphism: a `RewardOccupancyZone` that **subclasses `OccupancyZone`** is
picked up by every one of them for free. A fully standalone class is invisible to all three and
would force edits to `StimulusTriggerZone`, `OccupancyGuidanceZone`, and `ResetZone`. Subclassing
is the cheaper path.

The `OccupancyGuidanceZone` brake-gate condition (`!boundaryDisarmed`) is also polarity-specific:
in the aversive case it fires while the boundary is still armed (occupancy not yet met). With the
inverted reward semantics, `boundaryDisarmed = true` while occupancy is in progress, so the gate
becomes `!true = false` and the brake never fires when it would help. The reward variant
therefore also needs a `RewardOccupancyGuidanceZone` subclass with the gate inverted, OR the
`OccupancyGuidanceZone` gate must be made overridable via a `protected virtual` hook.

### Step 1: Expose virtual hooks in the canonical scripts (one-time edits)

`OccupancyZone.cs` keeps every behavioral method `private`. Add two `protected virtual` hooks so
the subclass can flip polarity without re-implementing the timer logic:

- A `protected virtual bool InitialBoundaryDisarmed => false;` field-init equivalent. Use it in
  `Start` and `ResetState` instead of the current literal `boundaryDisarmed = false`.
- A `protected virtual void OnOccupancyMet()` (change the existing `private` declaration to
  `protected virtual`).

`OccupancyGuidanceZone.cs` likewise keeps the gate inline. Add:

- A `protected virtual bool ShouldFireBrake => !_parentOccupancyZone.boundaryDisarmed;` property.
- Use it in the `OnTriggerEnter` check (`if (!_task.requireWait && !_hasTriggered && ShouldFireBrake)`).

These are minimal, behavior-preserving edits — every existing call site keeps the default
polarity. Run `/csharp-style` before committing.

### Step 2: Write `RewardOccupancyZone : OccupancyZone`

Under `Assets/InfiniteCorridorTask/Scripts/RewardOccupancyZone.cs`:

- Override `InitialBoundaryDisarmed => true` (trigger dormant on lap start).
- Override `OnOccupancyMet` to set `boundaryDisarmed = false` instead of `true` (arm on success).
  Stop the timer the same way the base class did.

Because the subclass keeps the `boundaryDisarmed` field, the timer stopwatch, and every Unity
callback inherited from `OccupancyZone`, no other code changes. Save and confirm
`RewardOccupancyZone.cs.meta` exists with a fresh `guid:`.

### Step 3: Write `RewardOccupancyGuidanceZone : OccupancyGuidanceZone`

Override `ShouldFireBrake => _parentOccupancyZone.boundaryDisarmed` — the brake now fires while
occupancy is still being measured (in the reward case, that's when `boundaryDisarmed = true`).
Everything else (the elapsed-time math, the MQTT `Delay` send, the `_hasTriggered` latch) is
inherited unchanged. Save and confirm the new `.cs.meta`.

### Step 4: Copy and rename the prefab

1. Copy `OccupancyTriggerZone.prefab` → `Prefabs/RewardOccupancyTriggerZone.prefab`.
2. Rename the root: `m_Name: OccupancyTriggerZone` → `m_Name: RewardOccupancyTriggerZone`.
3. Rename the occupancy region: `m_Name: OccupancyRegion` → `m_Name: RewardOccupancyRegion`.

### Step 5: Swap modifier script GUIDs

For the `OccupancyRegion` MonoBehaviour block: swap `OccupancyZone`'s GUID for
`RewardOccupancyZone`'s GUID (from its `.cs.meta`). Update `m_EditorClassIdentifier:` from
`Assembly-CSharp::OccupancyZone` to `Assembly-CSharp::RewardOccupancyZone`.

For the `OccupancyGuidanceRegion` MonoBehaviour block: swap `OccupancyGuidanceZone`'s GUID for
`RewardOccupancyGuidanceZone`'s GUID. Update `m_EditorClassIdentifier:` from
`Assembly-CSharp::OccupancyGuidanceZone` to `Assembly-CSharp::RewardOccupancyGuidanceZone`.

The root `StimulusTriggerZone` MonoBehaviour is unchanged — it still gets the polymorphic
`RewardOccupancyZone` via the inherited typed lookup.

### Step 6: Validate via `inspect_prefab_tool`

Expect a `RewardOccupancyTriggerZone` root with `StimulusTriggerZone` in its `components` list, a
`RewardOccupancyRegion` child with `RewardOccupancyZone` in `components`, and an
`OccupancyGuidanceRegion` grandchild with `RewardOccupancyGuidanceZone` in `components`. The
hierarchy depth and parent-child fileID pairings match the canonical occupancy prefab exactly.

### Step 7: Wire downstream per `SKILL.md` Step 7

- New `TriggerType` member (via `/library-extension`).
- New `BuildSegmentPrefabs` branch + `PlaceRewardOccupancyZone` helper (via `/task-generator`).
- `DeleteProtectedPaths` entry for `RewardOccupancyTriggerZone.prefab`.
- `ConfigLoader.ValidateTemplate` literal-check accepts the new `trigger_type` value.
- **No** `ResetZone.cs` edit needed — `RewardOccupancyZone` and `RewardOccupancyGuidanceZone`
  inherit from the canonical classes that `ResetZone.Start` already discovers polymorphically.

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
