---
name: mqtt-contract
description: >-
  Documents every MQTT topic sollertia-unity-tasks publishes or subscribes to — payload shape,
  direction, owning script — covering the bidirectional MQTT 5.0 contract with sollertia-experiment.
  All topics are flat PascalCase constants centralized in Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs.
  Use when authoring or modifying MQTT wiring, diagnosing a missed message, or adding a new
  trigger zone, lifecycle marker, or UI subscriber.
user-invocable: false
---

# Sollertia Unity MQTT contract

Documents the MQTT contract between `sollertia-unity-tasks` and `sollertia-experiment`. Every
topic is declared as a `public const string` in
`Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs`, which is the single source of truth for topic names,
payload shapes, and direction. This skill is the audit-ready mirror of that file.

**Reference-only skill.** No upstream — agents arrive here on demand from `/task-prefabs`
(generated zone scripts own these topics), `/scene-setup` (`UI-lick-reward` subscribes to
`Interaction` / `Stimulus`), `/task-parameters` (runtime alternative for `RequireInteraction` / `RequireWait`),
and `/play-mode` (mid-run flag flips).

---

## Scope

**Covers:**
- Every MQTT topic published or subscribed to by `sollertia-unity-tasks` scripts
- Payload shapes (trigger-only vs JSON-serialized typed messages)
- Owning script and initialization site for each channel
- Required topic conventions (flat PascalCase, no trailing slash, centralized constants)
- Diagnostic guidance when a message is sent from one side but not received on the other
- MQTT 5.0 protocol requirement and the broker's in-process loopback fallback

**Does not cover:**
- The `MQTTChannel` and `MQTTChannel<T>` class APIs (see `/gimbl-framework`)
- MQTT broker installation or configuration (see the project `README.md`)
- `sollertia-experiment`'s publishing side (owned by sollertia-experiment skills)
- The Task Parameters MCP surface that mirrors `RequireInteraction` / `RequireWait` at editor time
  (see `/task-parameters`)

---

## Topic conventions

- **Flat PascalCase identifiers**, no slashes (e.g., `Interaction`, `Stimulus`, `CueSequenceTrigger`). MQTT
  brokers treat `X` and `X/` as distinct topics; the flat convention removes a class of accidental
  routing mismatches between Unity and external publishers.
- **Centralized constants**: Every topic literal lives in `Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs`
  as a `public const string`. You MUST reference the constant (`MQTTTopics.CueSequence`) and MUST
  NOT hardcode a string literal — a rename here propagates automatically, a hand-typed literal
  does not. Each constant carries `Direction`, `Payload`, and `Callers` XML remarks, and you MUST
  keep those accurate when adding or modifying topics.
- **Case-sensitive routing**: `MQTTClient` compares topic strings with
  `string.Equals(..., StringComparison.Ordinal)` on both the broker and in-process loopback paths
  (`MQTTClient.cs:188` and `MQTTClient.cs:298`). Centralized constants make this invisible to
  Unity callers, but ad-hoc tools (`mosquitto_pub`, dashboards, hand-typed test publishers) must
  match the casing exactly — `interaction` and `Interaction` are different topics.
- **Trigger pairs**: "Trigger" topics come in pairs of `<Name>Trigger` (subscriber that asks
  Unity to publish) and `<Name>` (publisher that responds). The former carries no payload; the
  latter carries a JSON-serialized message. Lifecycle markers (`SessionStart` / `SessionStop`)
  are not trigger pairs — they are one-shot lifecycle notifications.
- **Subscription QoS vs publish QoS**: The `qosLevel` constructor parameter on `MQTTChannel`
  (default `2`) is the **subscription** QoS only. The **publish** QoS is hardcoded to
  `MqttQualityOfServiceLevel.ExactlyOnce` inside `MQTTClient.Publish` and cannot be lowered
  per-channel. A "QoS mismatch" symptom from `sollertia-experiment` therefore points at the
  experiment-side publisher's QoS, not the Unity channel constructor.

