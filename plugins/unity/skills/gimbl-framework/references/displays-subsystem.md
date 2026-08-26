# GIMBL displays subsystem

Class-by-class reference for `Assets/Gimbl/Scripts/Displays/`. Load this when touching the display rig: `DisplayObject`,
`PerspectiveProjection`, `Monitor`, `FullScreenView`, `FullScreenViewManager`, or `FullScreenViewsSaved`. `SKILL.md`
owns the directory map, the assembly layout, and the do-not-modify table. `/scene-setup` owns the Camera Mapping GUI
workflow and `/task-parameters` its programmatic mirror.

---

## `Gimbl.DisplayObject`

Represents a VR display attached to an actor. Manages:

- Parenting under the actor and Y offset from `settings.heightInVR`
- Camera culling to hide the actor's own layer from its own camera
- Brightness override via `currentBrightness` (0 to 100). The C# field initializer is `100f`, but
  `MainWindow.SyncDisplayBrightnessToSettings` syncs it to `settings.brightness` (default `50f`) at scene creation, so
  fresh scenes start with `currentBrightness == settings.brightness`.
- Editor-only `Create(displayName, modelName)` that instantiates the display prefab from
  `Resources/Displays/<modelName>`, **reuses** an existing `DisplaySettings` asset at
  `Assets/VRSettings/Displays/<displayName>.asset` (creating a fresh one only when none exists), and adds a `Camera`
  child per `MeshRenderer` found. The camera's name is derived by `mesh.name.Replace("Monitor", " View")`, so a monitor
  mesh named `*Monitor` produces a `<role> View` camera label and a mesh named anything else keeps its literal name as
  the camera label. The reuse semantic on the settings asset exists so user customizations to `brightness` and
  `heightInVR` survive subsequent scene rebuilds. Replacing the asset wholesale would clobber them.

`Create` also wires each spawned camera into the rig in three non-obvious ways:

1. **`targetDisplay = 8`**. Unity's standard Game-view displays are indexed 0 through 7, so targeting 8 keeps the
   per-monitor cameras from rendering into any of those slots. They only become visible through the `FullScreenView`
   popups managed by `FullScreenViewManager`. If the target-display value is ever changed, the cameras will appear in
   the Game view and double-render.
2. **`MeshRenderer` and `MeshCollider` on each monitor mesh are disabled**. The mesh exists only as a positioning anchor
   for `PerspectiveProjection.projectionScreen`. Re-enabling either component puts the monitor surface back into the
   rendered scene and into physics.
3. **`projection.setNearClipPlane = false`** is set explicitly on every spawned `PerspectiveProjection` (the field
   defaults to `true` on the component itself). The camera keeps the fixed `nearClipPlane = 0.3f` Create assigned
   instead of letting `PerspectiveProjection.UpdateView` recompute it each frame. Re-enabling auto-near-clip breaks the
   calibrated display rig.

Multi-monitor rigs use one `Camera` per monitor mesh inside the single `DisplayObject`. The count is whatever the
display model prefab under `Resources/Displays/` contains, and this project's standard rig is three displays (see
`/scene-setup`). `FullScreenViewManager` binds those cameras to OS monitor indices.

---

## `Gimbl.PerspectiveProjection`

`[ExecuteInEditMode]` `MonoBehaviour` attached to every display camera by `DisplayObject.Create`. It computes the
off-axis projection and view matrices each `LateUpdate` from the camera's position relative to its `projectionScreen`
GameObject, then applies them to the camera. Running in edit mode means the rig already looks correct in the Scene view
without entering Play Mode.

- **Screen mesh type**: `UpdateView` reads `projectionScreen.GetComponent<MeshFilter>().sharedMesh.name` and branches on
  `"Plane"` (10x10 unit, lies on XZ plane, lower-left at `(-5, 0, -5)`) or `"Quad"` (1x1 unit, lies on XY plane,
  lower-left at `(-0.5, -0.5, 0)`). The `switch` has a `default` case that logs `Unable to update the off-axis
  projection of '<camera>'. The mesh of projection screen '<screen>' must be named Plane or Quad, but it is named
  '<name>'.` and returns before any matrix is computed. The warning is emitted only when the mesh type is (re-)resolved,
  rather than once per `LateUpdate`, so an unsupported mesh reports once and the camera simply keeps its previous
  projection. Name every projection screen's `sharedMesh` asset `Plane` or `Quad`.
- **Negative-scale handling**: when the eye is behind the screen (dot-product test on `screenLowerLeft → upperLeft ×
  lowerRight`), the screen axes are flipped before normalization. This is what lets the project's Right wall use a
  negative geometry scale to mirror its cue texture without inverting the projection. It is also why the cue material
  must use Unity's `Legacy Shaders/Diffuse`, because the Standard shader breaks under negative scales. See
  `/task-generator` for the cue shader contract.
