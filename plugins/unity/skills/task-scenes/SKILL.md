---
name: task-scenes
description: >-
  Manages Unity task scenes and asset enumeration for sollertia-virtual-reality via the sollertia-shared-assets MCP
  server's Unity relay. Owns list_scenes_tool, open_scene_tool, save_scene_tool, inspect_scene_tool, list_assets_tool,
  and refresh_assets_tool. Use when listing, switching, saving, or inspecting task scenes, when enumerating project
  assets, or when importing files authored outside the Editor.
user-invocable: false
---

# Sollertia Unity task scenes

Lists, opens, saves, inspects, and enumerates Unity scenes and assets for the `sollertia-virtual-reality` project
through the Unity relay exposed by `slsa mcp`. This skill is the **exclusive** owner of `list_scenes_tool`,
`open_scene_tool`, `save_scene_tool`, `inspect_scene_tool`, and `refresh_assets_tool`. No other skill in the marketplace
may call the four scene tools or the asset importer. That exclusivity binds marketplace skills, not the acquisition
runtime: the host VR task driver resolves and opens the task scene over its own HTTP bridge client during a session (see
`experiment:vr-driver-interface`). This skill's asset enumeration tool (`list_assets_tool`) is read-only and may be
called as a **natural share** by any other skill that needs to enumerate project assets.

---

## Scope

**Covers:**
- Listing every Unity scene in the project and identifying the active one
- Opening an existing scene in the Editor with explicit handling of unsaved edits
- Saving the active scene back to its asset path to clear the dirty flag a play-mode preflight rejects
- Inspecting the active scene's root GameObjects, components, and dirty state for preflight verification
- Enumerating Unity assets by type (Prefab, Scene, Material, Texture2D, …) under a search path
- Importing files authored outside the Editor and reporting the compilation state the import produced

**Does not cover:**
- Task lifecycle, covering create and delete over the full template → prefab + segments + scene bundle (see
  `/task-prefabs`)
- Programmatic field reads / writes for Actor / MQTT / Display / Camera Mapping / Task (see `/task-parameters`)
- Entering / exiting Play Mode (see `/play-mode`)
- Editor-time scene wiring beyond what `inspect_scene_tool` reports (see `/scene-setup`)
- Unity Editor bridge diagnostics and Console reads (see `/unity-mcp-environment-setup`)

---

## MCP tool surface

| Tool                  | Purpose                                                                                  |
|-----------------------|------------------------------------------------------------------------------------------|
| `list_scenes_tool`    | Lists every scene path and returns the active-scene path as a separate field             |
| `open_scene_tool`     | Opens a scene in the Editor with explicit unsaved-changes handling                       |
| `save_scene_tool`     | Saves the active scene to its existing asset path and clears its dirty flag              |
| `inspect_scene_tool`  | Returns the active scene's metadata, dirty flag, and recursive root-GameObject hierarchy |
| `list_assets_tool`    | Lists asset paths of a given type under a search path (read-only natural share)          |
| `refresh_assets_tool` | Imports pending asset changes and reports the post-import compilation state              |

`list_assets_tool` is callable as a natural share by `/task-prefabs` when enumerating prefabs before inspection. For
task creation and deletion (which always operate on the full template → prefab + segments + scene bundle), hand off to
`/task-prefabs`.

Every tool on this surface returns a top-level `success` key: `true` alongside the payload fields documented below, or
`false` with an `error` message and nothing else. You MUST branch on `success` before reading any other field.

---

## Workflows

### Enumerate and switch scenes

1. **Verify prerequisites:** Unity Editor running with McpBridge reachable (else `/unity-mcp-environment-setup`), and
   the Editor in `edit`. Confirm with `get_play_state_tool` (`/play-mode`, a documented natural share) and exit Play
   Mode first, because `OpenScene` (`McpBridge.cs`) applies no play-state guard of its own.