For the `MQTTChannel` / `MQTTChannel<T>` class API, the MQTT 5.0 protocol requirement, the
in-process loopback fallback, the `JsonUtility`-needs-public-fields constraint, and the
`MQTTClient` lifecycle (`Awake` → `OnEnable` connect → `Start`), see `/gimbl-framework`. This
skill consumes those primitives; it does not redocument them.

---

## Topic catalog

The full catalog matches `MQTTTopics.cs`. Each row names the topic constant, its direction relative
to Unity, the channel type, the payload shape, and the script(s) that publish or subscribe.

### Session lifecycle (owned by `Gimbl.MQTTClient`)

| Constant       | Direction          | Channel type  | Payload | Publisher                                          | Subscriber           |
|----------------|--------------------|---------------|---------|----------------------------------------------------|----------------------|
| `SessionStart` | Unity → experiment | `MQTTChannel` | empty   | `MQTTClient.StartSessionAsync` (1 s after `Start`) | sollertia-experiment |
| `SessionStop`  | Unity → experiment | `MQTTChannel` | empty   | `MQTTClient.OnApplicationQuit`                     | sollertia-experiment |

Both are fire-and-forget lifecycle markers — no payload, no acknowledgement contract on the Unity
side. `sollertia-experiment` uses them to bracket per-session data acquisition.

### Treadmill input (owned by `Gimbl.LinearTreadmill` and `Gimbl.SimulatedLinearTreadmill`)

| Constant | Direction        | Channel type                                    | Payload                 | Publisher                               | Subscriber                  |
|----------|------------------|-------------------------------------------------|-------------------------|-----------------------------------------|-----------------------------|
| `Motion` | Unity ← hardware | `MQTTChannel<LinearTreadmill.TreadmillMessage>` | `{ "movement": float }` | sollertia-experiment / treadmill driver | `LinearTreadmill.OnMessage` |

`SimulatedLinearTreadmill` intentionally does **not** subscribe to `Motion` — the simulated rig
drives movement from keyboard input via Unity's Input System. The non-chaining `Start` contract
between `LinearTreadmill` and `SimulatedLinearTreadmill` is documented inline in
`LinearTreadmill.cs`; you MUST NOT promote either method to `virtual` without re-reading that
note.

### Interaction / stimulus (owned by `SL.Tasks.StimulusTriggerZone` and `Gimbl.SimulatedLinearTreadmill`)

| Constant      | Direction          | Channel type                                       | Payload                   | Publisher(s)                                                                   | Subscriber(s)                                                                            |
|---------------|--------------------|----------------------------------------------------|---------------------------|--------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| `Interaction` | bidirectional      | `MQTTChannel`                                      | empty                     | sollertia-experiment hardware lickport; `SimulatedLinearTreadmill` Jump action | `SL.Tasks.StimulusTriggerZone.OnInteractionDetected`; `SL.UI.LickStimulusSpawner.OnLick` |
| `Stimulus`    | Unity → experiment | `MQTTChannel<StimulusTriggerZone.StimulusMessage>` | `{ "trialName": string }` | `SL.Tasks.StimulusTriggerZone.TriggerStimulus`                                 | sollertia-experiment; `SL.UI.LickStimulusSpawner.OnStimulus` (intra-Unity)               |

Both topics are multi-subscriber — see [Multi-consumer topics](#multi-consumer-topics) for the
full subscriber map. `Interaction` is bidirectional because the simulated treadmill publishes
synthetic licks during keyboard-only runs while hardware publishes them in production. The
`Stimulus` payload carries the firing zone's owning trial name (set on `StimulusTriggerZone.trialName`
by `CreateTask` at generation), so the stimulus identifier is the trial name; `sollertia-experiment`
parses it into `VRTaskEvent.trial_name` and resolves the per-trial outcome from it.

### Occupancy guidance brake (owned by `SL.Tasks.OccupancyGuidanceZone`)

| Constant | Direction          | Channel type                                             | Payload                         | Publisher                                                                                                       | Subscriber           |
|----------|--------------------|----------------------------------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|----------------------|
| `Delay`  | Unity → experiment | `MQTTChannel<OccupancyGuidanceZone.TriggerDelayMessage>` | `{ "delayMilliseconds": uint }` | `OccupancyGuidanceZone.TriggerBrakeActivation` (only in guidance mode, and only while occupancy is not yet met) | sollertia-experiment |

