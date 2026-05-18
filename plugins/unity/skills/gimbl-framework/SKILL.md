---
name: gimbl-framework
description: >-
  Reference for the GIMBL VR framework inlined under `Assets/Gimbl/` in sollertia-unity-tasks:
  ActorObject, ControllerObject hierarchy (LinearTreadmill, SimulatedLinearTreadmill,
  ControllerOutput indirection), DisplayObject rig, FullScreenViewManager, MQTTClient /
  MQTTChannel, MQTTTopics, and the consolidated MainWindow Task Parameters editor. Use when
  reading or modifying code that instantiates GIMBL components, wiring a new script to the MQTT
  broker, or diagnosing null-reference errors during scene init.
user-invocable: true
---

# GIMBL framework reference

Documents the GIMBL VR framework as **inlined** into `sollertia-unity-tasks` under
`Assets/Gimbl/`. GIMBL is a third-party VR task framework that the project depends on; the inlined
copy has been heavily refactored (legacy editor windows consolidated into a single Task Parameters
window, MQTT switched to MQTTnet with MQTT 5.0, controller indirection added) and is no longer
tracked against the upstream repository.

---

## Scope

**Covers:**
- GIMBL package layout under `Assets/Gimbl/` and the `Gimbl` C# namespace (current file inventory)
- `MainWindow` consolidated Task Parameters editor window (the only editor window)
- `ActorObject` (animal representation, display + controller binding, model swap)
- Controller hierarchy: `ControllerObject` → `LinearTreadmill` → `SimulatedLinearTreadmill`, with
  `ControllerOutput` indirection
- `ControllerTypes` enum and `MainWindow.EnsureControllers` auto-create loop
- `DisplayObject` and `FullScreenViewManager` multi-monitor rig
- `MQTTClient` singleton, `MQTTChannel` / `MQTTChannel<T>` typed messaging, `MQTTTopics` constants
- Decision rules for whether a given behavior belongs in GIMBL or the `SL.*` task layer

**Does not cover:**
- Individual MQTT topic wiring and the topic catalog (see `/mqtt-contract`)
- Programmatic Task Parameters read / write (see `/task-parameters`)
- Editor scene configuration workflows (see `/scene-setup`)
- Task prefab generation (see `/task-generator`, `/task-prefabs`)
- Upstream GIMBL releases — the inlined copy is independently maintained

---

## Directory layout

```text
Assets/Gimbl/
├── Editor/
│   └── MainWindow.cs                  ← `Window → Task Parameters` — the only editor window
└── Scripts/
    ├── LayoutSettings.cs              ← Shared GUI styling helpers used by MainWindow
    ├── TagsAndLayers.cs               ← Tag / layer creation helpers (e.g., VRDisplay, TrackCam)
    ├── Actor/
    │   └── ActorObject.cs             ← Animal GameObject facade (Display, Controller, Model)
    ├── Controllers/
    │   ├── ControllerObject.cs        ← Abstract base + ValueBuffer movement accumulator
    │   ├── ControllerOutput.cs        ← Type-erased reference held by ActorObject.Controller
    │   ├── ControllerTypes.cs         ← Enum of supported controller subclasses
    │   ├── LinearTreadmill.cs         ← MQTT-driven (`Motion`) hardware treadmill
    │   ├── SimulatedLinearTreadmill.cs ← Keyboard-driven; hides Start() per non-chaining contract
    │   └── SimulatedInput.cs          ← Auto-generated Unity Input System action map
    ├── Displays/
    │   ├── DisplayObject.cs           ← Attach-to-actor display rig with per-monitor cameras
    │   ├── DisplaySettings.cs         ← ScriptableObject under Assets/VRSettings/Displays/
    │   ├── Monitor.cs                 ← Monitor entry in the FullScreenViewManager list
    │   ├── PerspectiveProjection.cs   ← Per-monitor projection matrix math (calibrated)
    │   ├── FullScreenView.cs          ← Runtime full-screen window per monitor
    │   ├── FullScreenViewManager.cs   ← Editor-side camera ↔ monitor binding + save asset
    │   └── FullScreenViewsSaved.cs    ← ScriptableObject persisting per-scene camera bindings
    └── MQTT/
        ├── MQTTClient.cs              ← MonoBehaviour broker connection singleton (MQTTnet)
        ├── MQTTChannel.cs             ← Untyped and typed (`MQTTChannel<T>`) channel classes
        ├── MQTTTopics.cs              ← `public const string` catalog of every topic
        └── MQTTConnectorObject.cs     ← Legacy connector utility (still present, rarely used)
```

