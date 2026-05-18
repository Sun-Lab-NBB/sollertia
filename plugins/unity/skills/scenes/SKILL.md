---
name: scenes
description: >-
  Manages Unity scenes and asset enumeration for sollertia-unity-tasks via the
  sollertia-shared-assets MCP server's Unity relay. Owns list_scenes_tool, open_scene_tool,
  create_scene_tool, list_unity_assets_tool, and inspect_scene_tool. Use when listing or switching
  scenes, creating a scene from a task prefab, inspecting the active scene's hierarchy, or
  enumerating project assets.
user-invocable: true
---

# Sollertia Unity scenes

Manages Unity scenes and enumerates Unity assets for the `sollertia-unity-tasks` project using the
Unity relay exposed by `slsa mcp`. This skill is the **exclusive** owner of `list_scenes_tool`,
`open_scene_tool`, `create_scene_tool`, `list_unity_assets_tool`, and `inspect_scene_tool` — no
other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Listing every Unity scene in the project and identifying the active one
- Opening an existing scene in the Editor with explicit handling of unsaved edits
- Creating a new scene from the `ExperimentTemplate.unity` base, optionally seeded with a task prefab
- Inspecting the active scene's root GameObjects, components, and dirty state for pre-flight verification
- Enumerating Unity assets by type (Prefab, Scene, Material, Texture2D, …) under a search path

**Does not cover:**
- Generating or validating task prefabs (see `/task-prefabs`)
- Entering / exiting Play Mode (see `/play-mode`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)
- Authoring YAML task templates or experiment configurations (see assets plugin)

---

## MCP tool surface

| Tool                       | Purpose                                                                                       |
|----------------------------|-----------------------------------------------------------------------------------------------|
| `list_scenes_tool`         | Lists every scene in the project and flags the active one (exclusive)                         |
| `open_scene_tool`          | Opens a scene in the Editor with explicit unsaved-changes handling (exclusive)                |
| `create_scene_tool`        | Creates a scene from `ExperimentTemplate.unity` with explicit unsaved-changes handling (exclusive) |
| `inspect_scene_tool`       | Returns the active scene's metadata, dirty flag, and recursive root hierarchy (exclusive)     |
| `list_unity_assets_tool`   | Lists assets of a given type under a path (exclusive)                                         |

`list_unity_assets_tool` is callable as a natural share by `/task-prefabs` when enumerating prefabs
before inspection. `delete_unity_asset_tool` (owned by `/task-prefabs`) is callable here as a
natural share for scene deletion under `Assets/Scenes/`.

---

## Workflows

### Enumerate and switch scenes

1. **Verify prerequisites:** Unity Editor running with McpBridge reachable (else
   `/unity-mcp-environment-setup`).
2. **List scenes:**
   ```text
   list_scenes_tool()
   ```
   Returns `scenes` (list of `Assets/…/*.unity` paths), `active_scene`, and `count`.