Published exactly once per lap when the animal enters the occupancy-guidance zone while
`Task.requireWait == false` and the parent `OccupancyZone.occupancyMet == false`. The payload
field is `delayMilliseconds` (unsigned int); `sollertia-experiment` reads it to brake the treadmill
for the remaining occupancy duration. The guidance brake is shared by all three occupancy trigger
modes (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`) — the per-mode firing rule lives in
the parent `StimulusTriggerZone`, not in the brake.

### Task lifecycle (owned by `SL.Tasks.Task`)

| Constant             | Direction          | Channel type                         | Payload                     | Publisher                   | Subscriber                                             |
|----------------------|--------------------|--------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------|
| `CueSequenceTrigger` | Unity ← experiment | `MQTTChannel`                        | empty                       | sollertia-experiment        | `Task.OnCueSequenceTrigger` (replies on `CueSequence`) |
| `CueSequence`        | Unity → experiment | `MQTTChannel<Task.SequenceMessage>`  | `{ "cueSequence": byte[] }` | `Task.OnCueSequenceTrigger` | sollertia-experiment                                   |
| `SceneNameTrigger`   | Unity ← experiment | `MQTTChannel`                        | empty                       | sollertia-experiment        | `Task.OnSceneNameTrigger` (replies on `SceneName`)     |
| `SceneName`          | Unity → experiment | `MQTTChannel<Task.SceneNameMessage>` | `{ "name": string }`        | `Task.OnSceneNameTrigger`   | sollertia-experiment                                   |
| `RequireInteraction` | Unity ← experiment | `MQTTChannel<Task.BoolMessage>`      | `{ "value": bool }`         | sollertia-experiment        | `Task.OnRequireInteraction`                            |
| `RequireWait`        | Unity ← experiment | `MQTTChannel<Task.BoolMessage>`      | `{ "value": bool }`         | sollertia-experiment        | `Task.OnRequireWait`                                   |

Payload classes (all defined as public nested classes inside `SL.Tasks.Task`):

```csharp
/// <summary>Wraps cue sequence data for MQTT transmission.</summary>
public class SequenceMessage
{
    /// <summary>The byte array containing the encoded cue sequence for the entire track.</summary>
    public byte[] cueSequence;
}

/// <summary>Wraps scene name data for MQTT transmission.</summary>
public class SceneNameMessage
{
    /// <summary>The name of the currently active Unity scene.</summary>
    public string name;
}

/// <summary>Wraps a single boolean payload for MQTT toggle channels.</summary>
public class BoolMessage
{
    /// <summary>The boolean payload value.</summary>
    public bool value;
}
```

Each `RequireInteraction` / `RequireWait` toggle is a single topic whose payload's `value` field carries
the new state. Editor-time changes to the same flags are available through `/task-parameters`
(`write_task_parameters_tool`).

### UI feedback (owned by `SL.UI.LickStimulusSpawner`)

| Constant      | Direction                     | Channel type  | Subscriber action                                           |
|---------------|-------------------------------|---------------|-------------------------------------------------------------|
| `Interaction` | Unity ← experiment / sim      | `MQTTChannel` | Spawns a lick indicator on the experimenter's UI canvas     |
| `Stimulus`    | Unity ← Unity (intra-process) | `MQTTChannel` | Spawns a stimulus indicator on the experimenter's UI canvas |

`LickStimulusSpawner` never publishes. The `Stimulus` case is an **intra-Unity** subscription —
`StimulusTriggerZone` publishes, and both `LickStimulusSpawner` and `sollertia-experiment` subscribe.
This is intentional, not a bug.

---

## Multi-consumer topics

`Interaction` and `Stimulus` each have more than one subscriber inside Unity. When editing either, you
MUST verify every subscriber still behaves correctly.

### `Interaction`

Subscribers inside Unity:
- `SL.Tasks.StimulusTriggerZone.OnInteractionDetected` — records an interaction that occurred while the
  animal was inside the zone (used by interaction mode to fire the stimulus).
- `SL.UI.LickStimulusSpawner.OnLick` — spawns a UI indicator on the experimenter's canvas.

Publishers:
- sollertia-experiment hardware lickport (production).
- `Gimbl.SimulatedLinearTreadmill` — `Jump` action (spacebar) keypress during dev testing; the
  publisher uses the in-process loopback when no broker is connected.

### `Stimulus`

Subscriber inside Unity:
- `SL.UI.LickStimulusSpawner.OnStimulus` — spawns a UI indicator.

Publisher:
- `SL.Tasks.StimulusTriggerZone.TriggerStimulus` — fires once per zone activation. All five trigger
  modes (`interaction`, `collision`, `occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`) publish
  this same `Stimulus` event; only the firing condition differs (an interaction event, a boundary-wall
  collision, or occupancy met), and the wire payload is identical across modes.

External:
- sollertia-experiment subscribes to log the stimulus event against the session timeline.

---

## Lifecycle rules

`/gimbl-framework` owns the channel lifecycle invariants (construct in `Start()`, never `Awake()`;
null-conditional `?.` in `OnDestroy()`; typed channels must be stored as `MQTTChannel<TMessage>`;
typed payloads must use public fields). When adding a new topic, follow the pattern already
established in `Task.cs`:

```csharp
// Start()
_myListener = new MQTTChannel(MQTTTopics.MyTopic, isListener: true);
_myListener.receivedEvent.AddListener(OnMyEvent);

// OnDestroy()
_myListener?.receivedEvent.RemoveListener(OnMyEvent);
```

The per-symptom diagnosis playbook below names which invariant a symptom violates and points back
at `/gimbl-framework` for the underlying class-API contract.

---

## Adding a new topic

You MUST work through this checklist before introducing a new MQTT topic:

```text
- [ ] Topic name is flat PascalCase with no slashes
- [ ] Constant is declared as `public const string` in Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs
- [ ] Constant's XML remarks include Direction, Payload, and Callers entries kept accurate
- [ ] If paired with a "Trigger" topic, both are named `<Name>Trigger` and `<Name>`
- [ ] Owning script(s) identified — a topic must not be split across unrelated MonoBehaviours
- [ ] Channel direction (subscriber vs publisher) matches the intent
- [ ] Start() constructs the channel and registers AddListener callbacks; OnDestroy() removes them with `?.`
- [ ] Payload class (if typed) is a public class with public fields (JsonUtility constraint)
- [ ] `experiment:vr-driver-interface` is updated on its side (`_VRTaskMQTTTopics`) — coordinate the change across both repos
- [ ] This skill's topic catalog is updated with the new entry
- [ ] Did NOT treat Editor / keyboard-only success as proof of wiring — `MQTTClient.Publish` loops messages in-process
      when no broker is connected, so a Unity-only topic with no `sollertia-experiment` counterpart appears to work
      locally and silently drops in production; confirmed the experiment-side publisher / subscriber landed in the
      same release
```

---

## Diagnosing a missed message

When a topic appears to work on one side but not the other, locate the matching symptom below and
work through its likely cause and first check.

### Unity subscribes but never receives

- **Likely cause**: Topic string mismatch between sides, hand-typed literal drifted from the
  constant, or casing differs (`Interaction` vs `interaction` — `MQTTClient` uses `StringComparison.Ordinal`).
- **First check**: Confirm both sides reference `MQTTTopics.<Name>` (Unity) or the matching
  `sl-experiment` constant with identical casing.

### Unity publishes but experiment never receives

- **Likely cause**: Broker not connected (the publish reaches in-process subscribers only and
  never crosses to the experiment process), or experiment-side subscription not active.
- **First check**: Check the Unity Console for "Successfully connected to MQTT Broker" and any
  "MQTT publish failed" line; confirm experiment subscribed to the matching `MQTTTopics.<Name>`.

### Channel constructor throws `InvalidOperationException`

- **Likely cause**: A `MQTTChannel` was created before `MQTTClient.Instance` was set (e.g., in
  another script's `Awake`).
- **First check**: Move the channel construction into the consumer's `Start()` — Unity
  guarantees every `Awake()` runs first.

### Typed message received but every field is default

- **Likely cause**: Payload class uses `{ get; set; }` instead of public fields.
- **First check**: Grep the payload class for `{ get; set; }` and convert to public fields.

### Typed channel `Send()` succeeds but subscriber callback never fires

- **Likely cause**: Reference stored as base `MQTTChannel` instead of `MQTTChannel<TMessage>`, so
  the typed `new`-shadowed `receivedEvent` is dropped.
- **First check**: Change the field declaration to `MQTTChannel<TMessage>` and re-register
  `AddListener` on the typed event.

### Subscriber callback stops firing after scene change

- **Likely cause**: `OnDestroy()` removed the listener; the new scene's `Start()` did not
  re-register.
- **First check**: Inspect the scene's MonoBehaviour `Start()` for the missing `AddListener` call.

### Random message loss under high rate

- **Likely cause**: Experiment-side publisher running below QoS 2 (Unity publish is hardcoded
  `ExactlyOnce`; Unity subscribe defaults to QoS 2).
- **First check**: Inspect the broker's retained messages and the experiment-side publisher's
  QoS setting.

### Unity receives its own `Stimulus` publication

- **Likely cause**: Expected — `LickStimulusSpawner` is an intentional intra-Unity subscriber on
  `Stimulus`.
- **First check**: Not a bug; documented in [Multi-consumer topics](#multi-consumer-topics).

### Keyboard-only test run still triggers UI lick / stimulus

- **Likely cause**: Expected — `MQTTClient.Publish` loops messages in-process when the broker is
  unreachable.
- **First check**: Not a bug; the in-process loopback is the dev-without-broker path.

### `RequireInteraction` / `RequireWait` writes have no effect

- **Likely cause**: Payload missing the `value` field, or the JSON wrapper is malformed.
- **First check**: Confirm payload is `{"value": true}` / `{"value": false}` (case-sensitive,
  JSON-serializable).

---

## Interaction contract

- **You MUST NOT** hardcode a topic string anywhere outside `MQTTTopics.cs`. You MUST always
  reference `MQTTTopics.<Name>` so a rename propagates automatically and skill / source drift
  is impossible.
- **You MUST NOT** rely on message ordering between two different topics — MQTT provides
  per-topic ordering only.
- **You MUST NOT** add a publisher and a subscriber on the same topic inside Unity expecting
  deduplication. MQTTnet delivers the message to every subscriber, including the local one;
  this is the intended behavior for `Stimulus` and the in-process loopback path.

---

## Verification checklist

```text
- [ ] The topic catalog above matches every `public const string` in MQTTTopics.cs
- [ ] Every new channel has an entry with direction, channel type, payload, and owning script
- [ ] Payload class shapes in this file match the C# definitions (public fields only)
- [ ] Cross-Unity subscriptions (multi-consumer topics) are documented when they exist
- [ ] Changes propagate to sollertia-experiment if a topic's direction, payload, or name changes
- [ ] No hand-typed topic literals appear outside MQTTTopics.cs (grep the project for the literal)
```

---

## Related skills

| Skill                                    | Relationship                                                                        |
|------------------------------------------|-------------------------------------------------------------------------------------|
| `/gimbl-framework` (this plugin)         | Owns `MQTTClient`, `MQTTChannel`, and `MQTTChannel<T>` class references             |
| `/task-prefabs` (this plugin)            | Generated prefabs wire the zones whose scripts own these topics                     |
| `/task-parameters` (this plugin)         | Editor-time alternative for `RequireInteraction` / `RequireWait` flags              |
| `/scene-setup` (this plugin)             | `UI-lick-reward` subsystem subscribes to `Interaction` and `Stimulus`               |
| `/play-mode` (this plugin)               | MQTT activity is only live while the Editor is in `playing` state                   |
| assets plugin `/task-templates`          | YAML cue codes appear as `byte` values in `CueSequence` payloads                    |
| experiment plugin `/vr-driver-interface` | Host (Python) side — `_VRTaskMQTTTopics` mirrors this catalog; change both together |
