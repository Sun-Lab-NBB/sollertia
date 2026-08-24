---
name: gimbl-framework
description: >-
  Reference for the GIMBL VR framework inlined under `Assets/Gimbl/` in sollertia-virtual-reality:
  ActorObject, ControllerObject hierarchy, DisplayObject rig, FullScreenViewManager,
  MQTTClient / MQTTChannel / MQTTTopics, and the MainWindow Task Parameters editor. Use when
  reading or modifying code that instantiates GIMBL components, wiring a new script to the MQTT
  broker, or diagnosing null-reference errors during scene init.
user-invocable: false
---

# GIMBL framework reference

The GIMBL VR framework is inlined into this codebase under `Assets/Gimbl/`. This skill is the only source of truth for
GIMBL in `sollertia-virtual-reality`. You MUST disregard any GIMBL knowledge (file names, class shapes, editor windows,
MQTT wiring, conventions) that does not come from this skill or from the code under `Assets/Gimbl/` itself.

**Reference-only skill.** No upstream. Agents arrive here on demand from `/scene-setup` (auto-created scene
infrastructure), `/task-parameters` (Actor / Display / MQTT class semantics), `/mqtt-contract` (channel API +
lifecycle), `/task-generator` (segment prefabs and the Actor coordinate frame), and `/zone-prefabs` (modifier-script
invariants).

---

## Scope

**Covers:**
- GIMBL package layout under `Assets/Gimbl/` and the `Gimbl` C# namespace (current file inventory), including the two
  assemblies its `.asmdef` files declare
- `MainWindow` Task Parameters editor window (the only GIMBL editor window with a menu entry)
- `ActorObject` (animal representation, display + controller binding, model swap)
- `MQTTClient` singleton, `MQTTChannel` / `MQTTChannel<T>` typed messaging, `MQTTTopics` constants
- Controller hierarchy: `ControllerObject` → `LinearTreadmill` → `SimulatedLinearTreadmill`, with `ControllerOutput`
  indirection and the `ControllerTypes` enum
- `DisplayObject`, `PerspectiveProjection`, `Monitor`, and the `FullScreenViewManager` rig
- Which EditMode / PlayMode fixture covers each GIMBL type
- Decision rules for whether a given behavior belongs in GIMBL or the `SL.*` task layer

**Does not cover:**
- Individual MQTT topic wiring and the topic catalog (see `/mqtt-contract`)
- Programmatic Task Parameters read / write (see `/task-parameters`)
- Editor scene configuration workflows (see `/scene-setup`)
- Task prefab generation (see `/task-generator`, `/task-prefabs`)
- Test-authoring conventions and the project-wide assembly catalog (see `/unity-tests`)

---

## Directory layout