3. **Open the target scene:**
   ```text
   open_scene_tool(scene_path="Assets/Scenes/<name>.unity")
   ```
   When the active scene is clean, the call switches scenes immediately. When the active scene has
   unsaved edits, the call returns an error — see [Unsaved-changes policy](#unsaved-changes-policy).

### Create a new scene

New scenes are created by copying `Assets/Scenes/ExperimentTemplate.unity` — the template defines
the standard camera rig, lighting, and audio setup used by every Sollertia task. The bridge
delegates to `CreateTask.CreateSceneFromTemplate`, which also runs `MainWindow.EnsureControllers`
to guarantee both `Linear` and `Simulated Linear` controllers exist in the new scene.

1. **(Optional) Ensure the task prefab is generated** — hand off to `/task-prefabs` to generate it
   before seeding the scene with it.
2. **Create the scene:**
   ```text
   create_scene_tool(
       scene_name="<scene-name>",
       task_prefab_path="Assets/InfiniteCorridorTask/Tasks/<template>.prefab"
   )
   ```
   The new scene is saved to `Assets/Scenes/<scene-name>.unity`. Omit `task_prefab_path` for an
   empty scene with only the template's base GameObjects. When the previously active scene has
   unsaved edits, the call returns an error — see [Unsaved-changes policy](#unsaved-changes-policy).

   Response shape: `{success, scene_path, message, simulated_controller_added, [warning]}`.
   - `simulated_controller_added` is `true` when `EnsureControllers` had to create the
     `Simulated Linear` GameObject and `false` when it was already present.
   - `warning: "task_prefab_not_found"` appears (and the call still succeeds) when a non-empty
     `task_prefab_path` was supplied but no asset could be loaded from it — the scene is created
     without the task hierarchy and the caller is expected to retry or surface the discrepancy.
   - The bridge **refuses** to clobber an existing scene at the requested path (returns an
     error). Delete the existing scene via `delete_unity_asset_tool` (`/task-prefabs`) under
     `Assets/Scenes/` first if you intend to overwrite.
3. **Verify:**
   ```text
   list_scenes_tool()
   ```
   Confirm the new scene appears in the list. The new scene is opened in the Editor automatically
   by `CreateSceneFromTemplate`, so a separate `open_scene_tool` call is only needed if the user
   navigates elsewhere first.

### Inspect the active scene

Use `inspect_scene_tool` to verify a scene is wired correctly before entering Play Mode. The tool
returns the active scene's path, name, dirty flag, root GameObject count, and a recursive
hierarchy with components and collider geometry for every root.

```text
inspect_scene_tool()
```

Common pre-flight checks:

- The `Task` component is present on a root GameObject (set up by `create_scene_tool` when a
  `task_prefab_path` is supplied).
- `ActorObject`, `MQTTClient`, the `Display` rig, and both `Linear` / `Simulated Linear`
  controller GameObjects from `MainWindow.InitializeScene` survived the scene copy. A regression
  here typically points to a hand-edited template scene.
- `is_dirty == false` before calling `enter_play_mode_tool`. A dirty scene means an earlier tool
  call modified the scene without saving.
- For programmatic field inspection (Actor / Display / MQTT / Camera Mapping / Task), prefer
  `/task-parameters` — it returns the same scene state plus the option lists and visibility flags
  that drive validation.

### Enumerate assets

```text
list_unity_assets_tool(
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

`open_scene_tool` and `create_scene_tool` both replace the active scene. When the active scene has
unsaved edits, the bridge refuses to proceed without an explicit policy and returns:

```text
Active scene '<path>' has unsaved changes. Specify unsaved_changes='save' to persist the current
scene before switching, or unsaved_changes='discard' to abandon the edits. Ask the user which
behavior they prefer before retrying.
```

You MUST ask the user for the choice when this error fires. Do not pick a default — discarding can
silently destroy work the user had open in the Editor.

| Value of `unsaved_changes` | Behavior                                                                                |
|----------------------------|-----------------------------------------------------------------------------------------|
| `"save"`                   | Calls `EditorSceneManager.SaveOpenScenes()` first, then switches                        |
| `"discard"`                | Switches immediately; unsaved edits are dropped silently                                |
| Omitted (`None`)           | Returns the dirty-scene error above when the scene is dirty; no-op when scene is clean  |

After collecting the user's choice, retry with the value:

```text
open_scene_tool(scene_path="Assets/Scenes/<name>.unity", unsaved_changes="save")
create_scene_tool(scene_name="<name>", unsaved_changes="discard")
```

`inspect_scene_tool` reports the active scene's `is_dirty` flag, so you can pre-emptively check
before invoking `open_scene_tool` / `create_scene_tool` and ask the user up-front when needed.

---

## Path conventions

All tools require **project-relative** paths starting with `Assets/`. Absolute filesystem paths are
rejected by the Unity AssetDatabase.

| Asset                  | Canonical location                                  |
|------------------------|-----------------------------------------------------|
| Scenes                 | `Assets/Scenes/<name>.unity`                        |
| Task prefabs           | `Assets/InfiniteCorridorTask/Tasks/<name>.prefab`   |
| Segment prefabs        | `Assets/InfiniteCorridorTask/Prefabs/<template>_<trial>.prefab` |
| Task template YAMLs    | `Assets/InfiniteCorridorTask/Configurations/<name>.yaml` |
| Scene base template    | `Assets/Scenes/ExperimentTemplate.unity`            |

---

## Troubleshooting

| Symptom                                                                         | Cause                                  | Resolution                                                                            |
|---------------------------------------------------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------|
| `open_scene_tool` returns "scene not found"                                     | Scene path typo or missing file        | Call `list_scenes_tool` and copy the exact path                                       |
| `open_scene_tool` / `create_scene_tool` returns "Active scene … has unsaved changes" | Active scene is dirty, no policy passed | Ask the user save vs discard, retry with `unsaved_changes="save"` or `"discard"`      |
| `create_scene_tool` fails with "Scene already exists at: …"                     | The target path is taken               | Delete the existing scene via `delete_unity_asset_tool` (`/task-prefabs`) under `Assets/Scenes/` first, then retry — the MCP path refuses overwrite to keep automated callers from silently destroying a hand-authored scene |
| `create_scene_tool` returns `warning: "task_prefab_not_found"`                  | A non-empty `task_prefab_path` did not resolve to a loadable prefab | Generate the prefab via `/task-prefabs`, or fix the path; the scene was created without the task hierarchy and can be re-seeded after generation |
| `create_scene_tool` fails with "Template scene not found"                       | `ExperimentTemplate.unity` absent      | Reinstall the `sollertia-unity-tasks` project                                         |
| `inspect_scene_tool` returns empty `root_objects`                                | No scene loaded, or `ExperimentTemplate.unity` was opened empty | Call `list_scenes_tool` and `open_scene_tool` to load a real scene first              |
| `list_unity_assets_tool` returns empty list                                     | `asset_type` or `search_path` wrong    | Broaden `search_path="Assets"` and confirm type                                       |
| Unity relay tools all fail                                                      | McpBridge down                         | `/unity-mcp-environment-setup`                                                        |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] list_scenes_tool returned a non-empty list before switching
- [ ] On dirty-scene error, asked the user save vs discard before retrying with unsaved_changes
- [ ] open_scene_tool resolved and the active scene changed
- [ ] create_scene_tool returned the expected scene_path under Assets/Scenes/
- [ ] inspect_scene_tool confirmed expected root components present before entering Play Mode
- [ ] list_unity_assets_tool returned the expected asset_type and count
```

---

## Related skills

| Skill                                         | Relationship                                              |
|-----------------------------------------------|-----------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                  |
| `/task-prefabs` (this plugin)                 | Upstream — generates the prefab seeded into a new scene   |
| `/scene-setup` (this plugin)                  | Consumer — configures the scene for runtime after opening |
| `/task-parameters` (this plugin)              | Consumer — reads / writes Actor / MQTT / Display / Camera Mapping / Task fields after opening |
| `/play-mode` (this plugin)                    | Consumer — typically entered after opening a target scene |
| assets plugin `/task-templates`               | Upstream — template filename defines the conventional scene name |