There is no longer an `ActorWindow.cs`, `DisplaysWindow.cs`, `SettingsWindow.cs`, or
`ActorSettings.cs` file — those split-window editors were consolidated into `MainWindow.cs`
during the GUI optimization pass. There is also no top-level `Resources/` folder under
`Assets/Gimbl/`; actor and display model prefabs live under the project's `Assets/.../Resources/`
folders (`Actors/Prefabs/...`, `Displays/...`) that the runtime loads via `Resources.LoadAll`.

---

## Core types

### `Gimbl.MainWindow`

`EditorWindow` registered under `Window → Task Parameters`. The single GUI entry point for every
per-scene configuration field (Actor, MQTT, Display, Camera Mapping, Task). See `/scene-setup` for
the GUI workflow and `/task-parameters` for the programmatic surface.

Notable invariants:

- `[InitializeOnLoadMethod] RegisterAutoOpen` subscribes to `EditorSceneManager.sceneOpened`,
  `EditorApplication.playModeStateChanged`, and `EditorApplication.delayCall` so the window
  reappears after Editor start, scene open, or Play Mode enter. Closing it manually is
  recoverable; the next hook fires re-opens it.
- `OnEnable() → InitializeScene()` ensures the active scene contains `Actors`, `Controllers`,
  `MQTT Client` roots, removes the Unity-default Main Camera, creates a default Actor + Display,
  and runs `EnsureControllers`. Existing GameObjects are left untouched.
- `EnsureControllers()` iterates `CachedControllerSpecs` (resolved once via reflection at type
  init) and creates one GameObject per `ControllerTypes` enum value. The actor's controller
  assignment is not auto-changed — user-chosen swaps survive a re-init.
- The window is `GetWindow<MainWindow>("Parameters", ..., new Type[] {InspectorWindow})` so it
  docks next to the Inspector by default; the docked tab label is the short `Parameters` and the
  menu entry is the long `Task Parameters` to disambiguate.

### `Gimbl.MQTTClient`

Singleton `MonoBehaviour` that owns the MQTTnet client. Attached to the auto-created `MQTT Client`
GameObject in every scene initialized by `MainWindow`.

- `MQTTClient.Instance` — static property; null until the GameObject's `Awake()` runs.
- Connection settings (`ipAddress`, `port`) load from `EditorPrefs` in `Awake()`, with a
  `127.0.0.1:1883` fallback when the prefs are missing.
- `Subscribe(channel, topic, qosLevel)` and `Publish(topic, payload)` are called by `MQTTChannel`;
  do not call them directly.
- **MQTT 5.0 only**: `MqttClientOptionsBuilder().WithProtocolVersion(MqttProtocolVersion.V500)`.
  Brokers must accept MQTT 5.0 (Mosquitto 2.0+).
- **In-process loopback**: when the broker is unreachable, `Publish` routes the payload directly
  to in-process subscribers on the matching topic so keyboard-only test runs without a broker
  still reach local listeners (for example, `LickStimulusSpawner`).
- **Lifecycle**: `Awake` sets `Instance`, `Start` opens `SessionStart` / `SessionStop` channels and
  fires `SessionStart` after a 1-second delay (gives downstream subscribers time to attach).
  `OnApplicationQuit` publishes `SessionStop`, unsubscribes every channel, and disposes the client.
  `OnDestroy` duplicates the dispose path so scene transitions that bypass quit still release the
  `IMqttClient`.

**Lifecycle gotcha:** `MQTTChannel` constructors throw `InvalidOperationException` if
`MQTTClient.Instance` is null. This typically means the scene lacks a `MQTT Client` GameObject (or
the script calling `new MQTTChannel(...)` is running in `Awake()`). Always create channels in
`Start()` — Unity guarantees every `Awake()` runs before any `Start()`.

### `Gimbl.MQTTChannel`

Untyped trigger channel for topics carrying no payload.

```csharp
var channel = new MQTTChannel(MQTTTopics.MyTopic, isListener: true, qosLevel: 2);
channel.receivedEvent.AddListener(OnMyEvent);  // void OnMyEvent() { ... }

// Later, to publish:
channel.Send();

// On teardown:
channel.receivedEvent.RemoveListener(OnMyEvent);
```

