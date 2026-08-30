---
name: mqtt-contract
description: >-
  Documents every MQTT topic to which sollertia-virtual-reality publishes or subscribes, with its payload shape,
  direction, and owning script, covering the bidirectional MQTT 5.0 contract with sollertia-experiment. All topics are
  flat PascalCase constants centralized in Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs. Use when authoring or modifying MQTT
  wiring, diagnosing a missed message, or adding a new trigger zone, lifecycle marker, or UI subscriber.
user-invocable: false
---

# Sollertia Unity MQTT contract

Documents the MQTT contract between `sollertia-virtual-reality` and `sollertia-experiment`. Every topic is declared as a
`public const string` in `Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs`, which is the single source of truth for topic names,
payload shapes, and direction. This skill is the audit-ready mirror of that file.

**Reference-only skill.** No upstream. Agents arrive here on demand from `/task-prefabs` (generated zone scripts own
these topics), `/scene-setup` (`UI-lick-reward` subscribes to `Interaction` / `Stimulus`), `/task-parameters` (runtime
alternative for `RequireInteraction` / `RequireWait`), and `/play-mode` (mid-run flag flips).

---

## Scope

**Covers:**
- Every MQTT topic with a `sollertia-virtual-reality` publisher or subscriber
- Payload shapes (trigger-only vs JSON-serialized typed messages)
- Owning script and initialization site for each channel
- Required topic conventions (flat PascalCase, no trailing slash, centralized constants)
- Diagnostic guidance when a message is sent from one side but not received on the other
- The in-process loopback's consequences for topic wiring and diagnosis (the mechanism itself lives in
  `/gimbl-framework`)

**Does not cover:**
- The `MQTTChannel` and `MQTTChannel<T>` class APIs (see `/gimbl-framework`)
- The MQTT 5.0 protocol requirement and the in-process loopback mechanism (see `/gimbl-framework`)
- MQTT broker installation or configuration (see the project `README.md`)
- `sollertia-experiment`'s publishing / subscribing side and the `_VRTaskMQTTTopics` mirror (see
  `experiment:vr-driver-interface`)
- The Task Parameters MCP surface that mirrors `RequireInteraction` / `RequireWait` at editor time (see
  `/task-parameters`)
- Authoring the zone prefab, its modifier script, and the generator placement branch whose channels carry these topics
  (see `/zone-prefabs`, `/task-generator`)

---

## Topic conventions

- **Flat PascalCase identifiers**, no slashes (e.g., `Interaction`, `Stimulus`, `CueSequenceTrigger`). MQTT brokers
  treat `X` and `X/` as distinct topics, and the flat convention removes a class of accidental routing mismatches
  between Unity and external publishers. The rule is machine-enforced by
  `MQTTTopicsTests.DeclaredTopics_EveryLiteral_IsASinglePascalCaseIdentifier`, which matches every literal against
  `^[A-Z][A-Za-z0-9]*$` (`MQTTTopicsTests.cs`) and so also bars whitespace and the MQTT wildcards `#` and `+`.
- **Centralized constants**: Every topic literal lives in `Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs` as a `public const
  string`. You MUST reference the constant (`MQTTTopics.CueSequence`) and MUST NOT hardcode a string literal. A rename
  here propagates automatically, while a hand-typed literal does not. Each constant carries `Direction`, `Payload`, and
  `Callers` XML remarks, and you MUST keep those accurate when adding or modifying topics. The constant's C# identifier
  MUST be identical to its literal value (`public const string Interaction = "Interaction";`).
  `MQTTTopicsTests.DeclaredTopics_EveryFieldName_EqualsItsLiteralValue` asserts `field.Name == field.GetValue(null)`
  (`MQTTTopicsTests.cs`).
- **Case-sensitive routing**: `MQTTClient` compares topic strings with `string.Equals(..., StringComparison.Ordinal)` on
  both the broker and in-process loopback paths (the `_messageReceivedHandler` routing loop and the loopback routing
  loop in `MQTTClient.Publish`, both in `MQTTClient.cs`). Centralized constants make this invisible to Unity callers,
  but ad-hoc tools (`mosquitto_pub`, dashboards, hand-typed test publishers) must match the casing exactly, because
  `interaction` and `Interaction` are different topics.
