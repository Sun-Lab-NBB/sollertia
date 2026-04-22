---
name: gimbl-framework
description: >-
  Reference for the GIMBL VR framework inlined under `Assets/Gimbl/` in sollertia-unity-tasks:
  ActorObject, ControllerObject (LinearTreadmill, SimulatedLinearTreadmill), DisplayObject
  rig, MQTTClient / MQTTChannel. Use when reading or modifying code that instantiates GIMBL
  components, wiring a new script to the MQTT broker, or diagnosing null-reference errors
  during scene init.
user-invocable: true
---

# GIMBL framework reference

Documents the GIMBL VR framework as **inlined** into `sollertia-unity-tasks` under `Assets/Gimbl/`. GIMBL is a
third-party VR task framework that the project depends on; the inlined copy has been refactored (logging removed,
unused MQTT topics deprecated) and is no longer tracked against the upstream repository.

---

## Scope

**Covers:**
- GIMBL package layout under `Assets/Gimbl/` and the `Gimbl` C# namespace
- `ActorObject` (animal representation, display + controller binding)
- Controller hierarchy (`ControllerObject` → `LinearTreadmill` → `SimulatedLinearTreadmill`)
- `DisplayObject` and the display rig structure
- `MQTTClient` singleton and `MQTTChannel` / `MQTTChannel<T>` contract
- The GIMBL `Window → Gimbl` editor menu that opens Settings, Actor, and Displays windows
- Decision rules for whether a given behavior belongs in GIMBL or in the `SL.*` task layer

**Does not cover:**
- Individual MQTT topic wiring in project scripts (see `/mqtt-contract`)
- Task prefab generation (see `/task-generator`, `/task-prefabs`)
- Display rig configuration workflows (see `/scene-setup`)
- Upstream GIMBL releases — the inlined copy is independently maintained

---

## Directory layout

```text
Assets/Gimbl/
├── Editor/                           ← Editor windows, prefab post-processing, inspector extensions
│   ├── ActorWindow.cs                ← Window → Gimbl opens this
│   ├── DisplaysWindow.cs             ← Window → Gimbl opens this
│   ├── MainWindow.cs                 ← Window → Gimbl (entry point)
│   └── ...
├── Plugins.meta                      ← Plugins folder metadata
├── Resources/                        ← Runtime-loaded assets (icons, default materials)
└── Scripts/
    ├── Actor/
    │   ├── ActorObject.cs            ← Animal GameObject facade
    │   └── ActorSettings.cs          ← ScriptableObject holding per-actor settings
    ├── Controllers/
    │   ├── ControllerObject.cs       ← Abstract base
    │   ├── ControllerOutput.cs       ← Output-side interface
    │   ├── ControllerTypes.cs        ← Enum / factory
    │   ├── LinearTreadmill.cs        ← MQTT-driven treadmill
    │   ├── SimulatedLinearTreadmill.cs ← Keyboard-driven treadmill for testing
    │   └── SimulatedInput.cs         ← Unity Input System action map
    ├── Displays/
    │   ├── DisplayObject.cs          ← Attach-to-actor display rig
    │   ├── Monitor.cs                ← Monitor entry in the display list
    │   ├── PerspectiveProjection.cs  ← Per-monitor projection matrix math
    │   └── FullScreenView.cs         ← Runtime fullscreen rendering path
    └── MQTT/
        ├── MQTTClient.cs             ← Broker connection singleton (MonoBehaviour)
        ├── MQTTChannel.cs            ← Untyped and typed channel classes
        └── MQTTConnectorObject.cs    ← Connector utility
```

---

## Core types

### `Gimbl.MQTTClient`

Singleton MonoBehaviour that owns the MQTTnet client. Attached to a `MQTT Client` GameObject in the scene.

- `MQTTClient.Instance` — static property; null outside of Play Mode until `Awake()` runs.
- Connection settings (`ipAddress`, `port`) are loaded from `EditorPrefs`, not serialized on the scene.
- Default settings: `127.0.0.1:1883` (Mosquitto broker running locally).
- `Subscribe(channel, topic, qosLevel)` and `Publish(topic, payload)` are called by `MQTTChannel`; do not call them
  directly.

**Lifecycle gotcha:** `MQTTChannel` constructors throw `InvalidOperationException` if `MQTTClient.Instance` is
null. This typically means the scene lacks a `MQTT Client` GameObject, or the script calling `new MQTTChannel(...)`
is running before `MQTTClient.Awake()`. Always create channels in `Start()` (not `Awake()`) — Unity guarantees every
`Awake()` runs before any `Start()`.