```text
Assets/Gimbl/
├── Editor/
│   ├── MainWindow.cs                   ← `Window → Task Parameters` — the only menu-entry window
│   └── Sollertia.Gimbl.Editor.asmdef   ← Editor-only assembly (see below)
├── Resources/
│   ├── Actors/
│   │   ├── Prefabs/
│   │   │   └── Rodent.prefab           ← Actor model prefab; loaded by `Resources.Load("Actors/Prefabs/<name>")`
│   │   └── Rodent/                     ← Source mesh, texture, and material for `Rodent.prefab`
│   ├── Displays/
│   │   └── MyMonitorSetup.prefab       ← Display model prefab; loaded by `Resources.Load("Displays/<name>")`
│   ├── Icons/                          ← Unreferenced legacy icon art (`mouse.png` / `mouse.psd`)
│   └── Materials/                      ← Unreferenced `MonitorTestPattern.psd` texture
└── Scripts/
    ├── LayoutSettings.cs               ← Shared GUI styling helpers used by MainWindow
    ├── Sollertia.Gimbl.asmdef          ← Runtime assembly (see below)
    ├── TagsAndLayers.cs                ← Tag / layer creation helpers (e.g., VRDisplay, TrackCam)
    ├── Actor/
    │   └── ActorObject.cs              ← Animal GameObject facade (Display, Controller, Model)
    ├── Controllers/
    │   ├── ControllerObject.cs         ← Abstract base + ValueBuffer movement accumulator
    │   ├── ControllerOutput.cs         ← Type-erased reference held by ActorObject.Controller
    │   ├── ControllerTypes.cs          ← Enum of supported controller subclasses
    │   ├── LinearTreadmill.cs          ← MQTT-driven (`Motion`) hardware treadmill
    │   ├── SimulatedInput.cs           ← Auto-generated Unity Input System binding class
    │   ├── SimulatedInput.inputactions ← Input System action map source for `SimulatedInput.cs`
    │   └── SimulatedLinearTreadmill.cs ← Keyboard-driven; hides Start() per non-chaining contract
    ├── Displays/
    │   ├── BrightnessShader.shader     ← `Hidden/BrightnessShader`; used by PerspectiveProjection
    │   ├── DisplayObject.cs            ← Attach-to-actor display rig with per-monitor cameras
    │   ├── DisplaySettings.cs          ← ScriptableObject under Assets/VRSettings/Displays/
    │   ├── Monitor.cs                  ← OS monitor enumeration (Win/Linux/macOS) + DPI probe + entry struct
    │   ├── PerspectiveProjection.cs    ← Per-monitor projection matrix math (calibrated)
    │   ├── FullScreenView.cs           ← Borderless `EditorWindow` popup (Editor-only) per monitor
    │   ├── FullScreenViewManager.cs    ← Editor-side camera ↔ monitor binding + save asset
    │   └── FullScreenViewsSaved.cs     ← ScriptableObject persisting per-scene camera bindings
    └── MQTT/
        ├── MQTTClient.cs               ← MonoBehaviour broker connection singleton (MQTTnet)
        ├── MQTTChannel.cs              ← Untyped and typed (`MQTTChannel<T>`) channel classes
        ├── MQTTTopics.cs               ← `public const string` catalog of every topic
        └── MQTTConnectorObject.cs      ← MonoBehaviour that calls `MQTTClient.Connect()` on `OnEnable`
```

Actor and display model prefabs live in `Assets/Gimbl/Resources/Actors/Prefabs/` and `Assets/Gimbl/Resources/Displays/`.
`ActorObject.SetModel` and `DisplayObject.Create` load them via `Resources.Load($"Actors/Prefabs/<name>")` and
`Resources.Load($"Displays/<name>")`. `MainWindow` enumerates the same folders via `Resources.LoadAll<GameObject>` to
populate the model dropdowns. Those two folders are the only `Resources/` subtrees any code reads. `Icons/` and
`Materials/` hold legacy art no script or asset references.

**Assemblies.** GIMBL compiles into two of the project's eight named assemblies: the runtime `Sollertia.Gimbl`
(rootNamespace `Gimbl`, references `Unity.InputSystem`), and the Editor-platform-only `Sollertia.Gimbl.Editor`
(references `Sollertia.Gimbl` and `Sollertia.InfiniteCorridorTask`, which is why `MainWindow` may `using SL.Tasks;`
while no runtime GIMBL script may). A new file joins an assembly by sitting inside its subtree. A brand-new folder needs
its own `.asmdef` plus a `references` entry from every consuming assembly, including the test assemblies, or the tests
cannot see it. `/unity-tests` owns the full catalog.

---

## Core types

### `Gimbl.MainWindow`