- **Callbacks arrive off the main thread**: a broker-delivered `receivedEvent` callback runs on an MQTTnet worker thread
  (`MQTTClient.cs` routes inside its `_messageReceivedHandler`), so a subscriber MUST NOT touch the Unity API
  inside the callback. The prohibition covers `Instantiate`, transform writes, and any scene mutation. Record the event
  with `Interlocked` / `Volatile` (`LickStimulusSpawner.OnLick` and `LickStimulusSpawner.OnStimulus`, with the
  reasoning in the `_pendingLickCount` remarks in `LickStimulusSpawner.cs`) or under a lock
  (`LinearTreadmill.OnMessage` in `LinearTreadmill.cs`), then act on it in `Update`.
- **Trigger pairs**: "Trigger" topics come in pairs of `<Name>Trigger` (subscriber that asks Unity to publish) and
  `<Name>` (publisher that responds). The former carries no payload, and the latter carries a JSON-serialized message.
  Lifecycle markers (`SessionStart` / `SessionStop`) are not trigger pairs, because they are one-shot lifecycle
  notifications.
- **Subscription QoS vs publish QoS**: The `qosLevel` constructor parameter on `MQTTChannel` (default `2`) is the
  **subscription** QoS only. The **publish** QoS is hardcoded to `MqttQualityOfServiceLevel.ExactlyOnce` inside
  `MQTTClient.Publish` and cannot be lowered per-channel. A "QoS mismatch" symptom from `sollertia-experiment` therefore
  points at the experiment-side publisher's QoS, not the Unity channel constructor.

For the `MQTTChannel` / `MQTTChannel<T>` class API, the MQTT 5.0 protocol requirement, the in-process loopback fallback,
the `JsonUtility`-needs-public-fields constraint, and the `MQTTClient` lifecycle (`MQTTClient.Awake` →
`MQTTConnectorObject.OnEnable` connect → subscriber `Start`), see `/gimbl-framework`. This skill consumes those
primitives rather than redocumenting them.

---

## Topic catalog

The full catalog matches `MQTTTopics.cs`. Each row names the topic constant, its direction relative to Unity, the
channel type, the payload shape, and the script(s) that publish or subscribe.

### Session lifecycle (owned by `Gimbl.MQTTClient`)

| Constant       | Direction          | Channel type  | Payload | Publisher(s)                         | Subscriber(s)        |
|----------------|--------------------|---------------|---------|--------------------------------------|----------------------|
| `SessionStart` | Unity → experiment | `MQTTChannel` | empty   | `Gimbl.MQTTClient.StartSessionAsync` | sollertia-experiment |
| `SessionStop`  | Unity → experiment | `MQTTChannel` | empty   | `Gimbl.MQTTClient.OnApplicationQuit` | sollertia-experiment |

Both are fire-and-forget lifecycle markers, with no payload and no acknowledgement contract on the Unity side.
`SessionStart` is published roughly one second after the scene's `Start()`, because `StartSessionAsync` awaits
`Task.Delay(1000)` before sending (`MQTTClient.cs`). A counterparty that samples immediately after commanding Play
Mode MUST therefore wait at least that long before treating its absence as a failure. `sollertia-experiment` waits on
`SessionStart` as the authoritative signal that Unity is armed (a bounded readiness wait inside its setup handshake) and
surfaces `SessionStop` as a `UNITY_TERMINATED` event that triggers its emergency-pause path.

### Treadmill input (owned by `Gimbl.LinearTreadmill` and `Gimbl.SimulatedLinearTreadmill`)

| Constant | Direction        | Channel type                                    | Payload                 | Publisher                               | Subscriber                  |
|----------|------------------|-------------------------------------------------|-------------------------|-----------------------------------------|-----------------------------|
| `Motion` | Unity ← hardware | `MQTTChannel<LinearTreadmill.TreadmillMessage>` | `{ "movement": float }` | sollertia-experiment / treadmill driver | `LinearTreadmill.OnMessage` |