Parameters:
- `topicString` — flat PascalCase constant from `MQTTTopics` (no trailing slash).
- `isListener` — `true` to subscribe, `false` for publish-only.
- `qosLevel` — defaults to 2 (`ExactlyOnce`); do not downgrade without cross-team agreement.

### `Gimbl.MQTTChannel<TMessage>`

Typed channel using `JsonUtility` for serialization. The payload type must have plain public
fields — `JsonUtility` does not read properties.

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

Serialization bytes: `Encoding.UTF8.GetBytes(JsonUtility.ToJson(message))`. The typed
`receivedEvent` shadows the base class's parameterless event with the `new` modifier — accessing
the channel through a base-class reference exposes only the trigger event and silently misses the
typed callback. See the XML remarks in `MQTTChannel.cs` for the rationale.

### `Gimbl.MQTTTopics`

`public static class` holding every topic literal as `public const string`. Each entry has XML
remarks declaring `Direction`, `Payload`, and `Callers`. The single source of truth for topic
names — see `/mqtt-contract` for the audit-ready catalog. Never hardcode a topic string elsewhere.

### `Gimbl.ActorObject`

`MonoBehaviour` that represents the animal in the VR environment. Holds references to a
`DisplayObject` and a `ControllerOutput`.

```csharp
public class ActorObject : MonoBehaviour
{
    public DisplayObject Display { get; set; }       // auto-parents display on set
    public ControllerOutput Controller { get; set; } // wires Controller.master.actor on set
}
```

- Setting `Display = someDisplay` calls `DisplayObject.ParentToActor(this)`, which reparents the
  display GameObject under the actor and configures camera culling so the actor's own layer is
  hidden from its own camera. Assigning `null` unparents.
- Setting `Controller = output` unhooks the previous controller's `actor` back-reference and wires
  the new one, marking the scene dirty (Editor only).
- `InitiateActor(modelName, trackCamera)` is the Editor-only bootstrap that parents under the
  `Actors` GameObject, adds a `CharacterController`, allocates a render layer, calls `SetModel`,
  and optionally creates a third-person tracking camera.
- `SetModel(modelName)` swaps the actor's `Model <name>` child GameObject by instantiating
  `Resources.Load("Actors/Prefabs/<modelName>")`. `"None"` removes the existing model without
  adding a new one.
- `EditMenu()` renders the Actor section of `MainWindow` (model + controller dropdowns).

The task layer (`SL.Tasks.Task`) holds a reference to the actor via the public `actor` field, but
the Inspector hides it (`[HideInInspector]`) and `Task.OnValidate()` auto-assigns it from the
scene. The Task Parameters window's Task section also re-resolves the actor when the field is
null on each render.

### `Gimbl.ControllerObject` and `ControllerOutput`

Abstract base for input devices that drive actor movement. The only runtime subclasses are:

| Class                      | Driven by                     | Purpose                                       |
|----------------------------|-------------------------------|-----------------------------------------------|
| `LinearTreadmill`          | MQTT `Motion` topic           | Production treadmill input from hardware       |
| `SimulatedLinearTreadmill` | Unity Input System (keyboard) | Developer testing without hardware             |

`SimulatedLinearTreadmill` **extends** `LinearTreadmill` but hides its private `Start()` and does
not chain. The lifecycle contract relies on Unity dispatching `Start` per most-derived type only;
if a third controller subclass appears, follow the same pattern (hide via its own `Start`; do not
chain). The non-chaining note in `LinearTreadmill.cs` is load-bearing for the contract.

**`ControllerOutput` indirection**: `ActorObject.Controller` is typed as `ControllerOutput` rather
than the concrete `ControllerObject` subclass so swapping controllers does not invalidate the
scene's serialized reference. `MainWindow.EnsureControllers` attaches one `ControllerOutput` per
generated controller GameObject and sets `master` to the sibling controller component. The
indirection erases the subclass distinction at the Inspector field level.

### `Gimbl.ControllerTypes`

Enum of supported controller subclasses (`LinearTreadmill`, `SimulatedLinearTreadmill`).
`MainWindow.BuildControllerSpecs` resolves each enum value to its runtime `Type` via reflection
(`controllerAssembly.GetType($"Gimbl.{enumName}")`) once at type init, and maps each to a display
name via a small switch (`LinearTreadmill → "Linear"`, `SimulatedLinearTreadmill → "Simulated
Linear"`). Adding a new controller class requires:

