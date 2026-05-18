---
name: gimbl-framework
description: >-
  Reference for the GIMBL VR framework inlined under `Assets/Gimbl/` in sollertia-unity-tasks:
  ActorObject, ControllerObject hierarchy, DisplayObject rig, FullScreenViewManager,
  MQTTClient / MQTTChannel / MQTTTopics, and the MainWindow Task Parameters editor. Use when
  reading or modifying code that instantiates GIMBL components, wiring a new script to the MQTT
  broker, or diagnosing null-reference errors during scene init.
user-invocable: true
---

# GIMBL framework reference

The GIMBL VR framework is inlined into this codebase under `Assets/Gimbl/`. This skill is the
only source of truth for GIMBL in `sollertia-unity-tasks` — you MUST disregard any GIMBL
knowledge (file names, class shapes, editor windows, MQTT wiring, conventions) that does not
come from this skill or from the code under `Assets/Gimbl/` itself.

---

## Scope

**Covers:**
- GIMBL package layout under `Assets/Gimbl/` and the `Gimbl` C# namespace (current file inventory)
- `MainWindow` Task Parameters editor window (the only editor window)
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

---

## Directory layout

```text
Assets/Gimbl/
├── Editor/
│   └── MainWindow.cs                   ← `Window → Task Parameters` — the only editor window
├── Resources/
│   ├── Actors/
│   │   ├── Prefabs/
│   │   │   └── Rodent.prefab           ← Actor model prefab; loaded by `Resources.Load("Actors/Prefabs/<name>")`
│   │   └── Rodent/                     ← Source mesh, texture, and material for `Rodent.prefab`
│   ├── Displays/
│   │   └── MyMonitorSetup.prefab       ← Display model prefab; loaded by `Resources.Load("Displays/<name>")`
│   ├── Icons/                          ← GUI icons used by the editor window
│   └── Materials/                      ← Shared materials (e.g., monitor test pattern)
└── Scripts/
    ├── LayoutSettings.cs               ← Shared GUI styling helpers used by MainWindow
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

Actor and display model prefabs live in `Assets/Gimbl/Resources/Actors/Prefabs/` and
`Assets/Gimbl/Resources/Displays/`. `ActorObject.SetModel` and `DisplayObject.Create` load them
via `Resources.Load($"Actors/Prefabs/<name>")` and `Resources.Load($"Displays/<name>")`;
`MainWindow` enumerates the same folders via `Resources.LoadAll<GameObject>` to populate the
model dropdowns.

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
  runs `EnsureControllers`, then runs `EnsureMqttDefaults` to apply the project-wide broker IP /
  port. Existing GameObjects are left untouched. The `MQTT Client` GameObject is created with
  `hideFlags = HideFlags.HideInHierarchy`, so it does **not** appear in the scene Hierarchy
  panel. Looking for the MQTT client via the Hierarchy is futile; query it via
  `GameObject.Find("MQTT Client")` or `MQTTClient.Instance`.
- `EnsureControllers()` iterates `CachedControllerSpecs` (resolved once via reflection at type
  init) and creates one GameObject per `ControllerTypes` enum value. The actor's controller
  assignment is not auto-changed — user-chosen swaps survive a re-init.
- `EnsureMqttDefaults()` (public static) finds the scene's `MQTT Client` GameObject and applies
  the broker IP / port from `EditorPrefs` with a `127.0.0.1:1883` fallback. Used by
  `InitializeScene` and by `CreateTask.CreateSceneFromTemplate` so a fresh scene reports the
  default broker to MCP reads synchronously, without waiting for the next `delayCall` tick.
- `SyncDisplayBrightnessToSettings()` (public static) sets the active scene's
  `DisplayObject.currentBrightness` to its referenced `DisplaySettings.brightness` value. Called
  by `CreateTask.CreateSceneFromTemplate` only — deliberately not called from `InitializeScene`
  so per-scene customizations to `currentBrightness` survive subsequent scene-open passes.
- The window is `GetWindow<MainWindow>("Parameters", ..., new Type[] {InspectorWindow})` so it
  docks next to the Inspector by default; the docked tab label is the short `Parameters` and the
  menu entry is the long `Task Parameters` to disambiguate.
- **Scene-component cache.** `MainWindow` caches `_cachedTask`, `_cachedActor`, `_cachedDisplay`,
  `_cachedGuidanceZone`, and `_cachedOccupancyZone` references on first access and invalidates
  them on `EditorSceneManager.activeSceneChangedInEditMode`. Without the cache, every `OnGUI`
  repaint (potentially many times per second) would run five `FindAnyObjectByType` walks of the
  scene. A code path that swaps a Task / Actor / Display / zone component in the active scene
  without going through the scene-change hook (rare) must call `InvalidateSceneCache` itself.

### `Gimbl.MQTTClient`

Singleton `MonoBehaviour` that owns the MQTTnet client. Attached to the auto-created `MQTT Client`
GameObject in every scene initialized by `MainWindow`.

- `MQTTClient.Instance` — static property; null until the GameObject's `Awake()` runs.
- Connection settings (`ipAddress`, `port`) have C# field initializers of `"127.0.0.1"` and
  `1883`, so a freshly-`AddComponent`'d client reports the loopback default even before any hook
  runs. `Awake()` overlays the same values from `EditorPrefs` (`SollertiaVR_MQTT_IP` /
  `SollertiaVR_MQTT_Port`) with the loopback fallback, and `MainWindow.EnsureMqttDefaults`
  applies the same logic synchronously at scene init and at scene creation. The three paths
  converge on the same authoritative value.
- `Subscribe(channel, topic, qosLevel)` and `Publish(topic, payload)` are called by `MQTTChannel`;
  do not call them directly.
- **`Subscribe` late-connect failure mode**: the channel is added to the internal `_channelList`
  unconditionally so the in-process loopback (below) can route messages, but the broker-side
  `SubscribeAsync` only fires when the client is currently connected. A channel created while the
  broker is offline will receive in-process loopback messages but will **not** auto-subscribe to
  the broker when it later comes online. Callers that need broker-delivered messages after a late
  broker connect must re-create the channel (or trigger a fresh subscribe pass).
- **MQTT 5.0 only**: `MqttClientOptionsBuilder().WithProtocolVersion(MqttProtocolVersion.V500)`.
  Brokers must accept MQTT 5.0 (Mosquitto 2.0+).
- **Connect has a 1000ms timeout**: `Connect()` runs `ConnectAsync` on a `Task.Run`, waits up
  to 1 second, and logs an error if the connection has not completed. The method returns either
  way — `MQTTClient` continues running in an unconnected state and `Publish` automatically falls
  back to the in-process loopback. A silently-failed connect is visible only via the
  `Could not connect to MQTT broker at <ip>:<port>` Debug.LogError, not via any thrown exception.
- **Raw `IMqttClient` is publicly accessible** as `MQTTClient.Instance.client`. External code
  can bypass the `MQTTChannel` abstraction and call `PublishAsync` / `SubscribeAsync` directly.
  This is a deliberate escape hatch for advanced cases; production code should still go through
  `MQTTChannel` so the in-process loopback and channel-list bookkeeping stay consistent.
- **In-process loopback**: when the broker is unreachable, `Publish` routes the payload directly
  to in-process subscribers on the matching topic so keyboard-only test runs without a broker
  still reach local listeners (for example, `LickStimulusSpawner`).
- **Lifecycle**: `Awake` sets `Instance` and loads `ipAddress` / `port` from `EditorPrefs` (with
  loopback fallback). `Connect()` is invoked externally by `MQTTConnectorObject.OnEnable()` —
  Unity runs every `OnEnable` after every `Awake` but before every `Start`, so the broker
  connection is established with the resolved settings before any subscriber `Start()` constructs
  channels. `Start` opens `SessionStart` / `SessionStop` channels and fires `SessionStart` after a
  1-second delay (gives downstream subscribers time to attach). `OnApplicationQuit` publishes
  `SessionStop`, unsubscribes every channel, and disposes the client. `OnDestroy` duplicates the
  dispose path so scene transitions that bypass quit still release the `IMqttClient`.

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
  Note: `qosLevel` controls **subscription** QoS only. `MQTTClient.Publish` is hardcoded to
  `MqttQualityOfServiceLevel.ExactlyOnce` regardless of the channel's `qosLevel`, so a lower
  `qosLevel` value affects only how the broker delivers inbound messages to this client.

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

**Malformed JSON throws.** `MQTTChannel<TMessage>.ReceivedMessage` wraps any
`JsonUtility.FromJson` failure in an `InvalidOperationException` whose message includes the
inner exception text. The exception propagates out of the broker's message-received callback,
which logs it but does not stop the client. This is distinct from the silent-failure mode where
`JsonUtility` returns a default-valued message (covered in Common pitfalls below) — that mode
fires when the payload class uses properties instead of public fields and the JSON parses
successfully but produces zeroed values.

### `Gimbl.MQTTTopics`

`public static class` holding every topic literal as `public const string`. Each entry has XML
remarks declaring `Direction`, `Payload`, and `Callers`. The single source of truth for topic
names — see `/mqtt-contract` for the audit-ready catalog. Never hardcode a topic string elsewhere.

### `Gimbl.MQTTConnectorObject`

Small `MonoBehaviour` whose only job is to call `MQTTClient.Instance.Connect(verbose: false)`
from `OnEnable()`. Attached to the `Logger` GameObject in
`Assets/Scenes/ExperimentTemplate.unity`, so every scene the `CreateTask` pipeline copies from
the template inherits one instance. Decoupling the connect call from `MQTTClient.Awake()` is
load-bearing: Unity runs every `OnEnable` after every `Awake` but before every `Start`, so the
broker connection is initiated only after `MQTTClient.Awake()` has populated `ipAddress` / `port`
from `EditorPrefs`, and finishes before any subscriber `Start()` constructs `MQTTChannel`
instances. Do not move the `Connect()` call back into `MQTTClient.Awake()` — the order would no
longer be guaranteed across all scripts in the scene.

### `Gimbl.ActorObject`

`MonoBehaviour` that represents the animal in the VR environment. Holds references to a
`DisplayObject` and a `ControllerOutput`.

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

- Setting `Display = someDisplay` calls `DisplayObject.ParentToActor(this)`, which reparents the
  display GameObject under the actor and configures camera culling so the actor's own layer is
  hidden from its own camera. Assigning `null` unparents.
- Setting `Controller = output` unhooks the previous controller's `actor` back-reference and wires
  the new one, marking the scene dirty (Editor only).
- `InitiateActor(modelName, trackCamera)` is the Editor-only bootstrap that parents under the
  `Actors` GameObject, adds a `CharacterController`, allocates a render layer, calls `SetModel`,
  and optionally creates a third-person tracking camera. The render-layer allocation goes through
  `TagsAndLayers.AddLayer(gameObject.name)`, which searches layer slots **starting at index 8**
  (Unity reserves 0–7 for built-in / default user layers). The tracking camera, when requested,
  scans every existing `TrackCam`-tagged camera in the scene, picks the lowest-numbered display
  index in `{0..7}` that no other tracking camera is using, and defaults to 7 if all are taken.
  Multi-actor scenes coordinate display assignments through this scan.
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

| Class                      | Driven by                     | Purpose                                  |
|----------------------------|-------------------------------|------------------------------------------|
| `LinearTreadmill`          | MQTT `Motion` topic           | Production treadmill input from hardware |
| `SimulatedLinearTreadmill` | Unity Input System (keyboard) | Developer testing without hardware       |

`SimulatedLinearTreadmill` **extends** `LinearTreadmill` but hides its private `Start()` and does
not chain. The lifecycle contract relies on Unity dispatching `Start` per most-derived type only;
if a third controller subclass appears, follow the same pattern (hide via its own `Start`; do not
chain). The non-chaining note in `LinearTreadmill.cs` is load-bearing for the contract.

**Thread-safety contract on the movement accumulator.** `ControllerObject.movement` is a
`ValueBuffer` written from two threads: `LinearTreadmill.OnMessage` runs on the MQTT message-dispatch 
thread (MQTTnet background), while `LinearTreadmill.ProcessMovement` runs on the Unity
main thread via `Update`. Both `OnMessage` and `ProcessMovement` take a `lock (movement)` around
the buffer mutation. Any future controller subclass that touches `movement` must take the same
lock, or it will race with the MQTT callback and drop or double-count movement samples.

**`SimulatedLinearTreadmill.MovementSpeedMultiplier = 8.0f`** is hardcoded as a private
`const float`. Anyone tuning keyboard-driven testing speed must edit this constant in
`SimulatedLinearTreadmill.cs`; there is no inspector field, Task Parameters entry, or MQTT topic
to override it at runtime. The lick trigger is wired to `_input.Player.Jump` (spacebar).

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
- Brightness override via `currentBrightness` (0–100). The C# field initializer is `100f`, but
  `MainWindow.SyncDisplayBrightnessToSettings` syncs it to `settings.brightness` (default `50f`)
  at scene creation, so fresh scenes start with `currentBrightness == settings.brightness`.
- Editor-only `Create(displayName, modelName)` that instantiates the display prefab from
  `Resources/Displays/<modelName>`, **reuses** an existing `DisplaySettings` asset at
  `Assets/VRSettings/Displays/<displayName>.asset` (creating a fresh one only when none exists),
  and adds a `Camera` child per `MeshRenderer` found. The camera's name is derived by
  `mesh.name.Replace("Monitor", " View")` — i.e., monitor meshes must be named `*Monitor` for the
  rename to produce a `<role> View` camera label; meshes named anything else keep their literal
  name as the camera label. The reuse semantic on the settings asset exists so user customizations
  to `brightness` and `heightInVR` survive subsequent scene rebuilds; replacing the asset
  wholesale would clobber them.

`Create` also wires each spawned camera into the rig in three non-obvious ways:

1. **`targetDisplay = 8`**. Unity's standard Game-view displays are indexed 0–7; targeting 8
   keeps the per-monitor cameras from rendering into any of those slots. They only become
   visible through the `FullScreenView` popups managed by `FullScreenViewManager`. If the
   target-display value is ever changed, the cameras will appear in the Game view and double-render.
2. **`MeshRenderer` and `MeshCollider` on each monitor mesh are disabled**. The mesh exists only
   as a positioning anchor for `PerspectiveProjection.projectionScreen`; re-enabling either
   component puts the monitor surface back into the rendered scene and into physics.
3. **`projection.setNearClipPlane = false`** is set explicitly on every spawned `PerspectiveProjection`
   (the field defaults to `true` on the component itself). The camera keeps the fixed
   `nearClipPlane = 0.3f` Create assigned instead of letting `PerspectiveProjection.UpdateView`
   recompute it each frame. Re-enabling auto-near-clip breaks the calibrated display rig.

Multi-monitor rigs (three displays for the mesoscope setup) use one `Camera` per monitor inside
the single `DisplayObject`. `FullScreenViewManager` binds those cameras to OS monitor indices.

### `Gimbl.PerspectiveProjection`

`[ExecuteInEditMode]` `MonoBehaviour` attached to every display camera by `DisplayObject.Create`.
It computes the off-axis projection and view matrices each `LateUpdate` from the camera's
position relative to its `projectionScreen` GameObject, then applies them to the camera. Running
in edit mode means the rig already looks correct in the Scene view without entering Play Mode.

- **Screen mesh type**: `UpdateView` reads `projectionScreen.GetComponent<MeshFilter>().sharedMesh.name`
  once and branches on `"Plane"` (10×10 unit, lies on XZ plane, lower-left at `(-5, 0, -5)`) or
  `"Quad"` (1×1 unit, lies on XY plane, lower-left at `(-0.5, -0.5, 0)`). Any other mesh name
  silently skips the projection update — the camera continues rendering with whatever matrices
  Unity assigned last.
- **Negative-scale handling**: when the eye is behind the screen (dot-product test on
  `screenLowerLeft → upperLeft × lowerRight`), the screen axes are flipped before normalization.
  This is what lets the project's Right wall use a negative geometry scale to mirror its cue
  texture without inverting the projection. It is also why the cue material must use Unity's
  `Legacy Shaders/Diffuse` (the Standard shader breaks under negative scales — see
  `/task-generator` for the cue shader contract).
- **Brightness post-process**: `OnRenderImage` blits the camera's output through a `Material`
  whose shader is `Hidden/BrightnessShader`, with `_brightness` set from `displayObject.currentBrightness`
  (or `100f` if no `displayObject` is wired). This is how the `Blank` / `Show` toggle in the
  Display section dims or restores the display.
- **`estimateViewFrustum`** (default `true`) points the camera at the screen center and sets its
  FOV to approximate the off-axis projection so Unity's culling — which uses the symmetric
  frustum, not the matrix written above — keeps the visible region.
- **`setNearClipPlane`** (default `true`) auto-adjusts the camera's near clip plane to the
  eye-to-screen distance. `DisplayObject.Create` explicitly sets this to `false` on every spawned
  camera so the configured `nearClipPlane = 0.3f` is preserved. Re-enabling it on a rig camera
  will break the calibrated display.

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

`LoadCameras` and `SaveCameras` skip the saved-views asset I/O entirely when the active scene has
no name (untitled / unsaved buffer). That guards against creating an orphan
`-savedFullScreenViews.asset` (hyphen-prefixed, no scene to consume it) when the Parameters
window or the McpBridge writes camera mapping before the scene has been saved. The companion
asset is also cascade-deleted when its owning scene is removed via `delete_asset_tool`
(see `/task-scenes`).

**Monitor enumeration timeout.** `Monitor.EnumerateMonitors` calls `xrandr` (Linux) or
`/usr/local/bin/displayplacer list` (macOS) as a subprocess with a hardcoded 5000ms timeout.
If the subprocess hangs, the manager kills it, logs a warning, and returns whatever monitors
were parsed before the timeout. On Windows the enumeration is a synchronous P/Invoke
(`EnumDisplayMonitors`) and has no timeout. Every detected monitor is then probed for its
`EditorGUIUtility.pixelsPerPoint` via a temporary 20×20 popup `MonitorTester` window, which
produces a brief visual flicker on each display.

**One-camera-per-monitor invariant.** `RenderMonitorRow` silently ignores any selection that
would alias another monitor's camera. The dropdown change appears to apply but the underlying
`cameraEntityId` is not reassigned (and the saved-views asset is not rewritten). To bind a
camera that is already assigned elsewhere, first set the original monitor's dropdown back to
`None`, then assign the new monitor.

---

## GIMBL editor window

`MainWindow` registers the only menu entry: `Window → Task Parameters`. Selecting it opens (or
focuses) the window. The five sections are rendered in fixed order by `OnGUI`:

1. **Actor** — model + controller dropdowns
2. **MQTT** — broker IP / port + Test Connection (greyed in Play Mode)
3. **Display** — Blank / Show toggle + `brightness` + `heightInVR`
4. **Camera Mapping** — Refresh + per-monitor camera dropdowns + Show Full-Screen Views (button
   greyed in Play Mode)
5. **Task** — `Require Lick` / `Require Wait` (conditional on zone presence) + `Track Length` /
   `Track Seed`. The entire Task section's controls are greyed in Play Mode.

See `/scene-setup` for the user-facing workflow and `/task-parameters` for the programmatic
mirror.

---

## GIMBL vs `SL.*` responsibility split

Use this table to decide which layer a change belongs in.

| Concern                                       | Owner         | Example files                                              |
|-----------------------------------------------|---------------|------------------------------------------------------------|
| Animal representation and display binding     | GIMBL         | `ActorObject.cs`, `DisplayObject.cs`                       |
| MQTT transport and serialization primitives   | GIMBL         | `MQTTClient.cs`, `MQTTChannel.cs`, `MQTTTopics.cs`         |
| Hardware treadmill input                      | GIMBL         | `LinearTreadmill.cs`                                       |
| Simulated treadmill input                     | GIMBL         | `SimulatedLinearTreadmill.cs`, `SimulatedInput.cs`         |
| Display rig projection math                   | GIMBL         | `PerspectiveProjection.cs`, `Monitor.cs`                   |
| Task Parameters editor window                 | GIMBL         | `MainWindow.cs`, `LayoutSettings.cs`                       |
| Corridor teleportation and cue sequence       | `SL.Tasks`    | `Task.cs`                                                  |
| Stimulus trigger zones and behavior modes     | `SL.Tasks`    | `StimulusTriggerZone.cs`, `OccupancyZone.cs`               |
| YAML template schema and loader               | `SL.Config`   | `ConfigLoader.cs`, `TaskTemplate.cs`                       |
| Editor-side prefab generation                 | `SL.Tasks`    | `CreateTask.cs` (Editor)                                   |
| Editor MCP bridge                             | `SL.Tasks`    | `McpBridge.cs` (Editor)                                    |
| UI feedback for lick / stimulus events        | `SL.UI`       | `LickStimulusSpawner.cs`                                   |

Rule of thumb: a change to actor / display / controller / MQTT primitives or the Task Parameters
editor belongs under `Assets/Gimbl/`. A change specific to the infinite corridor task, the
cue / segment / trial vocabulary, or the `sollertia-experiment` MQTT contract belongs under
`Assets/InfiniteCorridorTask/` or `Assets/UI-lick-reward/`.

---

## Do-not-modify zones

Avoid editing these GIMBL files unless absolutely required. They are either load-bearing across
the framework or easy to break in non-obvious ways:

| File                                | Reason                                                                              |
|-------------------------------------|-------------------------------------------------------------------------------------|
| `MQTT/MQTTClient.cs`                | Singleton lifecycle and in-process loopback; breakage leaves every channel dangling |
| `Displays/PerspectiveProjection.cs` | Projection matrix math is calibrated for the mesoscope rig                          |
| `Displays/Monitor.cs`               | Editor + runtime coordination for full-screen views                                 |
| `Controllers/ControllerObject.cs`   | Abstract base used by polymorphic editor inspectors                                 |
| `Controllers/LinearTreadmill.cs`    | The hide-don't-chain Start contract is required by `SimulatedLinearTreadmill`       |
| `Editor/MainWindow.cs`              | Auto-open hooks + EnsureControllers are load-bearing for scene initialization       |

If a change here is unavoidable, update the `/csharp-style` verification pass and test against the
full display rig (three monitors) before committing.

---

## Common pitfalls

| Pitfall                                                                 | Fix                                                                                                                      |
|-------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `NullReferenceException` on `MQTTClient.Instance`                       | Ensure the scene has a `MQTT Client` GameObject (auto-created by MainWindow); create channels in `Start()` not `Awake()` |
| `JsonUtility` returns default-valued messages                           | Payload class uses properties `{ get; set; }` — convert to public fields                                                 |
| Display attaches but actor sees through its own walls                   | Camera culling layer missing; check that the actor's layer matches the scene hierarchy                                   |
| `Window → Task Parameters` opens but a section says "missing component" | The scene predates `MainWindow.InitializeScene`; re-open the window to repair                                            |
| MQTT settings reset after reopening the project                         | `EditorPrefs` are per-user-per-project-path — expected; reconfigure once                                                 |
| Adding a new controller does not appear in the dropdown                 | Missing `ControllerTypes` enum entry, or `BuildControllerSpecs` could not resolve the type at `Gimbl.<EnumName>`         |

---

## Verification checklist

```text
- [ ] New scripts that use MQTT create channels in Start(), not Awake() or constructors
- [ ] Typed channels use payload classes with public fields (no properties)
- [ ] Null-conditional operator (?.) used when removing listeners in OnDestroy()
- [ ] Display / Controller assignments use the ActorObject property setters, not direct field access
- [ ] New controllers add a ControllerTypes enum value AND a display-name case if needed
- [ ] Changes to MQTTClient, PerspectiveProjection, Monitor, or MainWindow have been tested on the three-monitor rig
- [ ] `/mqtt-contract` is updated if any new channel is introduced via GIMBL primitives
- [ ] No hand-typed MQTT topic literals appear outside MQTTTopics.cs
```

---

## Related skills

| Skill                               | Relationship                                                                                   |
|-------------------------------------|------------------------------------------------------------------------------------------------|
| `/mqtt-contract` (this plugin)      | Topic catalog for every channel constructed on top of `MQTTChannel`                            |
| `/scene-setup` (this plugin)        | User-facing workflow for the MainWindow Task Parameters window                                 |
| `/task-parameters` (this plugin)    | Programmatic mirror of the same window's Actor / MQTT / Display / Camera Mapping / Task fields |
| `/task-generator` (this plugin)     | Segment prefabs sit inside the actor's coordinate frame                                        |
| `/csharp-style` (automation plugin) | GIMBL code is held to the same C# conventions as the project                                   |