`SimulatedLinearTreadmill` intentionally does **not** subscribe to `Motion`, because the simulated rig drives movement
from keyboard input via Unity's Input System. The non-chaining `Start` contract between `LinearTreadmill` and
`SimulatedLinearTreadmill` is documented inline in `LinearTreadmill.cs`, and you MUST NOT promote either method to
`virtual` without re-reading that note.

### Interaction / stimulus (owned by `SL.Tasks.StimulusTriggerZone` and `Gimbl.SimulatedLinearTreadmill`)

| Constant      | Direction     | Channel type                                       | Payload                                                       | Publisher(s)                                                                    | Subscriber(s)                                                                            |
|---------------|---------------|----------------------------------------------------|---------------------------------------------------------------|---------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| `Interaction` | bidirectional | `MQTTChannel`                                      | empty                                                         | sollertia-experiment interaction sensor, `SimulatedLinearTreadmill` Jump action | `SL.Tasks.StimulusTriggerZone.OnInteractionDetected`, `SL.UI.LickStimulusSpawner.OnLick` |
| `Stimulus`    | bidirectional | `MQTTChannel<StimulusTriggerZone.StimulusMessage>` | `{ "trialName": string, "delivered": bool, "cause": string }` | `SL.Tasks.StimulusTriggerZone.TriggerStimulus`                                  | sollertia-experiment, `SL.UI.LickStimulusSpawner.OnStimulus` (intra-Unity)               |

Both topics are multi-subscriber. See [Multi-consumer topics](#multi-consumer-topics) for the full subscriber map.
`Interaction` is bidirectional because the simulated treadmill publishes synthetic interactions during keyboard-only
runs while the acquisition runtime's interaction sensor publishes them in production. `Stimulus` is bidirectional
because Unity both publishes it from `StimulusTriggerZone` and subscribes to it from `LickStimulusSpawner`, while its
cross-boundary flow toward `sollertia-experiment` runs one way. The `Stimulus` payload carries three fields. The first
is the resolving trial's name (`trialName`, set on `StimulusTriggerZone.trialName` by `CreateTask` at generation, so the
stimulus identifier is the trial name). The other two are a `delivered` flag (whether the physical stimulus fired or was
omitted) and a `cause` string (`behavior` or `guidance`). `sollertia-experiment` parses these into `VRTaskEvent` and
resolves the per-trial outcome from them.

`cause` is derived per mode, and the derivation is not "did the animal act":