### `Gimbl.MQTTChannel`

Untyped trigger channel. Used for topics carrying no payload.

```csharp
var channel = new MQTTChannel("MyTopic/", isListener: true, qosLevel: 2);
channel.receivedEvent.AddListener(OnMyEvent);  // void OnMyEvent() { ... }

// Later, to publish:
channel.Send();

// On teardown:
channel.receivedEvent.RemoveListener(OnMyEvent);
```

Parameters:
- `topicString` — trailing slash required by project convention
- `isListener` — `true` to subscribe, `false` for publish-only
- `qosLevel` — defaults to 2 (exactly-once); do not downgrade without cross-team agreement

### `Gimbl.MQTTChannel<TMessage>`

Typed channel using `JsonUtility` for serialization. The payload type must have plain public fields — `JsonUtility`
does not read properties.

```csharp
public class MyMessage
{
    public float value;
    public string note;
}

var channel = new MQTTChannel<MyMessage>("MyTypedTopic/", isListener: true);
channel.receivedEvent.AddListener((MyMessage msg) => { /* ... */ });
channel.Send(new MyMessage { value = 1.0f, note = "hi" });
```

Serialization bytes: `Encoding.UTF8.GetBytes(JsonUtility.ToJson(message))`.

### `Gimbl.ActorObject`

`MonoBehaviour` that represents the animal in the VR environment. Holds references to a `DisplayObject` and a
`ControllerOutput`, and exposes `isActive` to gate movement.

```csharp
public class ActorObject : MonoBehaviour
{
    public bool isActive;
    public ActorSettings settings;
    public DisplayObject Display { get; set; }      // auto-parents display on set
    public ControllerOutput Controller { get; set; } // auto-registers on set
}
```

Setting `Display = someDisplay` calls `DisplayObject.ParentToActor(this)`, which reparents the display GameObject
under the actor and configures camera culling layers. Assigning `null` unparents.

The task layer (`SL.Tasks.Task`) holds a reference to the actor via the public `actor` field. `Task.OnValidate()`
auto-assigns the first `ActorObject` in the scene if the field is null — the user still needs to confirm the
assignment in the Inspector before Play Mode.

### `Gimbl.ControllerObject` and subclasses

Abstract base for input devices that drive actor movement. The only runtime subclasses used in this project are:

| Class                      | Driven by                         | Purpose                                           |
|----------------------------|-----------------------------------|---------------------------------------------------|
| `LinearTreadmill`          | MQTT `<deviceName>/Data/` topic   | Production treadmill input from hardware          |
| `SimulatedLinearTreadmill` | Unity Input System (keyboard)     | Developer testing without hardware                |

`SimulatedLinearTreadmill` extends `LinearTreadmill` and replaces MQTT input with keyboard movement. It also
publishes `LickPort/` on left-click to simulate a lick event — see `/mqtt-contract` for the cross-listener
implications.

### `Gimbl.DisplayObject`

Represents a VR display attached to an actor. Manages:
- Parenting under the actor and Y-offset from `settings.heightInVR`
- Camera culling to hide the actor's own layer from its own camera
- Brightness override via `currentBrightness` (0–100)

Multi-monitor rigs (three displays for the mesoscope setup) use one `DisplayObject` per monitor, each with its own
child Camera. `FullScreenViewManager` creates the fullscreen views at runtime.

---

## GIMBL editor windows

`MainWindow` registers the `Window → Gimbl` menu entry. Selecting it opens three editor windows:

| Window            | Purpose                                                  |
|-------------------|----------------------------------------------------------|
| Settings          | MQTT broker IP/port, session configuration, setup import |
| Actor             | Create / edit `ActorObject` instances in the scene       |
| Displays          | Configure `DisplayObject` rig (monitors, projection)     |

The Settings panel persists MQTT settings to Unity `EditorPrefs` — they are **not** stored per-scene. A fresh
checkout inherits the settings of the last project the Editor opened.

---

## GIMBL vs `SL.*` responsibility split

Use this table to decide which layer a change belongs in. GIMBL is refactored but still shared framework code —
changes there affect every project that adopts the inlined copy.

