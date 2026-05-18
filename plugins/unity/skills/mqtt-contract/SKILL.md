---
name: mqtt-contract
description: >-
  Documents every MQTT topic sollertia-unity-tasks publishes or subscribes to — payload shape,
  direction, owning script — covering the bidirectional MQTT 5.0 contract with sollertia-experiment.
  All topics are flat PascalCase constants centralized in Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs.
  Use when authoring or modifying MQTT wiring, diagnosing a missed message, or adding a new
  trigger zone, lifecycle marker, or UI listener.
user-invocable: true
---

# Sollertia Unity MQTT contract

Documents the MQTT contract between `sollertia-unity-tasks` and `sollertia-experiment`. Every
topic is declared as a `public const string` in
`Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs`, which is the single source of truth for topic names,
payload shapes, and direction. This skill is the audit-ready mirror of that file.

---

## Scope

**Covers:**
- Every MQTT topic published or subscribed to by `sollertia-unity-tasks` scripts
- Payload shapes (trigger-only vs JSON-serialized typed messages)
- Owning script and initialization site for each channel
- Required topic conventions (flat PascalCase, no trailing slash, centralized constants)
- Diagnostic checklist when a message is sent from one side but not received on the other
- MQTT 5.0 protocol requirement and the broker's in-process loopback fallback

**Does not cover:**
- The `MQTTChannel` and `MQTTChannel<T>` class APIs (see `/gimbl-framework`)
- MQTT broker installation or configuration (see the project `README.md`)
- `sollertia-experiment`'s publishing side (owned by sollertia-experiment skills)
- The Task Parameters MCP surface that mirrors `RequireLick` / `RequireWait` at editor time
  (see `/task-parameters`)

---

## Topic conventions

- **Flat PascalCase identifiers**, no slashes (e.g., `Lick`, `Stimulus`, `CueSequenceTrigger`). MQTT
  brokers treat `X` and `X/` as distinct topics; the flat convention removes a class of accidental
  routing mismatches between Unity and external publishers. The previous trailing-slash and
  nested `/True` / `/False` patterns were retired with the MQTT topic standardization pass.
- **Centralized constants**: Every topic literal lives in `Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs`
  as a `public const string`. Reference the constant
  (`MQTTTopics.CueSequence`), never a string literal. A rename here propagates automatically; a
  hand-typed literal does not. Each constant carries `Direction`, `Payload`, and `Callers` XML
  remarks — keep those accurate when adding or modifying topics.
- **Trigger pairs**: "Trigger" topics come in pairs of `<Name>Trigger` (listener that asks Unity
  to publish) and `<Name>` (publisher that responds). The former carries no payload; the latter
  carries a JSON-serialized message. Lifecycle markers (`SessionStart` / `SessionStop`) are not
  trigger pairs — they are one-shot lifecycle notifications.
- **MQTT 5.0 only**: `MQTTClient` connects with `MqttProtocolVersion.V500`. Brokers must accept
  MQTT 5.0 connections (Mosquitto 2.0+). This matches the `sollertia-experiment` MQTT runtime.
- **Channel types**: `MQTTChannel` (untyped) sends and receives empty-payload trigger messages.
  `MQTTChannel<T>` sends and receives `Encoding.UTF8.GetBytes(JsonUtility.ToJson(message))` — a
  JSON payload whose deserialization target is `T`. The payload type must use **public fields**;
  `JsonUtility` does not read properties.