1. Subclass `ControllerObject` (and hide `Start` per the non-chaining contract if the subclass adds
   an MQTT subscription).
2. Add a value to `ControllerTypes`.
3. Optionally add a display-name case to `BuildControllerSpecs` if the default `ToString` is not
   user-friendly.

`MainWindow.EnsureControllers` will pick up the new entry on the next scene init.

### `Gimbl.DisplayObject`

Represents a VR display attached to an actor. Manages:

- Parenting under the actor and Y offset from `settings.heightInVR`
- Camera culling to hide the actor's own layer from its own camera
- Brightness override via `currentBrightness` (0–100)
- Editor-only `Create(displayName, modelName)` that instantiates the display prefab from
  `Resources/Displays/<modelName>`, allocates a `DisplaySettings` asset, and adds a `Camera`
  child per `MeshRenderer` found (each renamed to `<role> View`).

Multi-monitor rigs (three displays for the mesoscope setup) use one `Camera` per monitor inside
the single `DisplayObject`. `FullScreenViewManager` binds those cameras to OS monitor indices.

### `Gimbl.FullScreenViewManager`

Editor-side helper consumed by `MainWindow`'s Camera Mapping section. Enumerates OS monitors via
`Monitor.EnumerateMonitors()`, persists camera bindings in
`Assets/VRSettings/Displays/<scene>-savedFullScreenViews.asset`, and creates one
`FullScreenView` per assigned monitor at Play Mode entry. The same instance is shared by the
McpBridge's `read_task_parameters` / `write_task_parameters` handlers so editor edits and MCP
edits stay in sync.

The McpBridge's `AcquireFullScreenManager` reuses the open Parameters window's manager when one
exists and otherwise constructs a fresh one (whose `LoadCameras` reads the saved asset). The
shared instance is the reason editor-side writes via `/task-parameters` are visible in the
already-open GUI without a reload.

---

## GIMBL editor window

`MainWindow` registers the only menu entry: `Window → Task Parameters`. Selecting it opens (or
focuses) the consolidated window. The five sections are rendered in fixed order by `OnGUI`:

1. **Actor** — model + controller dropdowns
2. **MQTT** — broker IP / port + Test Connection (greyed in Play Mode)
3. **Display** — Blank / Show toggle + `brightness` + `heightInVR`
4. **Camera Mapping** — Refresh + per-monitor camera dropdowns + Show Full-Screen Views (button
   greyed in Play Mode)
5. **Task** — `Require Lick` / `Require Wait` (conditional on zone presence) + `Track Length` /
   `Track Seed` (greyed in Play Mode)

See `/scene-setup` for the user-facing workflow and `/task-parameters` for the programmatic
mirror.

---

## GIMBL vs `SL.*` responsibility split

Use this table to decide which layer a change belongs in. GIMBL is refactored but still shared
framework code — changes there affect every project that adopts the inlined copy.

| Concern                                       | Owner         | Example files                                              |
|-----------------------------------------------|---------------|------------------------------------------------------------|
| Animal representation and display binding     | GIMBL         | `ActorObject.cs`, `DisplayObject.cs`                       |
| MQTT transport and serialization primitives   | GIMBL         | `MQTTClient.cs`, `MQTTChannel.cs`, `MQTTTopics.cs`         |
| Hardware treadmill input                      | GIMBL         | `LinearTreadmill.cs`                                       |
| Simulated treadmill input                     | GIMBL         | `SimulatedLinearTreadmill.cs`, `SimulatedInput.cs`         |
| Display rig projection math                   | GIMBL         | `PerspectiveProjection.cs`, `Monitor.cs`                   |
| Consolidated Task Parameters editor window    | GIMBL         | `MainWindow.cs`, `LayoutSettings.cs`                       |
| Corridor teleportation and cue sequence       | `SL.Tasks`    | `Task.cs`                                                  |
| Stimulus trigger zones and behavior modes     | `SL.Tasks`    | `StimulusTriggerZone.cs`, `OccupancyZone.cs`               |
| YAML template schema and loader               | `SL.Config`   | `ConfigLoader.cs`, `TaskTemplate.cs`                       |
| Editor-side prefab generation                 | `SL.Tasks`    | `CreateTask.cs` (Editor)                                   |
| Editor MCP bridge                             | `SL.Tasks`    | `McpBridge.cs` (Editor)                                    |
| UI feedback for lick / stimulus events        | `SL.UI`       | `LickStimulusSpawner.cs`                                   |

