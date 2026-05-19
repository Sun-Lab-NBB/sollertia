---
name: task-scenes
description: >-
  Manages Unity task scenes and asset enumeration for sollertia-unity-tasks via the
  sollertia-shared-assets MCP server's Unity relay. Owns list_scenes_tool, open_scene_tool,
  inspect_scene_tool, and list_assets_tool. Use when listing, switching, or inspecting task
  scenes, or when enumerating project assets.
user-invocable: true
---

# Sollertia Unity task scenes

Lists, opens, inspects, and enumerates Unity scenes and assets for the `sollertia-unity-tasks`
project through the Unity relay exposed by `slsa mcp` — the **exclusive** owner of
`list_scenes_tool`, `open_scene_tool`, `inspect_scene_tool`, and `list_assets_tool`, which no
other skill in the marketplace may call.

---

## Scope

**Covers:**
- Listing every Unity scene in the project and identifying the active one
- Opening an existing scene in the Editor with explicit handling of unsaved edits
- Inspecting the active scene's root GameObjects, components, and dirty state for pre-flight verification
- Enumerating Unity assets by type (Prefab, Scene, Material, Texture2D, …) under a search path

**Does not cover:**
- Task lifecycle (create / delete the full template → prefab + segments + scene bundle) — see
  `/task-prefabs`
- Programmatic field reads / writes for Actor / MQTT / Display / Camera Mapping / Task — see
  `/task-parameters`
- Entering / exiting Play Mode — see `/play-mode`
- Editor-time scene wiring beyond what `inspect_scene_tool` reports — see `/scene-setup`
- Unity Editor bridge diagnostics — see `/unity-mcp-environment-setup`

---

## MCP tool surface

| Tool                 | Purpose                                                                                   |
|----------------------|-------------------------------------------------------------------------------------------|
| `list_scenes_tool`   | Lists every scene in the project and flags the active one (exclusive)                     |
| `open_scene_tool`    | Opens a scene in the Editor with explicit unsaved-changes handling (exclusive)            |
| `inspect_scene_tool` | Returns the active scene's metadata, dirty flag, and recursive root hierarchy (exclusive) |
| `list_assets_tool`   | Lists assets of a given type under a path (exclusive)                                     |

`list_assets_tool` is callable as a natural share by `/task-prefabs` when enumerating prefabs
before inspection. For task creation and deletion (which always operate on the full template →
prefab + segments + scene bundle), hand off to `/task-prefabs`.

---

## Workflows

### Enumerate and switch scenes

1. **Verify prerequisites:** Unity Editor running with McpBridge reachable (else
   `/unity-mcp-environment-setup`).