2. **List scenes:**
   ```text
   list_scenes_tool()
   ```
   Returns `scenes` (list of `Assets/…/*.unity` paths) and `active_scene` (the currently open scene's path).
3. **Open the target scene:**
   ```text
   open_scene_tool(scene_path="Assets/Scenes/<name>.unity")
   ```
   When the active scene is clean, the call switches scenes immediately and returns `{success, message: "Opened scene:
   <path>", scene_path}`. Echo `scene_path` back to confirm the switch landed on the requested scene. When the active
   scene has unsaved edits, the call returns an error. See [Unsaved-changes policy](#unsaved-changes-policy).

### Save the active scene

`save_scene_tool` writes the active scene back to its existing asset path and clears the dirty flag that a
`write_task_parameters_tool` write sets. It takes no arguments and touches only the active scene.

```text
save_scene_tool()
```

Returns `{success, message: "Saved scene: <path>", scene_path, is_dirty}`. Two states are refused outright. Play Mode
returns `Cannot save the active scene while the Editor is in Play Mode.`, because the Editor discards scene edits on
exit, so call `exit_play_mode_tool` (`/play-mode`) first. A scene that has never been saved carries no asset path and is
refused rather than routed into a save dialog the bridge's caller cannot dismiss, so generate it through
`create_task_tool` (`/task-prefabs`) instead. A save Unity itself rejects returns `Unable to save the active scene to:
<path>`.

### Inspect the active scene

Use `inspect_scene_tool` to verify a scene is wired correctly before entering Play Mode.

```text
inspect_scene_tool()
```

Top-level response:

| Field          | Description                                                                            |
|----------------|----------------------------------------------------------------------------------------|
| `success`      | `true` on success, and a failed call returns `{success: false, error}` instead         |
| `scene_path`   | Project-relative path of the active scene (e.g. `Assets/Scenes/<name>.unity`)          |
| `scene_name`   | The active scene's filename without extension                                          |
| `is_dirty`     | `true` when the active scene has unsaved edits                                         |
| `root_objects` | List of recursive node objects (see below), one per root GameObject in hierarchy order |

Each entry in `root_objects`, and every descendant inside it, has this shape:

| Field                 | Always present?          | Description                                                         |
|-----------------------|--------------------------|---------------------------------------------------------------------|
| `name`                | yes                      | GameObject name                                                     |
| `active_self`         | yes                      | `GameObject.activeSelf`, the object's own enabled flag              |
| `position`            | yes                      | `transform.localPosition` (local, not world) as `{x, y, z}`         |
| `rotation`            | yes                      | `transform.localEulerAngles` (local, not world) as `{x, y, z}`      |
| `scale`               | yes                      | `transform.localScale` as `{x, y, z}`                               |
| `components`          | yes                      | List of component type names (null components are silently dropped) |
| `component_states`    | yes                      | List of `{type, enabled}` objects, one per entry in `components`    |
| `collider_center`     | only with `BoxCollider`  | `BoxCollider.center` as `{x, y, z}`                                 |
| `collider_size`       | only with `BoxCollider`  | `BoxCollider.size` as `{x, y, z}`                                   |
| `collider_is_trigger` | only with `BoxCollider`  | `BoxCollider.isTrigger` boolean                                     |
| `children`            | only when childCount > 0 | List of recursive node objects in transform-child order             |

Three consequences worth flagging:

- A GameObject that has a missing-script slot (broken `MonoBehaviour` GUID after a refactor) exposes a `null` component
  in Unity, and `inspect_scene_tool` silently drops it, so the symptom is "expected script not in `components`" rather
  than an explicit error.
- `component_states[i].enabled` is `null` for a component type Unity offers no enabled flag on, so a present-but-`null`
  entry reports "cannot be disabled" rather than "disabled". Branch on `null` before treating the value as a boolean.
- A node with no `BoxCollider` carries no `collider_*` keys, and a leaf node carries no `children` key. Always probe
  with key-presence checks rather than assuming empty values.

Common preflight checks:

- The `Task` component is present on a root GameObject (seeded by `create_task_tool` on `/task-prefabs`).
- `ActorObject`, `MQTTClient`, the `Display` rig, and the controller GameObjects seeded by `MainWindow.InitializeScene`
  survived the template copy. A regression here typically points to a hand-edited template scene.
- A generated scene holds no camera named `Main Camera` and none tagged `MainCamera`, because `CreateSceneFromTemplate`
  calls `MainWindow.RemoveDefaultMainCamera` after instantiating the task prefab. The `Display` rig owns the per-monitor
  cameras and `ActorObject` owns the tracking camera.
- `is_dirty == false` before calling `enter_play_mode_tool`. A dirty scene means an earlier tool call modified the scene
  without saving, and `save_scene_tool` clears it in one call.
- For programmatic field inspection (Actor / Display / MQTT / Camera Mapping / Task), prefer `/task-parameters`, which
  returns the same scene state plus the option lists and visibility flags that drive validation.

### Enumerate assets

```text
list_assets_tool(
    asset_type="Prefab",
    search_path="Assets/InfiniteCorridorTask"
)
```

Default `asset_type` is `Prefab` and default `search_path` is `Assets/InfiniteCorridorTask`. Other types the Unity
AssetDatabase understands: `Scene`, `Material`, `Texture2D`, `AudioClip`, `ScriptableObject`, etc.

Response shape: `{success, asset_type, search_path, assets}`, where `assets` is the alphabetically sorted list of
project-relative asset paths matching `t:<asset_type>` under `<search_path>`. An invalid `asset_type` or a search path
outside `Assets/` yields an empty `assets` list rather than an error.

Common uses:
- Audit every segment prefab under `Assets/InfiniteCorridorTask/Prefabs/`.
- Find all task prefabs under `Assets/InfiniteCorridorTask/Tasks/` before a batch regeneration.
- Enumerate materials or textures when debugging a rendering issue.

### Import assets authored outside the Editor

`refresh_assets_tool` is the agentic counterpart of the Editor's automatic refresh on focus. An Editor that nobody
focuses, such as one left open on an unattended rig, never receives that event, so a C# file written from outside stays
unimported and the type it declares stays unresolvable. Call it after authoring a script and before any tool that
references the new type, such as `clone_zone_prefab_tool` on `/zone-prefabs`.

```text
refresh_assets_tool()
```

Returns `{success, message, is_compiling, is_updating}`. A true `is_compiling` means a domain reload is in flight. A
false one does not prove the import produced no compilation, because the Editor may not have started one by the time the
handler reads the flag. Poll `get_play_state_tool` until it reports a state other than `compiling` before issuing
further calls. Compiler errors the import raises land in the Unity Console, which `read_console_tool`
(`/unity-mcp-environment-setup`, a natural share) reads with `level="error"`.

---

## Unsaved-changes policy

`open_scene_tool` is the only tool on this surface that gates on the active scene's dirty state. `create_task_tool` on
`/task-prefabs` applies the identical policy through the same bridge handler, because scene generation opens the newly
generated scene. When the active scene has unsaved edits and `unsaved_changes` is omitted, the bridge refuses to switch
scenes and returns:

```text
Active scene '<path>' has unsaved changes. Specify unsaved_changes='save' to persist the current
scene before switching, or unsaved_changes='discard' to abandon the edits. Ask the user which
behavior they prefer before retrying.
```

You MUST ask the user for the choice when this error fires. You MUST NOT pick a default, because discarding can silently
destroy work the user had open in the Editor.

| Value of `unsaved_changes` | Behavior                                                                                        |
|----------------------------|-------------------------------------------------------------------------------------------------|
| `"save"`                   | Calls `EditorSceneManager.SaveOpenScenes()` first when the active scene is dirty, then switches |
| `"discard"`                | Switches immediately, and unsaved edits are dropped silently                                    |
| Omitted (`None`)           | Returns the dirty-scene error above when the scene is dirty, and no-ops when the scene is clean |

The bridge consults `unsaved_changes` only when the **active** scene is dirty, and ignores the value entirely when the
active scene is clean. When the policy does run, `EditorSceneManager.SaveOpenScenes()` saves **every** open scene, not
just the active one, so in multi-scene Editor setups expect `unsaved_changes="save"` to write to every dirty scene file.
A clean active scene skips that save even when another open scene is dirty. Either way the switch calls
`EditorSceneManager.OpenScene(scene_path)` in the default `OpenSceneMode.Single`, which closes every other open scene,
dropping any edits the policy did not just write. `save_scene_tool` writes the active scene alone and leaves it clean,
so call it first to persist the active scene without writing the others, and warn the user that the switch discards
those other edits rather than preserving them.

After collecting the user's choice, retry with the value:

```text
open_scene_tool(scene_path="Assets/Scenes/<name>.unity", unsaved_changes="save")
```

`inspect_scene_tool` reports the active scene's `is_dirty` flag, so you can pre-emptively check before invoking
`open_scene_tool` and ask the user up-front when needed.

---

## Path conventions

All tools on this surface expect **project-relative** paths starting with `Assets/`. The underlying APIs enforce this
differently per tool:

- `list_assets_tool` uses `AssetDatabase.FindAssets`, which silently returns an empty list when the `search_path` is
  absolute or sits outside `Assets/`.
- `open_scene_tool` resolves `scene_path` through `AssetDatabase.LoadAssetAtPath<SceneAsset>` before opening it. The
  AssetDatabase resolves the project-relative path against the project root, so the answer is independent of the
  Editor's working directory, and typing the lookup to `SceneAsset` keeps a folder or a non-scene asset out of
  `EditorSceneManager.OpenScene`. An absolute path, a path outside `Assets/`, or a `.unity` file the AssetDatabase does
  not hold is rejected with `Scene not found at: <path>` rather than producing undefined behavior. Always pass a path
  copied from `list_scenes_tool`.
- `save_scene_tool` takes no path at all. It reuses the active scene's own `path`, so the destination is whatever
  `list_scenes_tool` reports as `active_scene`.

Task scenes live at `Assets/Scenes/<name>.unity`. `Assets/Scenes/ExperimentTemplate.unity` is the hand-authored base
scene the bridge copies for new task scenes (see `/task-prefabs` for the asset chain). Scene removal never travels
through `delete_asset_tool`, which refuses every `.unity` file under `Assets/Scenes/` outright, the protected template
included, with `Refusing to delete scene '<path>' via delete_asset. Use delete_task to remove a task's scene together
with its task prefab and segment prefabs in one atomic call.` Scenes are deleted exclusively through `delete_task_tool`,
which also cascade-deletes the per-scene `savedFullScreenViews` companion.

`delete_task_tool` in turn checks `McpBridge.DeleteProtectedPaths` for both the resolved scene path and the resolved
task-prefab path before deleting anything, so `template_name="ExperimentTemplate"` is refused with `Refusing to delete
task 'ExperimentTemplate'. Its scene or task prefab is a protected hand-authored asset that the generation pipeline
loads by hardcoded path.` The template scene cannot be destroyed through either delete path.

---

## Troubleshooting

| Symptom                                                        | Cause                                   | Resolution                                                                               |
|----------------------------------------------------------------|-----------------------------------------|------------------------------------------------------------------------------------------|
| `open_scene_tool` returns "Scene not found at: …"              | Path not a scene in the AssetDatabase   | Call `list_scenes_tool` and copy the exact path                                          |
| `open_scene_tool` returns "Active scene … has unsaved changes" | Active scene is dirty, no policy passed | Ask the user save vs discard, retry with `unsaved_changes="save"` or `"discard"`         |
| `save_scene_tool` returns "while the Editor is in Play Mode"   | Editor sits in `playing`                | `exit_play_mode_tool` (`/play-mode`), re-poll until `edit`, then retry                   |
| `save_scene_tool` returns "it has never been saved"            | Active scene has no asset path          | Generate the scene through `create_task_tool` (`/task-prefabs`)                          |
| `inspect_scene_tool` returns empty `root_objects`              | No scene loaded                         | Call `open_scene_tool`. `ExperimentTemplate.unity` is currently the project's only scene |
| `list_assets_tool` returns empty list                          | `asset_type` or `search_path` wrong     | Broaden `search_path="Assets"` and confirm type                                          |
| A newly authored script's type stays unresolvable              | Unfocused Editor imported nothing       | `refresh_assets_tool`, then poll `get_play_state_tool` out of `compiling`                |
| A tool fails without naming an Editor-side cause               | The detail was logged to the Console    | `read_console_tool(level="error")` (`/unity-mcp-environment-setup`)                      |
| Unity relay tools all fail                                     | McpBridge down                          | `/unity-mcp-environment-setup`                                                           |

---

## Related skills

| Skill                                        | Relationship                                                                                 |
|----------------------------------------------|----------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable, and owns `read_console_tool`                       |
| `/task-prefabs` (this plugin)                | Owns task lifecycle (`create_task_tool`, `delete_task_tool`) and the asset-chain reference   |
| `/scene-setup` (this plugin)                 | Consumer, configures the scene for runtime after opening                                     |
| `/task-parameters` (this plugin)             | Consumer, reads / writes Actor / MQTT / Display / Camera Mapping / Task fields after opening |
| `/play-mode` (this plugin)                   | Gate, `get_play_state_tool` confirms `edit` before a scene switch or a save                  |
| `/zone-prefabs` (this plugin)                | Consumer, needs `refresh_assets_tool` here before cloning against a newly authored script    |
| `assets:task-templates`                      | Upstream, template filename defines the conventional scene name                              |
| `/mqtt-contract` (this plugin)               | Active scene name is also exchanged over the `SceneName` / `SceneNameTrigger` wire pair      |
| `experiment:vr-driver-interface`             | Peer client, the host opens and verifies the task scene through these same bridge tools      |
| `assets:experiment-configuration`            | Owns `unity_scene_name`, which selects the scene to open                                     |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change that calls `list_scenes_tool`,
`open_scene_tool`, `save_scene_tool`, `inspect_scene_tool`, `list_assets_tool`, or `refresh_assets_tool`.

```text
Task Scenes Compliance:
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] get_play_state_tool reported "edit" before open_scene_tool or save_scene_tool was called
- [ ] list_scenes_tool returns a non-empty list before any switch
- [ ] On dirty-scene error from open_scene_tool, the user is asked save vs discard before retry
- [ ] open_scene_tool resolves and the active scene matches the requested path
- [ ] save_scene_tool cleared is_dirty, or inspect_scene_tool already reported the scene clean
- [ ] inspect_scene_tool confirms expected root components before entering Play Mode
- [ ] list_assets_tool returns the expected asset_type and matching paths
- [ ] refresh_assets_tool ran after any script authored outside the Editor, and compiling finished
- [ ] Scene create / delete operations are dispatched to /task-prefabs, not attempted here
```