Rule of thumb: a change useful to any GIMBL-based project belongs under `Assets/Gimbl/`. A change
specific to the infinite corridor task, the cue / segment / trial vocabulary, or the
`sollertia-experiment` MQTT contract belongs under `Assets/InfiniteCorridorTask/` or
`Assets/UI-lick-reward/`.

---

## Do-not-modify zones

Avoid editing these GIMBL files unless absolutely required. They are either load-bearing across
the framework or easy to break in non-obvious ways:

| File                                    | Reason                                                                                |
|-----------------------------------------|----------------------------------------------------------------------------------------|
| `MQTT/MQTTClient.cs`                    | Singleton lifecycle and in-process loopback; breakage leaves every channel dangling   |
| `Displays/PerspectiveProjection.cs`     | Projection matrix math is calibrated for the mesoscope rig                            |
| `Displays/Monitor.cs`                   | Editor + runtime coordination for full-screen views                                   |
| `Controllers/ControllerObject.cs`       | Abstract base used by polymorphic editor inspectors                                   |
| `Controllers/LinearTreadmill.cs`        | The hide-don't-chain Start contract is required by `SimulatedLinearTreadmill`         |
| `Editor/MainWindow.cs`                  | Auto-open hooks + EnsureControllers are load-bearing for scene initialization         |

If a change here is unavoidable, update the `/csharp-style` verification pass and test against the
full display rig (three monitors) before committing.

---

## Common pitfalls

| Pitfall                                                   | Fix                                                                                    |
|-----------------------------------------------------------|----------------------------------------------------------------------------------------|
| `NullReferenceException` on `MQTTClient.Instance`         | Ensure the scene has a `MQTT Client` GameObject (auto-created by MainWindow); create channels in `Start()` not `Awake()` |
| `JsonUtility` returns default-valued messages             | Payload class uses properties `{ get; set; }` — convert to public fields              |
| Display attaches but actor sees through its own walls     | Camera culling layer missing; check that the actor's layer matches the scene hierarchy |
| `Window → Task Parameters` opens but a section says "missing component" | The scene predates `MainWindow.InitializeScene`; re-open the window to repair    |
| MQTT settings reset after reopening the project           | `EditorPrefs` are per-user-per-project-path — expected; reconfigure once             |
| `SimulatedLinearTreadmill` publishes duplicate licks      | Holding mouse left button OR space keeps firing; tap once per intended lick           |
| Adding a new controller does not appear in the dropdown   | Missing `ControllerTypes` enum entry, or `BuildControllerSpecs` could not resolve the type at `Gimbl.<EnumName>` |

---

## Verification checklist

```text
- [ ] New scripts that use MQTT create channels in Start(), not Awake() or constructors
- [ ] Typed channels use payload classes with public fields (no properties)
- [ ] Null-conditional operator (?.) used when removing listeners in OnDestroy()
- [ ] Display / Controller assignments use the ActorObject property setters, not direct field access
- [ ] New controllers add a ControllerTypes enum value AND a display-name case if needed
- [ ] Changes to MQTTClient, PerspectiveProjection, Monitor, or MainWindow have been tested against the full multi-monitor rig
- [ ] `/mqtt-contract` is updated if any new channel is introduced via GIMBL primitives
- [ ] No hand-typed MQTT topic literals appear outside MQTTTopics.cs
```

---

## Related skills

| Skill                               | Relationship                                                                            |
|-------------------------------------|------------------------------------------------------------------------------------------|
| `/mqtt-contract` (this plugin)      | Topic catalog for every channel constructed on top of `MQTTChannel`                      |
| `/scene-setup` (this plugin)        | User-facing workflow for the MainWindow Task Parameters window                           |
| `/task-parameters` (this plugin)    | Programmatic mirror of the same window's Actor / MQTT / Display / Camera Mapping / Task fields |
| `/task-generator` (this plugin)     | Segment prefabs sit inside the actor's coordinate frame                                  |
| `/csharp-style` (automation plugin) | GIMBL code is held to the same C# conventions as the project                             |