2. **List scenes:**
   ```text
   list_scenes_tool()
   ```
   Returns `scenes` (list of `Assets/…/*.unity` paths) and `active_scene` (the currently open
   scene's path).
3. **Open the target scene:**
   ```text
   open_scene_tool(scene_path="Assets/Scenes/<name>.unity")
   ```
   When the active scene is clean, the call switches scenes immediately. When the active scene has
   unsaved edits, the call returns an error — see [Unsaved-changes policy](#unsaved-changes-policy).

### Inspect the active scene

Use `inspect_scene_tool` to verify a scene is wired correctly before entering Play Mode. The tool
returns `scene_path`, `scene_name`, `is_dirty`, and `root_objects` — a recursive hierarchy with
the transform, components, and collider geometry of every root GameObject.

```text
inspect_scene_tool()
```

Common pre-flight checks:

- The `Task` component is present on a root GameObject (set up by `create_task_tool` on
  `/task-prefabs`).
- `ActorObject`, `MQTTClient`, the `Display` rig, and the controller GameObjects from
  `MainWindow.InitializeScene` survived the scene copy. A regression here typically points to a
  hand-edited template scene.
- `is_dirty == false` before calling `enter_play_mode_tool`. A dirty scene means an earlier tool
  call modified the scene without saving.
- For programmatic field inspection (Actor / Display / MQTT / Camera Mapping / Task), prefer
  `/task-parameters` — it returns the same scene state plus the option lists and visibility flags
  that drive validation.

### Enumerate assets

```text
list_assets_tool(
    asset_type="Prefab",
    search_path="Assets/InfiniteCorridorTask"
)
```

Default `asset_type` is `Prefab` and default `search_path` is `Assets/InfiniteCorridorTask`. Other
types the Unity AssetDatabase understands: `Scene`, `Material`, `Texture2D`, `AudioClip`,
`ScriptableObject`, etc.

Common uses:
- Audit every segment prefab under `Assets/InfiniteCorridorTask/Prefabs/`.
- Find all task prefabs under `Assets/InfiniteCorridorTask/Tasks/` before a batch regeneration.
- Enumerate materials or textures when debugging a rendering issue.

---

## Unsaved-changes policy

`open_scene_tool` is the only tool on this surface that gates on the active scene's dirty state.
When the active scene has unsaved edits and `unsaved_changes` is omitted, the bridge refuses to
switch scenes and returns:

```text
Active scene '<path>' has unsaved changes. Specify unsaved_changes='save' to persist the current
scene before switching, or unsaved_changes='discard' to abandon the edits. Ask the user which
behavior they prefer before retrying.
```

You MUST ask the user for the choice when this error fires. You MUST NOT pick a default —
discarding can silently destroy work the user had open in the Editor.

| Value of `unsaved_changes` | Behavior                                                                                |
|----------------------------|-----------------------------------------------------------------------------------------|
| `"save"`                   | Calls `EditorSceneManager.SaveOpenScenes()` first, then switches                        |
| `"discard"`                | Switches immediately; unsaved edits are dropped silently                                |
| Omitted (`None`)           | Returns the dirty-scene error above when the scene is dirty; no-op when scene is clean  |

After collecting the user's choice, retry with the value:

```text
open_scene_tool(scene_path="Assets/Scenes/<name>.unity", unsaved_changes="save")
```

`inspect_scene_tool` reports the active scene's `is_dirty` flag, so you can pre-emptively check
before invoking `open_scene_tool` and ask the user up-front when needed.

---

## Path conventions

All tools require **project-relative** paths starting with `Assets/`. Absolute filesystem paths
are rejected by the Unity AssetDatabase. Task scenes live at `Assets/Scenes/<name>.unity`;
`Assets/Scenes/ExperimentTemplate.unity` is the protected hand-authored base scene that the
bridge copies for new task scenes (see `/task-prefabs` for the asset chain).

---

## Troubleshooting

| Symptom                                                        | Cause                                                           | Resolution                                                                                            |
|----------------------------------------------------------------|-----------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `open_scene_tool` returns "Scene not found at: …"              | Scene path typo or missing file                                 | Call `list_scenes_tool` and copy the exact path                                                       |
| `open_scene_tool` returns "Active scene … has unsaved changes" | Active scene is dirty, no policy passed                         | Ask the user save vs discard, retry with `unsaved_changes="save"` or `"discard"`                      |
| `inspect_scene_tool` returns empty `root_objects`              | No scene loaded, or `ExperimentTemplate.unity` was opened empty | Call `list_scenes_tool` and `open_scene_tool` to load a real scene first                              |
| `list_assets_tool` returns empty list                          | `asset_type` or `search_path` wrong                             | Broaden `search_path="Assets"` and confirm type                                                       |
| Unity relay tools all fail                                     | McpBridge down                                                  | `/unity-mcp-environment-setup`                                                                        |

---

## Related skills

| Skill                                        | Relationship                                                                                  |
|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable                                                      |
| `/task-prefabs` (this plugin)                | Owns task lifecycle (`create_task_tool`, `delete_task_tool`) and the asset-chain reference    |
| `/scene-setup` (this plugin)                 | Consumer — configures the scene for runtime after opening                                     |
| `/task-parameters` (this plugin)             | Consumer — reads / writes Actor / MQTT / Display / Camera Mapping / Task fields after opening |
| `/play-mode` (this plugin)                   | Consumer — typically entered after opening a target scene                                     |
| assets plugin `/task-templates`              | Upstream — template filename defines the conventional scene name                              |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change that calls
`list_scenes_tool`, `open_scene_tool`, `inspect_scene_tool`, or `list_assets_tool`.

```text
Task Scenes Compliance:
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] list_scenes_tool returns a non-empty list before any switch
- [ ] On dirty-scene error from open_scene_tool, the user is asked save vs discard before retry
- [ ] open_scene_tool resolves and the active scene matches the requested path
- [ ] inspect_scene_tool confirms expected root components before entering Play Mode
- [ ] list_assets_tool returns the expected asset_type and matching paths
- [ ] Scene create / delete operations are dispatched to /task-prefabs, not attempted here
```