`EditorWindow` registered under `Window → Task Parameters`, the only GIMBL `EditorWindow` with a menu entry
(`FullScreenView` and `Monitor`'s private `MonitorTester` are the other two, both created programmatically). The single
GUI entry point for every per-scene configuration field (Actor, MQTT, Display, Camera Mapping, Task). See `/scene-setup`
for the GUI workflow and `/task-parameters` for the programmatic surface.

Notable invariants:

- `[InitializeOnLoadMethod] RegisterAutoOpen` subscribes to `EditorSceneManager.sceneOpened`,
  `EditorApplication.playModeStateChanged`, and `EditorApplication.delayCall` so the window reappears after Editor
  start, scene open, or Play Mode enter. Closing it manually is recoverable, because the next hook that fires re-opens
  it. In batch mode (`Application.isBatchMode`) none of the three hooks is registered, so CI and headless test runs
  never auto-open it. Opening the window there would run `InitializeScene` against the throwaway startup scene and block
  the run on a save dialog batch mode cancels.
- `OnEnable() → InitializeScene()` first creates the `Assets/VRSettings` and `Assets/VRSettings/Displays` folders when
  missing (`DisplayObject.Create` and `FullScreenViewManager.LoadCameras` write their assets there). It then ensures the
  active scene contains `Actors`, `Controllers`, `MQTT Client` roots, removes the Unity-default Main Camera, creates a
  default Actor + Display, runs `EnsureControllers`, and finally runs `EnsureMqttDefaults` to apply the project-wide
  broker IP / port. Existing GameObjects keep their components, but every pass re-stamps `hideFlags`: `MQTT Client` is
  forced to `HideFlags.HideInHierarchy` and `Controllers` to `HideFlags.None`, whether the object was just created or
  already present. So the MQTT client does **not** appear in the scene Hierarchy panel. Looking for it there is futile,
  so query it via `GameObject.Find("MQTT Client")` or `MQTTClient.Instance`.
- `EnsureControllers()` iterates `CachedControllerSpecs` (resolved once via reflection at type init) and creates one
  GameObject per `ControllerTypes` enum value. The actor's controller assignment is not auto-changed, so user-chosen
  swaps survive a re-init.
- `EnsureMqttDefaults()` (public static) finds the scene's `MQTT Client` GameObject and applies the broker IP / port
  from `EditorPrefs` with a `127.0.0.1:1883` fallback. Used by `InitializeScene` and by
  `CreateTask.CreateSceneFromTemplate` so a fresh scene reports the default broker to MCP reads synchronously, without
  waiting for the next `delayCall` tick.
- `SyncDisplayBrightnessToSettings()` (public static) sets the active scene's `DisplayObject.currentBrightness` to its
  referenced `DisplaySettings.brightness` value. Called by `CreateTask.CreateSceneFromTemplate` only, and deliberately
  not called from `InitializeScene`, so per-scene customizations to `currentBrightness` survive subsequent scene-open
  passes.
- The window is `GetWindow<MainWindow>("Parameters", ..., new Type[] {InspectorWindow})` so it docks next to the
  Inspector by default. The docked tab label is the short `Parameters` and the menu entry is the long `Task Parameters`
  to disambiguate.
- **Scene-component cache.** `MainWindow` caches `_client` (the scene's `MQTTClient`), `_cachedTask`, `_cachedActor`,
  `_cachedDisplay`, `_cachedGuidanceZone`, and `_cachedOccupancyZone` references on first access and invalidates them on
  `EditorSceneManager.activeSceneChangedInEditMode`. Without the cache, every `OnGUI` repaint (potentially many times
  per second) would run six `FindAnyObjectByType` walks of the scene. `InvalidateSceneCache` is a private instance
  method called only from `OnEnable` and from the scene-change handler, so no external code path can trigger it. A code
  path that swaps a Task / Actor / Display / zone component in the active scene without an active-scene change keeps
  reading the stale reference until the window is reopened. The cached fields do self-heal, because each `GetCachedX`
  re-resolves whenever its field compares equal to null, which includes a destroyed Unity object.

### `Gimbl.MQTTClient`

Singleton `MonoBehaviour` that owns the MQTTnet client. Attached to the auto-created `MQTT Client` GameObject in every
scene initialized by `MainWindow`.

- `MQTTClient.Instance` is a static property, null until the GameObject's `Awake()` runs.
- Connection settings (`ipAddress`, `port`) have C# field initializers of `"127.0.0.1"` and `1883`, so a
  freshly-`AddComponent`'d client reports the loopback default even before any hook runs. `Awake()` overlays the same
  values from `EditorPrefs` (`SollertiaVR_MQTT_IP` / `SollertiaVR_MQTT_Port`) with the loopback fallback, and
  `MainWindow.EnsureMqttDefaults` applies the same logic synchronously at scene init and at scene creation. The three
  paths agree for an unset or in-range preference, but their port fallbacks differ. `Awake` rewrites any port outside
  1-65535 to 1883, while `EnsureMqttDefaults` falls back only when the stored port is exactly `0`. A stored out-of-range
  non-zero port therefore survives the editor pass and is corrected at `Awake`.
- `Subscribe(channel, topic, qosLevel)` and `Publish(topic, payload)` are called by `MQTTChannel`, so do not call them
  directly. `Unsubscribe(channel)` is the exception, because callers ARE expected to invoke it. It removes the channel
  from the routing list so a destroyed listener stops receiving messages, and deliberately leaves the broker-side topic
  filter in place, because several channels may share one topic while the filter is per topic. A component that builds
  channels in `Start()` releases them in `OnDestroy()`.
- **`Subscribe` late-connect failure mode**: the channel is added to the internal `_channelList` unconditionally so the
  in-process loopback (below) can route messages, but the broker-side `SubscribeAsync` only fires when the client is
  currently connected. A channel created while the broker is offline will receive in-process loopback messages but will
  **not** auto-subscribe to the broker when it later comes online. Callers that need broker-delivered messages after a
  late broker connect must re-create the channel (or trigger a fresh subscribe pass).
- **MQTT 5.0 only**: `MqttClientOptionsBuilder().WithProtocolVersion(MqttProtocolVersion.V500)`. Brokers must accept
  MQTT 5.0 (Mosquitto 2.0+).
- **Connect has a 10000ms timeout**: `Connect()` runs `ConnectAsync` on a `Task.Run`, waits up to 10 seconds
  (`ConnectTimeoutMilliseconds`), and logs an error if the connection has not completed. The method returns either way.
  `MQTTClient` continues running in an unconnected state and `Publish` automatically falls back to the in-process
  loopback. A broker that actively refuses the connection reports through a `Could not connect to MQTT broker at
  <ip>:<port>` Debug.LogError that a refused connection extends with `: <inner exception message>`, not via any thrown
  exception. The matching `Successfully connected to MQTT Broker at: <ip>:<port>` log fires only when `Connect(verbose:
  true)` is passed, so the default wiring is silent on success.
- **Raw `IMqttClient` is publicly accessible** as `MQTTClient.Instance.client`. External code can bypass the
  `MQTTChannel` abstraction and call `PublishAsync` / `SubscribeAsync` directly. This is a deliberate escape hatch for
  advanced cases, and production code should still go through `MQTTChannel` so the in-process loopback and channel-list
  bookkeeping stay consistent.
- **In-process loopback**: when the broker is unreachable, `Publish` routes the payload directly to in-process
  subscribers on the matching topic so keyboard-only test runs without a broker still reach local listeners (for
  example, `LickStimulusSpawner`). The first loopback delivery on each topic logs `MQTTClient: broker unreachable, so
  '<topic>' is delivered to in-process subscribers only ...`, deduplicated per topic via the `_loopbackWarnedTopics`
  set, so a console full of these warnings names exactly which topics have no wired experiment-side counterpart.
  Loopback calls `ReceivedMessage` synchronously on the publisher's thread, so a typed channel's deserialization
  `InvalidOperationException` propagates back out of the caller's `Send`, not only out of the broker callback.
- **Lifecycle**: `Awake` first guards against a duplicate. A second `MQTTClient` in the scene logs `MQTTClient: Multiple
  instances found, using existing instance` and returns without claiming `Instance` or loading any settings, so its
  `ipAddress` / `port` keep their field initializers. Otherwise `Awake` sets `Instance` and loads `ipAddress` / `port`
  from `EditorPrefs` (with loopback fallback). `Connect()` is invoked externally by `MQTTConnectorObject.OnEnable()`,
  which runs after every `Awake` and before any subscriber `Start()` constructs channels (see
  `Gimbl.MQTTConnectorObject` below). `Start` opens `SessionStart` / `SessionStop` channels and fires `SessionStart`
  after a 1-second delay (gives downstream subscribers time to attach). `OnApplicationQuit` publishes `SessionStop`,
  unsubscribes every channel, and disposes the client. `OnDestroy` duplicates the dispose path so scene transitions that
  bypass quit still release the `IMqttClient`. Both null `Instance` when it still points at the departing component, so
  `MQTTChannel` construction after teardown throws again.
- **`Connect()` is re-entrant**: it unhooks `_messageReceivedHandler` from, and disposes, any `IMqttClient` an earlier
  call installed before building the replacement, so a `MQTTConnectorObject` that re-enables repeatedly releases one
  broker handle per disable instead of accumulating them. The reset does not touch the routing list, so existing
  `MQTTChannel` instances keep receiving in-process loopback deliveries. Their broker-side subscriptions died with the
  disposed client and are never re-issued, because `Subscribe` runs only from the channel constructor. That is the
  second reason a channel created before a reconnect stops receiving broker-delivered messages.

**Lifecycle gotcha:** `MQTTChannel` constructors throw `InvalidOperationException` if `MQTTClient.Instance` is null.
This typically means the scene lacks a `MQTT Client` GameObject (or the script calling `new MQTTChannel(...)` is running
in `Awake()`). Always create channels in `Start()`, because Unity guarantees every `Awake()` runs before any `Start()`.

### `Gimbl.MQTTChannel`

Untyped trigger channel for topics carrying no payload.

```csharp
var channel = new MQTTChannel(MQTTTopics.MyTopic, isListener: true, qosLevel: 2);
channel.receivedEvent.AddListener(OnMyEvent);  // void OnMyEvent() { ... }

// Later, to publish:
channel.Send();

// On teardown (in OnDestroy):
channel.receivedEvent.RemoveListener(OnMyEvent);
channel.client?.Unsubscribe(channel);
```

Parameters:
- `topicString` is a flat PascalCase constant from `MQTTTopics` (no trailing slash).
- `isListener` is `true` to subscribe and `false` for publish-only.
- `qosLevel` defaults to 2 (`ExactlyOnce`), and you should not downgrade it without cross-team agreement. Note:
  `qosLevel` controls **subscription** QoS only. `MQTTClient.Publish` is hardcoded to
  `MqttQualityOfServiceLevel.ExactlyOnce` regardless of the channel's `qosLevel`, so a lower `qosLevel` value affects
  only how the broker delivers inbound messages to this client.

### `Gimbl.MQTTChannel<TMessage>`

Typed channel using `JsonUtility` for serialization. The payload type must have plain public fields, because
`JsonUtility` does not read properties.

```csharp
public class MyMessage
{
    public float value;
    public string note;
}

var channel = new MQTTChannel<MyMessage>(MQTTTopics.MyTypedTopic, isListener: true);
channel.receivedEvent.AddListener((MyMessage msg) => { /* ... */ });
channel.Send(new MyMessage { value = 1.0f, note = "hi" });
```

**The parameterless `Send()` is hidden and throws.** `MQTTChannel<TMessage>` re-declares `Send()` with the `new`
modifier and throws `NotSupportedException`, so a typed channel must publish through `Send(TMessage)`. The hide converts
an empty payload arriving at typed listeners as a null message into a failure at the call site. Because the hide is
`new` rather than `override`, calling `Send()` through a base `MQTTChannel` reference still publishes an empty payload,
which is another reason to hold typed channels as `MQTTChannel<TMessage>`.

Serialization bytes: `Encoding.UTF8.GetBytes(JsonUtility.ToJson(message))`. The typed `receivedEvent` shadows the base
class's parameterless event with the `new` modifier, so accessing the channel through a base-class reference exposes
only the trigger event and silently misses the typed callback. See the XML remarks in `MQTTChannel.cs` for the
rationale.

**Malformed JSON throws.** `MQTTChannel<TMessage>.ReceivedMessage` wraps any `JsonUtility.FromJson` failure in an
`InvalidOperationException` whose message includes the inner exception text. The exception propagates out of the
broker's message-received callback, which logs it but does not stop the client. This is distinct from the silent-failure
mode where `JsonUtility` returns a default-valued message (covered in Common pitfalls below). That mode fires when the
payload class uses properties instead of public fields and the JSON parses successfully but produces zeroed values.

### `Gimbl.MQTTTopics`

`public static class` holding every topic literal as `public const string`. Each entry has XML remarks declaring
`Direction`, `Payload`, and `Callers`. The single source of truth for topic names. See `/mqtt-contract` for the
audit-ready catalog. Never hardcode a topic string elsewhere.

### `Gimbl.MQTTConnectorObject`

Small `MonoBehaviour` whose only job is to call `MQTTClient.Instance.Connect(verbose: false)` from `OnEnable()`.
Attached to the `Logger` GameObject in `Assets/Scenes/ExperimentTemplate.unity`, so every scene the `CreateTask`
pipeline copies from the template inherits one instance. Decoupling the connect call from `MQTTClient.Awake()` is
load-bearing. Unity runs every `OnEnable` after every `Awake` but before every `Start`, so the broker connection is
initiated only after `MQTTClient.Awake()` has populated `ipAddress` / `port` from `EditorPrefs`, and finishes before any
subscriber `Start()` constructs `MQTTChannel` instances. Do not move the `Connect()` call back into
`MQTTClient.Awake()`, because the order would not be guaranteed across all scripts in the scene.

### `Gimbl.ActorObject`

`MonoBehaviour` that represents the animal in the VR environment. Holds references to a `DisplayObject` and a
`ControllerOutput`.

```csharp
public class ActorObject : MonoBehaviour
{
    [SerializeField] private DisplayObject _display;
    [SerializeField] private ControllerOutput _controller;

    public DisplayObject Display          // setter parents the new display, unparents the old
    {
        get => _display;
        set { /* see ActorObject.cs */ }
    }
    public ControllerOutput Controller    // setter unwires previous master.actor, wires new
    {
        get => _controller;
        set { /* see ActorObject.cs */ }
    }
}
```

- Setting `Display = someDisplay` calls `DisplayObject.ParentToActor(this)`, which reparents the display GameObject
  under the actor and configures camera culling so the actor's own layer is hidden from its own camera. Assigning `null`
  unparents.
- Setting `Controller = output` unhooks the previous controller's `actor` back-reference and wires the new one, marking
  the scene dirty (Editor only).
- `InitiateActor(modelName, trackCamera)` is the Editor-only bootstrap that parents under the `Actors` GameObject, adds
  a `CharacterController`, allocates a render layer, calls `SetModel`, and optionally creates a third-person tracking
  camera. The render-layer allocation goes through `TagsAndLayers.AddLayer(gameObject.name)`, which searches layer slots
  **starting at index 8** (Unity reserves 0-7 for built-in / default user layers). The tracking camera, when requested,
  scans every existing `TrackCam`-tagged camera in the scene, picks the lowest-numbered display index in `{0..7}` that
  no other tracking camera is using, and defaults to 7 if all are taken. Multi-actor scenes coordinate display
  assignments through this scan.
- `SetModel(modelName)` swaps the actor's `Model <name>` child GameObject by instantiating
  `Resources.Load("Actors/Prefabs/<modelName>")`. `"None"` removes the existing model without adding a new one.
- `EditMenu()` renders the Actor section of `MainWindow` (model + controller dropdowns).

The task layer (`SL.Tasks.Task`) holds a reference to the actor via the public `actor` field, but the Inspector hides it
(`[HideInInspector]`) and `Task.OnValidate()` auto-assigns it from the scene. The Task Parameters window's Task section
also re-resolves the actor when the field is null on each render.

### Subsystem references

- [references/controllers-subsystem.md](references/controllers-subsystem.md) covers `ControllerObject` /
  `ControllerOutput`, both treadmill subclasses, the cross-thread `movement` accumulator, and the `ControllerTypes`
  extension recipe.
- [references/displays-subsystem.md](references/displays-subsystem.md) covers `DisplayObject`, `PerspectiveProjection`,
  `Monitor`, `FullScreenView`, `FullScreenViewManager`, and `FullScreenViewsSaved`: the multi-monitor camera rig, its
  off-axis projection math, OS monitor enumeration, the shared `RefreshMonitorPositions` entry point, and the
  saved-views asset.

### Test coverage

The controller hierarchy shares one `ControllerTests` fixture. Every other GIMBL type except `LayoutSettings` and
`MQTTConnectorObject` has its own `<Type>Tests` EditMode fixture under `Assets/Tests/EditMode/` (`namespace
SL.Tests.EditMode`), and `MQTTConnectorObject` plus the runtime broker path are covered by `MqttClientPlayModeTests`
under `Assets/Tests/PlayMode/`. `Sollertia.Tests.EditMode` references both GIMBL assemblies, so a change to any GIMBL
primitive extends the matching fixture, adding one when the type has none. See `/unity-tests`.

---

## GIMBL editor window

`MainWindow` registers the only menu entry: `Window → Task Parameters`. Selecting it opens (or focuses) the window. The
five sections are rendered in fixed order by `OnGUI`:

1. **Actor**, with model and controller dropdowns
2. **MQTT**, with broker IP / port and Test Connection (greyed in Play Mode)
3. **Display**, with the Blank / Show toggle, `brightness`, and `heightInVR`
4. **Camera Mapping**, with Refresh Monitor Positions (which shares `FullScreenViewManager.RefreshMonitorPositions` with
   the `refresh_monitors` MCP tool), per-monitor camera dropdowns, and Show Full-Screen Views (button greyed in Play
   Mode)
5. **Task**, with `Require Interaction` / `Require Wait` (conditional on zone presence) and `Track Length` / `Track
   Seed`. The entire Task section's controls are greyed in Play Mode.

See `/scene-setup` for the user-facing workflow and `/task-parameters` for the programmatic mirror.

---

## GIMBL vs `SL.*` responsibility split

Use this table to decide which layer a change belongs in.

| Concern                                     | Owner       | Example files                                      |
|---------------------------------------------|-------------|----------------------------------------------------|
| Animal representation and display binding   | GIMBL       | `ActorObject.cs`, `DisplayObject.cs`               |
| MQTT transport and serialization primitives | GIMBL       | `MQTTClient.cs`, `MQTTChannel.cs`, `MQTTTopics.cs` |
| Hardware treadmill input                    | GIMBL       | `LinearTreadmill.cs`                               |
| Simulated treadmill input                   | GIMBL       | `SimulatedLinearTreadmill.cs`, `SimulatedInput.cs` |
| Display rig projection math                 | GIMBL       | `PerspectiveProjection.cs`                         |
| Editor-only OS monitor enumeration          | GIMBL       | `Monitor.cs`, `FullScreenViewManager.cs`           |
| Task Parameters editor window               | GIMBL       | `MainWindow.cs`, `LayoutSettings.cs`               |
| Corridor teleportation and cue sequence     | `SL.Tasks`  | `Task.cs`                                          |
| Stimulus trigger zones and behavior modes   | `SL.Tasks`  | `StimulusTriggerZone.cs`, `OccupancyZone.cs`       |
| YAML template schema and loader             | `SL.Config` | `ConfigLoader.cs`, `TaskTemplate.cs`               |
| Editor-side prefab generation               | `SL.Tasks`  | `CreateTask.cs` (Editor)                           |
| Editor MCP bridge                           | `SL.Tasks`  | `McpBridge.cs` (Editor)                            |
| UI feedback for lick / stimulus events      | `SL.UI`     | `LickStimulusSpawner.cs`                           |

Rule of thumb: a change to actor / display / controller / MQTT primitives or the Task Parameters editor belongs under
`Assets/Gimbl/`. A change specific to the infinite corridor task, the cue / segment / trial vocabulary, or the
`sollertia-experiment` MQTT contract belongs under `Assets/InfiniteCorridorTask/` or `Assets/UI-lick-reward/`.

---

## Do-not-modify zones

Avoid editing these GIMBL files unless absolutely required. They are either load-bearing across the framework or easy to
break in non-obvious ways:

| File                                | Reason                                                                                 |
|-------------------------------------|----------------------------------------------------------------------------------------|
| `MQTT/MQTTClient.cs`                | Singleton lifecycle and in-process loopback, so breakage leaves every channel dangling |
| `Displays/PerspectiveProjection.cs` | Off-axis matrix math is calibrated against each screen's physical placement            |
| `Displays/Monitor.cs`               | Editor-only (`#if UNITY_EDITOR`) OS enumeration, whose shape `refresh_monitors` parses |
| `Controllers/ControllerObject.cs`   | Reflection target of `MainWindow.BuildControllerSpecs`, and owns the `movement` buffer |
| `Controllers/LinearTreadmill.cs`    | The hide-don't-chain Start contract is required by `SimulatedLinearTreadmill`          |
| `Editor/MainWindow.cs`              | Auto-open hooks + EnsureControllers are load-bearing for scene initialization          |

If a change here is unavoidable, update the `automation:csharp-style` (ataraxis marketplace) verification pass, run the
`Sollertia.Tests.EditMode` and `Sollertia.Tests.PlayMode` suites, then verify on the target deployment's actual monitor
arrangement before committing.

---

## Common pitfalls

| Pitfall                                                                     | Fix                                                                                                                                                                                                                                                         |
|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `NullReferenceException` on `MQTTClient.Instance`                           | Ensure the scene has a `MQTT Client` GameObject (auto-created by MainWindow), and create channels in `Start()` rather than `Awake()`                                                                                                                        |
| `JsonUtility` returns default-valued messages                               | Payload class uses properties `{ get; set; }`, so convert them to public fields                                                                                                                                                                             |
| Actor's own model appears in its VR view (display camera renders the actor) | No project layer is named exactly after the actor GameObject, so check the console for `DisplayObject.ParentToActor: unable to cull the actor model` and add the layer, or re-run `InitiateActor` so `TagsAndLayers.AddLayer(gameObject.name)` allocates it |
| A section says "No Actor / Display / MQTT Client in the active scene"       | The scene predates `MainWindow.InitializeScene`, so close and reopen the window to auto-create the root                                                                                                                                                     |
| The Task section says "No Task component found in the current scene."       | Expected outside a generated task scene, and reopening the window does not repair it                                                                                                                                                                        |
| MQTT settings reset after reopening the project                             | `EditorPrefs` are per-user-per-project-path, which is expected, so reconfigure once                                                                                                                                                                         |
| Adding a new controller does not appear in the dropdown                     | Missing `ControllerTypes` enum entry, or `BuildControllerSpecs` could not resolve the type at `Gimbl.<EnumName>`                                                                                                                                            |

---

## Related skills

| Skill                            | Relationship                                                                                   |
|----------------------------------|------------------------------------------------------------------------------------------------|
| `/mqtt-contract` (this plugin)   | Topic catalog for every channel constructed on top of `MQTTChannel`                            |
| `/scene-setup` (this plugin)     | User-facing workflow for the MainWindow Task Parameters window                                 |
| `/task-parameters` (this plugin) | Programmatic mirror of the same window's Actor / MQTT / Display / Camera Mapping / Task fields |
| `/task-generator` (this plugin)  | Segment prefabs sit inside the actor's coordinate frame                                        |
| `/task-prefabs` (this plugin)    | Generated task prefabs sit inside the `ActorObject` coordinate frame                           |
| `/zone-prefabs` (this plugin)    | Modifier-script invariants for zone prefabs built on GIMBL primitives                          |
| `/unity-tests` (this plugin)     | Test-assembly layout and the EditMode / PlayMode fixtures covering every GIMBL type            |
| `automation:csharp-style`        | GIMBL code is held to the same C# conventions as the project                                   |
| `experiment:vr-driver-interface` | Python peer of `MQTTClient` / `MQTTTopics`, the host end of the wire contract                  |

---

## Verification checklist

```text
- [ ] New scripts that use MQTT create channels in Start(), not Awake() or constructors
- [ ] Typed channels use payload classes with public fields (no properties)
- [ ] Channels created in Start() are released in OnDestroy(): RemoveListener + client?.Unsubscribe(channel)
- [ ] Display / Controller assignments use the ActorObject property setters, not direct field access
- [ ] New controllers add a ControllerTypes enum value, a display-name case if needed, and a ControllerTests update
- [ ] New script folders declare an .asmdef and are referenced from every consuming assembly, tests included
- [ ] Changes to any GIMBL primitive pass its EditMode fixture (see Test coverage) and MqttClientPlayModeTests
- [ ] Display-rig changes were confirmed against the deployment's actual monitor count and layout
- [ ] `/mqtt-contract` is updated if any new channel is introduced via GIMBL primitives
- [ ] No hand-typed MQTT topic literals appear outside MQTTTopics.cs
```