- **Brightness post-process**: `OnRenderImage` blits the camera's output through a `Material` whose shader is
  `Hidden/BrightnessShader`, with `_brightness` set from `displayObject.currentBrightness` (or `100f` if no
  `displayObject` is wired). This is how the `Blank` / `Show` toggle in the Display section dims or restores the
  display.
- **`estimateViewFrustum`** (default `true`) points the camera at the screen center and sets its FOV to approximate the
  off-axis projection so Unity's culling keeps the visible region. That culling reads the symmetric frustum rather than
  the matrix written above, which is why the approximation is needed.
- **`setNearClipPlane`** (default `true`) auto-adjusts the camera's near clip plane to `eyeToScreenDistance +
  nearClipDistanceOffset`, where the public `nearClipDistanceOffset` field defaults to `-0.01f` (pulling the plane
  slightly in front of the screen surface). `DisplayObject.Create` explicitly sets `setNearClipPlane` to `false` on
  every spawned camera so the configured `nearClipPlane = 0.3f` is preserved. Re-enabling it on a rig camera will break
  the calibrated display.

---

## `Gimbl.FullScreenViewManager`

Editor-side helper consumed by `MainWindow`'s Camera Mapping section. Enumerates OS monitors via
`Monitor.EnumerateMonitors()`, persists camera bindings in
`Assets/VRSettings/Displays/<scene>-savedFullScreenViews.asset` (a `FullScreenViewsSaved` `ScriptableObject` holding one
camera GameObject path per monitor index), and creates one `FullScreenView` (a borderless popup `EditorWindow`) per
assigned monitor at Play Mode entry. The same instance is shared by the McpBridge's `read_task_parameters`,
`write_task_parameters`, and `refresh_monitors` handlers so editor edits and MCP edits stay in sync.

The McpBridge's `AcquireFullScreenManager` reuses the open Parameters window's manager when one exists and otherwise
reuses a per-scene cached manager (`_cachedFullScreenManager ??= new FullScreenViewManager()`), whose constructor
already runs `LoadCameras` against the saved asset, so no second load happens. The cache is cleared on every
`activeSceneChangedInEditMode`, so monitor enumeration runs once per scene rather than once per request, which is why a
physical monitor rearrangement mid-session needs an explicit `refresh_monitors` call. The shared instance is the reason
editor-side writes via `/task-parameters` are visible in the already-open GUI without a reload.

**`RefreshMonitorPositions()`** is the shared re-detection entry point behind both the Camera Mapping "Refresh Monitor
Positions" button and the `refresh_monitors` MCP tool. It re-runs `Monitor.EnumerateMonitors()` and carries existing
camera assignments across **by monitor index**, so removing a monitor from the middle of the arrangement shifts every
later assignment up one slot. The refreshed list stays in memory and only reaches the companion asset on the next
`SaveCameras` call.

`LoadCameras` and `SaveCameras` skip the saved-views asset I/O entirely when the active scene has no name (untitled /
unsaved buffer). That guards against creating an orphan `-savedFullScreenViews.asset` (hyphen-prefixed, no scene to
consume it) when the Parameters window or the McpBridge writes camera mapping before the scene has been saved.
`SaveCameras` additionally no-ops when `monitors.Count == 0`, so a host where enumeration found no monitors (a headless
runner, or a macOS box without `displayplacer`) cannot wipe a scene's persisted camera assignments. The companion asset
is also cascade-deleted when its owning scene is removed via `delete_task_tool` (see `/task-prefabs`).

**Monitor enumeration timeout.** `Monitor` is Editor-only (the whole file sits inside `#if UNITY_EDITOR`).
`Monitor.EnumerateMonitors` calls `xrandr` (Linux) or `displayplacer list` (macOS) as a subprocess, and
`/unity-mcp-environment-setup` owns which helper each platform needs and the order
`Monitor.ResolveDisplayPlacerPath` searches for the macOS executable. A host carrying none of them logs
`Monitor enumeration: failed to start 'displayplacer'. Install it with 'brew install displayplacer'.` and returns
an empty monitor list rather than throwing. The 5000ms `SubprocessTimeoutMilliseconds` budget is applied twice, once to
`process.WaitForExit` and again to the stdout read, and a process that overruns it is killed, logged as a warning, and
parsed from whatever it produced. On Windows the enumeration is a synchronous P/Invoke (`EnumDisplayMonitors`) and has
no timeout. Every detected monitor is then probed for its `EditorGUIUtility.pixelsPerPoint` via a temporary 20x20 popup
`MonitorTester` window, which produces a brief visual flicker on each display.

**One-camera-per-monitor invariant.** `RenderMonitorRow` silently ignores any selection that would alias another
monitor's camera. The dropdown change appears to apply but the underlying `cameraEntityId` is not reassigned. The
persistence call is gated independently on the dropdown selection having changed, so `SaveCameras` still runs and, on a
saved scene, rewrites the saved-views asset with the unchanged bindings. To bind a camera that is already assigned
elsewhere, first set the original monitor's dropdown back to `None`, then assign the new monitor.