- **Occupancy modes** (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`): `cause` is `guidance` exactly when
  this zone's child `OccupancyGuidanceZone` already published `Delay` earlier in the same lap, otherwise `behavior`.
  That derivation comes from
  `bool brakeGuided = _occupancyGuidanceZone != null && _occupancyGuidanceZone.BrakeTriggered;`
  in `StimulusTriggerZone.UpdateOccupancyMode`. `BrakeTriggered` latches inside `TriggerBrakeActivation` immediately
  after the `Delay` send (`OccupancyGuidanceZone.cs`). Publishing `Delay` therefore deterministically changes the
  later `Stimulus.cause` on that lap.
- **Interaction mode**: `guidance` marks the two fallback resolutions, which are entering the nested `GuidanceZone`
  while `requireInteraction` is false, or entering the stimulus zone at all when no `GuidanceZone` exists. Both
  fallbacks and both sensor-interaction branches live in `StimulusTriggerZone.UpdateInteractionMode`. `behavior` marks
  a sensor interaction inside the zone **and** the boundary-exit resolution in `StimulusTriggerZone.OnTriggerExit`,
  which reports `behavior` even when `delivered` is false, so `cause: behavior` does not imply the animal interacted.
- **Collision mode**: always `behavior` (`StimulusTriggerZone.UpdateCollisionMode`).

### Occupancy guidance brake (owned by `SL.Tasks.OccupancyGuidanceZone`)

| Constant | Direction          | Channel type                                             | Payload                         | Publisher                                                                                                       | Subscriber           |
|----------|--------------------|----------------------------------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|----------------------|
| `Delay`  | Unity → experiment | `MQTTChannel<OccupancyGuidanceZone.TriggerDelayMessage>` | `{ "delayMilliseconds": uint }` | `OccupancyGuidanceZone.TriggerBrakeActivation` (only in guidance mode, and only while occupancy is not yet met) | sollertia-experiment |

Published exactly once per lap when the animal enters the occupancy-guidance zone while `Task.requireWait == false` and
the parent `OccupancyZone.occupancyMet == false`. The payload field is `delayMilliseconds` (unsigned int), and
`sollertia-experiment` reads it to brake the treadmill for the remaining occupancy duration. The guidance brake is
shared by all three occupancy trigger modes (`occupancy_disarm`, `occupancy_arm`, `occupancy_trigger`). The per-mode
firing rule lives in the parent `StimulusTriggerZone` rather than in the brake.

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
/// <remarks>
/// Required because <see cref="UnityEngine.JsonUtility"/> cannot serialize or deserialize bare primitives at
/// the top level. The wrapper makes the value addressable via the JSON object <c>{"value": true|false}</c>
/// contract that <see cref="MQTTChannel{TMessage}"/> uses.
/// </remarks>
public class BoolMessage
{
    /// <summary>Determines whether the toggled requirement is enabled.</summary>
    public bool value;
}
```

Each `RequireInteraction` / `RequireWait` toggle is a single topic whose payload's `value` field carries the new state.
Editor-time changes to the same flags are available through `/task-parameters` (`write_task_parameters_tool`).

### UI feedback (owned by `SL.UI.LickStimulusSpawner`)

| Constant      | Direction                     | Channel type                                       | Subscriber action                                                                    |
|---------------|-------------------------------|----------------------------------------------------|--------------------------------------------------------------------------------------|
| `Interaction` | Unity ← experiment / sim      | `MQTTChannel`                                      | Spawns a lick indicator on the experimenter's UI canvas                              |
| `Stimulus`    | Unity ← Unity (intra-process) | `MQTTChannel<StimulusTriggerZone.StimulusMessage>` | Spawns a stimulus indicator on the experimenter's UI canvas when `delivered` is true |

`LickStimulusSpawner` never publishes. The `Stimulus` case is an **intra-Unity** subscription. `StimulusTriggerZone`
publishes, and both `LickStimulusSpawner` and `sollertia-experiment` subscribe. This is intentional, not a bug.

Neither callback spawns anything itself: both only bump a counter with `Interlocked.Increment`
(`LickStimulusSpawner.OnLick` and `LickStimulusSpawner.OnStimulus`) because the broker delivery path invokes them on
an MQTTnet worker thread, and `Update` drains the counters on the main thread. `OnDestroy` both removes the listeners
and releases the channels through `MQTTClient.Unsubscribe`, by way of the `ReleaseChannel` helper in
`LickStimulusSpawner.cs`. This is the reference implementation for any new subscriber.

---

## Multi-consumer topics

`Interaction` has several in-Unity subscribers (one `StimulusTriggerZone` per generated segment plus
`LickStimulusSpawner`). `Stimulus` has one in-Unity subscriber (`LickStimulusSpawner`) alongside its cross-process
subscriber. When editing either, you MUST verify every subscriber still behaves correctly.

### `Interaction`

Subscribers inside Unity:
- `SL.Tasks.StimulusTriggerZone.OnInteractionDetected` records an interaction that occurred while the animal was inside
  the zone (used by interaction mode to fire the stimulus).
- `SL.UI.LickStimulusSpawner.OnLick` records one pending lick indicator (`Interlocked.Increment` in
  `LickStimulusSpawner.cs`), and `Update` spawns it on the main thread.

Publishers:
- sollertia-experiment interaction sensor (production). The acquisition runtime resolves a concrete sensor (a lick port,
  button, lever, or pressure plate). The sensor choice is a per-system decision on the `sollertia-experiment` side and
  is invisible to Unity.
- `Gimbl.SimulatedLinearTreadmill` publishes on a `Jump` action (spacebar) keypress during dev testing, and uses the
  in-process loopback when no broker is connected.

### `Stimulus`

Subscriber inside Unity:
- `SL.UI.LickStimulusSpawner.OnStimulus` records one pending stimulus indicator when `delivered` is true
  (`LickStimulusSpawner.cs`), and `Update` spawns it on the main thread.

Publisher:
- `SL.Tasks.StimulusTriggerZone.TriggerStimulus` publishes exactly once per trial at its resolution (delivered or
  omitted) in the `interaction`, `collision`, `occupancy_disarm`, and `occupancy_arm` modes. The `occupancy_trigger`
  mode publishes at most once per trial, because `UpdateOccupancyMode` sends only when occupancy is met and leaves the
  not-met case for the driver to infer. All five modes publish this same `Stimulus` event with an identical wire payload
  shape, and only the resolution condition differs (an interaction, a boundary-wall collision, occupancy met or not, or
  leaving an interaction zone without interacting).

External:
- sollertia-experiment subscribes to resolve the per-trial outcome and command the stimulus hardware.

---

## Lifecycle rules

`/gimbl-framework` owns the channel lifecycle invariants (construct in `Start()` rather than `Awake()`, use the
null-conditional `?.` in `OnDestroy()`, store typed channels as `MQTTChannel<TMessage>`, and give typed payloads public
fields). When adding a new topic, follow the pattern `LickStimulusSpawner.Start`, `LickStimulusSpawner.OnDestroy`, and
its `ReleaseChannel` helper establish:

```csharp
// Start()
_myListener = new MQTTChannel(MQTTTopics.MyTopic, isListener: true);
_myListener.receivedEvent.AddListener(OnMyEvent);

// OnDestroy()
_myListener?.receivedEvent.RemoveListener(OnMyEvent);
_myListener?.client?.Unsubscribe(_myListener);
```

Removing the listener is not enough on its own. The channel stays in `MQTTClient`'s routing list until it is released,
so a channel that outlives its owner keeps receiving every publish on its topic and a typed channel keeps deserializing
each payload. `MQTTClient.Unsubscribe` (`MQTTClient.cs`) is what removes the routing entry, and a component that
builds channels in `Start` MUST release them in `OnDestroy`.

The per-symptom diagnosis playbook below names which invariant a symptom violates and points back at `/gimbl-framework`
for the underlying class-API contract.

---

## Adding a new topic

You MUST work through this checklist before introducing a new MQTT topic:

```text
- [ ] Topic name is flat PascalCase with no slashes
- [ ] Constant is declared as `public const string` in Assets/Gimbl/Scripts/MQTT/MQTTTopics.cs
- [ ] Constant's C# identifier is identical to its literal value (`public const string Foo = "Foo";`)
- [ ] Constant's XML remarks include Direction, Payload, and Callers entries kept accurate
- [ ] If paired with a "Trigger" topic, both are named `<Name>Trigger` and `<Name>`
- [ ] Owning script(s) identified, and a topic is not split across unrelated MonoBehaviours
- [ ] Channel direction (subscriber vs publisher) matches the intent
- [ ] Start() constructs the channel and registers AddListener callbacks, and OnDestroy() removes the listeners
      with `?.` and releases each listener channel via `MQTTClient.Unsubscribe`
- [ ] Subscriber callbacks touch no Unity API, so they record the event (`Interlocked` / `Volatile` / lock) and act
      on it in `Update`, because broker delivery arrives on an MQTTnet worker thread
- [ ] Payload class (if typed) is a public class with public fields (JsonUtility constraint)
- [ ] `Assets/Tests/EditMode/MQTTTopicsTests.cs` is updated: bump `ExpectedTopicCount`, add the literal to
      `ExpectedTopics`, and add a per-topic `<Name>_Constant_EqualsTheContractLiteral` test. Otherwise every
      structural test in the fixture fails at once (see `/unity-tests`)
- [ ] `experiment:vr-driver-interface` is updated on its side (`_VRTaskMQTTTopics`), coordinating the change across
      both repos
- [ ] This skill's topic catalog is updated with the new entry
- [ ] Did NOT treat Editor / keyboard-only success as proof of wiring, because `MQTTClient.Publish` loops messages
      in-process when no broker is connected, so a Unity-only topic with no `sollertia-experiment` counterpart
      appears to work locally and drops in production
- [ ] Grepped the Console for `in-process subscribers only`, the stable fragment of the `Unable to deliver '<topic>'
      to the MQTT broker at <ip>:<port>` warning (`MQTTClient.Publish`) that `MQTTClient` logs once per topic on the
      first loopback publish, to enumerate every topic that never crossed the process boundary
- [ ] Confirmed the experiment-side publisher / subscriber landed in the same release
```

The test suite discovers topics reflectively. `MqttTestHarness.KnownTopics()` enumerates the literals on `MQTTTopics`
(`MqttTestHarness.cs`), so a new constant is captured by every zone and task fixture with no harness edit. Only
`MQTTTopicsTests.cs` names the catalog by hand, and it is the file that fails if the count, the literal set, or the
identifier-equals-literal rule is broken. See `/unity-tests` for how to run both platforms.

---

## Diagnosing a missed message

When a topic appears to work on one side but not the other, locate the matching symptom below and work through its
likely cause and first check.

### Unity subscribes but never receives

- **Likely cause**: Topic string mismatch between sides, hand-typed literal drifted from the constant, or casing differs
  (`Interaction` vs `interaction`, because `MQTTClient` uses `StringComparison.Ordinal`).
- **First check**: Confirm both sides reference `MQTTTopics.<Name>` (Unity) or the matching `_VRTaskMQTTTopics` member
  on the `sollertia-experiment` side with identical casing.
- **Second likely cause**: The channel was constructed while the broker was unreachable. `MQTTClient.Subscribe` adds the
  channel to the routing list unconditionally but returns before `SubscribeAsync` when `IsConnected()` is false
  (`MQTTClient.cs`) and never retries, so the channel receives in-process loopback traffic only and does **not**
  auto-subscribe once the broker comes online.
- **Second check**: Bring the broker up first, then re-enter Play Mode (or re-create the channel) to force a fresh
  subscribe pass.

### Unity publishes but experiment never receives

- **Likely cause**: Broker not connected (the publish reaches in-process subscribers only and never crosses to the
  experiment process), or experiment-side subscription not active.
- **First check**: The scene's connector calls `Connect(verbose: false)` (`MQTTConnectorObject.OnEnable`), so **no
  success line is printed during a run**. Look instead for `Unable to connect to the MQTT broker at <ip>:<port>`
  (`MQTTClient.Connect`), for the once-per-topic loopback warning `Unable to deliver '<topic>' to the MQTT broker at
  <ip>:<port>` (`MQTTClient.Publish`), whose stable fragment is `in-process subscribers only`, and for any `Unable to
  publish to the MQTT topic '<topic>'` line. Then confirm experiment subscribed to the matching `MQTTTopics.<Name>`.
  The `Successfully connected to MQTT Broker at: <ip>:<port>` line appears only when the Task Parameters window's Test
  Connection button is pressed (`MainWindow.DrawMQTTSection` passes `verbose: true`).

### Channel constructor throws `InvalidOperationException`

- **Likely cause**: A `MQTTChannel` was created before `MQTTClient.Instance` was set (e.g., in another script's
  `Awake`).
- **First check**: Move the channel construction into the consumer's `Start()`, because Unity guarantees every `Awake()`
  runs first.

### Typed message received but every field is default

- **Likely cause**: Payload class uses `{ get; set; }` instead of public fields.
- **First check**: Grep the payload class for `{ get; set; }` and convert to public fields.

### Typed channel `Send()` succeeds but subscriber callback never fires

- **Likely cause**: Reference stored as base `MQTTChannel` instead of `MQTTChannel<TMessage>`, so the typed
  `new`-shadowed `receivedEvent` is dropped.
- **First check**: Change the field declaration to `MQTTChannel<TMessage>` and re-register `AddListener` on the typed
  event.

### Subscriber callback stops firing after scene change

- **Likely cause**: `OnDestroy()` removed the listener, and the new scene's `Start()` did not re-register.
- **First check**: Inspect the scene's MonoBehaviour `Start()` for the missing `AddListener` call.

### Random message loss under high rate

- **Likely cause**: Experiment-side publisher running below QoS 2 (Unity publish is hardcoded `ExactlyOnce`, and Unity
  subscribe defaults to QoS 2).
- **First check**: Inspect the broker's retained messages and the experiment-side publisher's QoS setting.

### Unity receives its own `Stimulus` publication

- **Likely cause**: Expected, because `LickStimulusSpawner` is an intentional intra-Unity subscriber on `Stimulus`.
- **First check**: This is documented in [Multi-consumer topics](#multi-consumer-topics) rather than a defect.

### Keyboard-only test run still triggers UI lick / stimulus

- **Likely cause**: Expected, because `MQTTClient.Publish` loops messages in-process when the broker is unreachable.
- **First check**: The in-process loopback is the dev-without-broker path rather than a defect. The Console records it
  once per topic as a warning that begins `Unable to deliver '<topic>' to the MQTT broker at <ip>:<port>`
  (`MQTTClient.Publish`), so grep for its stable fragment `in-process subscribers only` to list every topic that
  stayed inside Unity.

### `RequireInteraction` / `RequireWait` writes have no effect

- **Likely cause**: Payload missing the `value` field, or the JSON wrapper is malformed.
- **First check**: Confirm payload is `{"value": true}` / `{"value": false}` (case-sensitive, JSON-serializable).

---

## Interaction contract

- **You MUST NOT** hardcode a topic string anywhere outside `MQTTTopics.cs`. You MUST always reference
  `MQTTTopics.<Name>` so a rename propagates automatically and skill / source drift is impossible.
- **You MUST NOT** rely on message ordering between two different topics, because MQTT provides per-topic ordering only.
- **You MUST NOT** add a publisher and a subscriber on the same topic inside Unity expecting deduplication. MQTTnet
  delivers the message to every subscriber, including the local one, and this is the intended behavior for `Stimulus`
  and the in-process loopback path.

---

## Related skills

| Skill                                            | Relationship                                                                                  |
|--------------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/gimbl-framework` (this plugin)                 | Owns `MQTTClient`, `MQTTChannel`, and `MQTTChannel<T>` class references                       |
| `/task-prefabs` (this plugin)                    | Generated prefabs wire the zones whose scripts own these topics                               |
| `/zone-prefabs`, `/task-generator` (this plugin) | Own the zone prefab and the placement branch whose scripts construct channels on these topics |
| `/task-parameters` (this plugin)                 | Editor-time alternative for `RequireInteraction` / `RequireWait` flags                        |
| `/scene-setup` (this plugin)                     | `UI-lick-reward` subsystem subscribes to `Interaction` and `Stimulus`                         |
| `/play-mode` (this plugin)                       | MQTT activity is only live while the Editor is in `playing` state                             |
| `/unity-tests` (this plugin)                     | `MQTTTopicsTests` pins this catalog, and `MqttTestHarness` needs no broker                    |
| `assets:task-templates`                          | YAML cue codes appear as `byte` values in `CueSequence` payloads                              |
| `experiment:vr-driver-interface`                 | Host (Python) side, where `_VRTaskMQTTTopics` mirrors this catalog, so change both together   |

---

## Verification checklist

```text
- [ ] The topic catalog above matches every `public const string` in MQTTTopics.cs
- [ ] Every new channel has an entry with direction, channel type, payload, and owning script
- [ ] Payload class shapes in this file match the C# definitions (public fields only)
- [ ] Cross-Unity subscriptions (multi-consumer topics) are documented when they exist
- [ ] Changes propagate to sollertia-experiment if a topic's direction, payload, or name changes
- [ ] No hand-typed topic literals appear outside MQTTTopics.cs (grep the project for the literal)
- [ ] Assets/Tests/EditMode/MQTTTopicsTests.cs pins the new topic (ExpectedTopicCount, ExpectedTopics,
      per-topic <Name>_Constant_EqualsTheContractLiteral)
- [ ] Both test platforms pass (see /unity-tests)
```