| Concern                                       | Owner         | Example files                               |
|-----------------------------------------------|---------------|---------------------------------------------|
| Animal representation and display binding     | GIMBL         | `ActorObject.cs`, `DisplayObject.cs`        |
| MQTT transport and serialization primitives   | GIMBL         | `MQTTClient.cs`, `MQTTChannel.cs`           |
| Hardware treadmill input                      | GIMBL         | `LinearTreadmill.cs`                        |
| Simulated treadmill input                     | GIMBL         | `SimulatedLinearTreadmill.cs`               |
| Display rig projection math                   | GIMBL         | `PerspectiveProjection.cs`, `Monitor.cs`    |
| Corridor teleportation and cue sequence       | `SL.Tasks`    | `Task.cs`                                   |
| Stimulus trigger zones and behavior modes     | `SL.Tasks`    | `StimulusTriggerZone.cs`, `OccupancyZone.cs`|
| YAML template schema and loader               | `SL.Config`   | `ConfigLoader.cs`, `TaskTemplate.cs`        |
| Editor-side prefab generation                 | `SL.Tasks`    | `CreateTask.cs` (Editor)                    |
| Editor MCP bridge                             | `SL.Tasks`    | `McpBridge.cs` (Editor)                     |
| UI feedback for lick/stimulus events          | `SL.UI`       | `LickStimulusSpawner.cs`                    |

Rule of thumb: a change that would be useful to any GIMBL-based project belongs in `Assets/Gimbl/`. A change
specific to the infinite corridor task, the cue / segment / trial vocabulary, or the sollertia-experiment MQTT
contract belongs in `Assets/InfiniteCorridorTask/` or `Assets/UI-lick-reward/`.

---

## Do-not-modify zones

Avoid editing these GIMBL files unless absolutely required. They are either load-bearing across the framework or
easy to break in non-obvious ways:

| File                                    | Reason                                                         |
|-----------------------------------------|----------------------------------------------------------------|
| `MQTT/MQTTClient.cs`                    | Singleton lifecycle; breakage leaves every channel dangling    |
| `Displays/PerspectiveProjection.cs`     | Projection matrix math is calibrated for the mesoscope rig     |
| `Displays/Monitor.cs`                   | Editor + runtime coordination for fullscreen views             |
| `Controllers/ControllerObject.cs`       | Abstract base used by polymorphic editor inspectors            |

If a change here is unavoidable, update the `/csharp-style` verification pass and test against the full display rig
(three monitors) before committing.

---

## Common pitfalls

| Pitfall                                                   | Fix                                                            |
|-----------------------------------------------------------|----------------------------------------------------------------|
| `NullReferenceException` on `MQTTClient.Instance`          | Ensure the scene has a `MQTT Client` GameObject with `MQTTClient` attached; create channels in `Start()` not `Awake()` |
| `JsonUtility` returning default-valued messages            | Payload class uses properties `{ get; set; }` — convert to public fields |
| Display rig attaches but actor sees through walls          | Camera culling layer missing; check that the actor's layer matches the scene hierarchy |
| `Window → Gimbl` opens only one panel                      | The other panels are hidden behind the Inspector dock; detach the Inspector to reveal |
| Settings reset after reopening the project                 | `EditorPrefs` are per-user-per-project-path; use project-scoped settings file if needed |
| `SimulatedLinearTreadmill` publishes duplicate licks       | Left-click publishes one `LickPort/` per frame; holding the button sends continuous events |

---

## Verification checklist

```text
- [ ] New scripts that use MQTT create channels in Start(), not Awake() or constructors
- [ ] Typed channels use payload classes with public fields (no properties)
- [ ] Null-conditional operator (?.) used when removing listeners in OnDestroy()
- [ ] Display / Controller assignments use the ActorObject property setters, not direct field access
- [ ] Changes to MQTTClient, PerspectiveProjection, or Monitor have been tested against the full multi-monitor rig
- [ ] `/mqtt-contract` is updated if any new channel is introduced via GIMBL primitives
```

---

## Related skills

| Skill                               | Relationship                                                       |
|-------------------------------------|--------------------------------------------------------------------|
| `/mqtt-contract` (this plugin)      | Topic catalog for every channel constructed on top of `MQTTChannel`|
| `/scene-setup` (this plugin)        | Display rig configuration consumes `DisplayObject` APIs            |
| `/task-generator` (this plugin)     | Segment prefabs sit inside the actor's coordinate frame            |
| `/csharp-style` (automation plugin) | GIMBL code is held to the same C# conventions as the project       |