- **In-process loopback**: When the broker is unreachable, `MQTTClient.Publish` routes the message
  directly to every in-process subscriber on the matching topic so keyboard-only test runs without
  a broker still reach local listeners. Production runs with a real broker go through MQTT as
  normal. See [Diagnosing a missed message](#diagnosing-a-missed-message).

---

## Topic catalog

The full catalog matches `MQTTTopics.cs`. Each row names the topic constant, its direction relative
to Unity, the channel type, the payload shape, and the script(s) that publish or subscribe.

### Session lifecycle (owned by `Gimbl.MQTTClient`)

| Constant       | Direction        | Channel type   | Payload | Publisher                          | Subscriber                         |
|----------------|------------------|----------------|---------|------------------------------------|------------------------------------|
| `SessionStart` | Unity → experiment | `MQTTChannel` | empty   | `MQTTClient.StartSessionAsync` (1 s after `Start`) | sollertia-experiment              |
| `SessionStop`  | Unity → experiment | `MQTTChannel` | empty   | `MQTTClient.OnApplicationQuit`     | sollertia-experiment               |

Both are fire-and-forget lifecycle markers — no payload, no ack, no acknowledgement contract on
the Unity side. `sollertia-experiment` uses them to bracket per-session data acquisition.

### Treadmill input (owned by `Gimbl.LinearTreadmill` and `Gimbl.SimulatedLinearTreadmill`)

| Constant | Direction          | Channel type                       | Payload                                                 | Publisher                                  | Subscriber                  |
|----------|--------------------|------------------------------------|---------------------------------------------------------|--------------------------------------------|------------------------------|
| `Motion` | Unity ← hardware   | `MQTTChannel<LinearTreadmill.TreadmillMessage>` | `{ "movement": float }`                          | sollertia-experiment / treadmill driver    | `LinearTreadmill.OnMessage` |

`SimulatedLinearTreadmill` intentionally does **not** subscribe to `Motion` — the simulated rig
drives movement from keyboard input via Unity's Input System. The non-chaining
`Start` contract between `LinearTreadmill` and `SimulatedLinearTreadmill` is documented inline in
`LinearTreadmill.cs`; do not promote either method to `virtual` without re-reading that note.

### Lick / stimulus (owned by `SL.Tasks.StimulusTriggerZone` and `Gimbl.SimulatedLinearTreadmill`)

| Constant   | Direction          | Channel type   | Payload | Publisher(s)                                                                | Subscriber(s)                                                              |
|------------|--------------------|----------------|---------|-----------------------------------------------------------------------------|----------------------------------------------------------------------------|
| `Lick`     | bidirectional      | `MQTTChannel`  | empty   | sollertia-experiment hardware lickport; `SimulatedLinearTreadmill` Jump action       | `SL.Tasks.StimulusTriggerZone.OnLickDetected`; `SL.UI.LickStimulusSpawner.OnLick` |
| `Stimulus` | Unity → experiment | `MQTTChannel`  | empty   | `SL.Tasks.StimulusTriggerZone.TriggerStimulus`                              | sollertia-experiment; `SL.UI.LickStimulusSpawner.OnStimulus` (intra-Unity)  |

Both topics are multi-consumer — see [Multi-consumer topics](#multi-consumer-topics) for the full
listener map. `Lick` is bidirectional because the simulated treadmill publishes synthetic licks
during keyboard-only runs while hardware publishes them in production.

### Occupancy guidance brake (owned by `SL.Tasks.OccupancyGuidanceZone`)

| Constant | Direction          | Channel type                                           | Payload                       | Publisher                                                           | Subscriber           |
|----------|--------------------|--------------------------------------------------------|-------------------------------|---------------------------------------------------------------------|----------------------|
| `Delay`  | Unity → experiment | `MQTTChannel<OccupancyGuidanceZone.TriggerDelayMessage>` | `{ "delayMilliseconds": uint }` | `OccupancyGuidanceZone.TriggerBrakeActivation` (only in guidance mode and only when the boundary is still armed) | sollertia-experiment |

Published exactly once per lap when the animal enters the occupancy-guidance zone while
`Task.requireWait == false` and the parent `OccupancyZone.boundaryDisarmed == false`. The payload
field is `delayMilliseconds` (unsigned int); `sollertia-experiment` reads it to brake the treadmill
for the remaining occupancy duration.

### Task lifecycle (owned by `SL.Tasks.Task`)

| Constant             | Direction          | Channel type                       | Payload                          | Publisher                                | Subscriber                                |
|----------------------|--------------------|------------------------------------|----------------------------------|------------------------------------------|--------------------------------------------|
| `CueSequenceTrigger` | Unity ← experiment | `MQTTChannel`                      | empty                            | sollertia-experiment                     | `Task.OnCueSequenceTrigger` (replies on `CueSequence`) |
| `CueSequence`        | Unity → experiment | `MQTTChannel<Task.SequenceMessage>` | `{ "cueSequence": byte[] }`      | `Task.OnCueSequenceTrigger`              | sollertia-experiment                       |
| `SceneNameTrigger`   | Unity ← experiment | `MQTTChannel`                      | empty                            | sollertia-experiment                     | `Task.OnSceneNameTrigger` (replies on `SceneName`)     |
| `SceneName`          | Unity → experiment | `MQTTChannel<Task.SceneNameMessage>` | `{ "name": string }`             | `Task.OnSceneNameTrigger`                | sollertia-experiment                       |
| `RequireLick`        | Unity ← experiment | `MQTTChannel<Task.BoolMessage>`    | `{ "value": bool }`              | sollertia-experiment                     | `Task.OnRequireLick`                       |
| `RequireWait`        | Unity ← experiment | `MQTTChannel<Task.BoolMessage>`    | `{ "value": bool }`              | sollertia-experiment                     | `Task.OnRequireWait`                       |

Payload classes (all defined as public nested classes inside `SL.Tasks.Task`):

```csharp
// CueSequence
public class SequenceMessage { public byte[] cueSequence; }

// SceneName
public class SceneNameMessage { public string name; }

// RequireLick / RequireWait
public class BoolMessage { public bool value; }
```

The previous `RequireLick/True/`, `RequireLick/False/`, `RequireWait/True/`, `RequireWait/False/`
four-topic pattern has been **retired**. Each toggle is now a single topic whose payload's `value`
field carries the new state, mirroring the standard MQTT JSON-payload pattern. Editor-time changes
to the same flags are available through `/task-parameters` (`write_task_parameters_tool`).

### UI feedback (owned by `SL.UI.LickStimulusSpawner`)

| Constant   | Direction               | Channel type   | Subscriber action                                            |
|------------|-------------------------|----------------|--------------------------------------------------------------|
| `Lick`     | Unity ← experiment / sim | `MQTTChannel` | Spawns a lick indicator on the experimenter's UI canvas      |
| `Stimulus` | Unity ← Unity (intra-process) | `MQTTChannel` | Spawns a stimulus indicator on the experimenter's UI canvas |

`LickStimulusSpawner` never publishes. The `Stimulus` case is an **intra-Unity** subscription —
`StimulusTriggerZone` publishes, and both `LickStimulusSpawner` and `sollertia-experiment` subscribe.
This is intentional, not a bug.

---

## Multi-consumer topics

`Lick` and `Stimulus` each have more than one subscriber inside Unity. When editing either, verify
every listener still behaves correctly:

### `Lick`

Subscribers inside Unity:
- `SL.Tasks.StimulusTriggerZone.OnLickDetected` — records a lick that occurred while the animal
  was inside the zone (used by lick mode to fire the stimulus).
- `SL.UI.LickStimulusSpawner.OnLick` — spawns a UI indicator on the experimenter's canvas.

Publishers:
- sollertia-experiment hardware lickport (production).
- `Gimbl.SimulatedLinearTreadmill` — `Jump` action (spacebar) keypress during dev testing; the
  publisher uses the in-process loopback when no broker is connected.

### `Stimulus`

Subscriber inside Unity:
- `SL.UI.LickStimulusSpawner.OnStimulus` — spawns a UI indicator.

Publisher:
- `SL.Tasks.StimulusTriggerZone.TriggerStimulus` — fires once per armed zone entry.

External:
- sollertia-experiment subscribes to log the stimulus event against the session timeline.

---

## Lifecycle rules

All channels are created in `Start()` (runtime, not editor) and removed in `OnDestroy()`. Add a
new channel by following the pattern already established in `Task.cs`:

```csharp
// Start()
_myListener = new MQTTChannel(MQTTTopics.MyTopic, isListener: true);
_myListener.receivedEvent.AddListener(OnMyEvent);

// OnDestroy()
_myListener?.receivedEvent.RemoveListener(OnMyEvent);
```

- Call `AddListener` after the channel is constructed, never in the constructor.
- Use the null-conditional `?.` in `OnDestroy` — `MQTTClient.Instance` may have torn down before
  the consumer's destructor runs (common in scene transitions).
- Typed channels (`MQTTChannel<T>`) must have a `T` decorated with plain public fields (no
  `{ get; set; }` auto-properties), because `JsonUtility` requires them.
- A `new MQTTChannel(...)` call before `MQTTClient.Instance` is set throws
  `InvalidOperationException` — create channels in `Start()` (which Unity guarantees runs after
  every `Awake()`), never in `Awake()` or a constructor.

---

## Adding a new topic

Follow this checklist before introducing a new MQTT topic:

```text
- [ ] Topic name is flat PascalCase with no slashes
- [ ] Constant is declared as `public const string` in Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs
- [ ] Constant's XML remarks include Direction, Payload, and Callers entries kept accurate
- [ ] If paired with a "Trigger" topic, both are named `<Name>Trigger` and `<Name>`
- [ ] Owning script(s) identified — a topic must not be split across unrelated MonoBehaviours
- [ ] Channel direction (listener vs publisher) matches the intent
- [ ] Start() constructs the channel and registers listeners; OnDestroy() removes them with `?.`
- [ ] Payload class (if typed) is a public class with public fields (JsonUtility constraint)
- [ ] sollertia-experiment is aware of the topic on its side — coordinate the change across both repos
- [ ] This skill's topic catalog is updated with the new entry
```

---

## Diagnosing a missed message

When a topic appears to work on one side but not the other, the probable causes cluster by
symptom:

| Symptom                                                  | Likely cause                                                                                  | First check                                                                                  |
|----------------------------------------------------------|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Unity subscribes but never receives                       | Topic string mismatch between sides, or hand-typed literal drifted from the constant         | Confirm both sides reference `MQTTTopics.<Name>` (Unity) or the matching `sl-experiment` constant |
| Unity publishes but experiment never receives             | `MQTTClient.Instance` not initialized before `Start()`, or broker not connected               | Check the Unity Console for the "Successfully connected to MQTT Broker" line and the "MQTT subscribe failed" warning |
| Typed message received but every field is default        | Payload class uses `{ get; set; }` instead of public fields                                   | Grep the payload class for `{ get; set; }` and convert to public fields                       |
| Listener stops firing after scene change                  | `OnDestroy()` removed the listener; the new scene's `Start()` did not re-register             | Inspect the scene's MonoBehaviour `Start()` for the missing `AddListener` call                |
| Random message loss under high rate                       | QoS level mismatch (default QoS is 2 on both sides — `ExactlyOnce`)                          | Inspect the broker's retained messages and the publisher's QoS setting                        |
| Unity receives its own `Stimulus` publication             | Expected — `LickStimulusSpawner` is an intentional intra-Unity subscriber on `Stimulus`       | Not a bug; documented in [Multi-consumer topics](#multi-consumer-topics)                      |
| Keyboard-only test run still triggers UI lick / stimulus  | Expected — `MQTTClient.Publish` loops messages in-process when the broker is unreachable      | Not a bug; the in-process loopback is the dev-without-broker path                             |
| `RequireLick` / `RequireWait` writes have no effect       | Payload missing the `value` field, or the JSON wrapper is malformed                           | Confirm payload is `{"value": true}` / `{"value": false}` (case-sensitive, JSON-serializable) |

---

## Interaction contract

- **Never** hardcode a topic string anywhere outside `MQTTTopics.cs`. Always reference
  `MQTTTopics.<Name>` so a rename propagates automatically and skill / source drift is impossible.
- **Never** rely on message ordering between two different topics — MQTT provides per-topic ordering
  only.
- **Never** add a publisher and a subscriber on the same topic inside Unity expecting deduplication.
  MQTTnet delivers the message to every subscriber, including the local one. This is the intended
  behavior for `Stimulus` and the in-process loopback path.
- **Never** flip a `BoolMessage` topic from a generic payload to a topic-suffix pattern (e.g.,
  splitting `RequireLick` back into `RequireLick/True` / `RequireLick/False`). The standardization
  pass on 2026-05-17 retired those splits in favor of a single topic with a payload `value` —
  reverting would require an in-lockstep update on `sollertia-experiment`.

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

| Skill                                   | Relationship                                                                            |
|-----------------------------------------|------------------------------------------------------------------------------------------|
| `/gimbl-framework` (this plugin)        | Owns `MQTTClient`, `MQTTChannel`, and `MQTTChannel<T>` class references                  |
| `/task-prefabs` (this plugin)           | Generated prefabs wire the zones whose scripts own these topics                          |
| `/task-parameters` (this plugin)        | Editor-time alternative for `RequireLick` / `RequireWait` flags                          |
| `/scene-setup` (this plugin)            | `UI-lick-reward` subsystem subscribes to `Lick` and `Stimulus`                           |
| `/play-mode` (this plugin)              | MQTT activity is only live while the Editor is in `playing` state                        |
| assets plugin `/task-templates`         | YAML cue codes appear as `byte` values in `CueSequence` payloads                         |
