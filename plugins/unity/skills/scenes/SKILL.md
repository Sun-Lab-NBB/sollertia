---
name: scenes
description: >-
  Manages Unity scenes and asset enumeration for sollertia-unity-tasks via the
  sollertia-shared-assets MCP server's Unity relay. Owns list_scenes_tool, open_scene_tool,
  create_scene_tool, and list_unity_assets_tool. Use when listing or switching scenes,
  creating a scene from a task prefab, or enumerating project assets.
user-invocable: true
---

# Sollertia Unity scenes

Manages Unity scenes and enumerates Unity assets for the `sollertia-unity-tasks` project using the
Unity relay exposed by `slsa mcp`. This skill is the **exclusive** owner of `list_scenes_tool`,
`open_scene_tool`, `create_scene_tool`, and `list_unity_assets_tool` — no other skill in the
marketplace may call these.

---

## Scope

**Covers:**
- Listing every Unity scene in the project and identifying the active one
- Opening an existing scene in the Editor
- Creating a new scene from the `ExperimentTemplate.unity` base, optionally seeded with a task prefab
- Enumerating Unity assets by type (Prefab, Scene, Material, Texture2D, …) under a search path

**Does not cover:**
- Generating or validating task prefabs (see `/task-prefabs`)
- Entering / exiting Play Mode (see `/play-mode`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)
- Authoring YAML task templates or experiment configurations (see assets plugin)

---

## MCP tool surface

| Tool                       | Purpose                                                               |
|----------------------------|-----------------------------------------------------------------------|
| `list_scenes_tool`         | Lists every scene in the project and flags the active one (exclusive) |
| `open_scene_tool`          | Opens a scene in the Editor (exclusive)                               |
| `create_scene_tool`        | Creates a scene from `ExperimentTemplate.unity` (exclusive)           |
| `list_unity_assets_tool`   | Lists assets of a given type under a path (exclusive)                 |

`list_unity_assets_tool` is callable as a natural share by `/task-prefabs` when enumerating prefabs
before inspection.

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
   Unsaved changes in the currently open scene are saved automatically before switching.

### Create a new scene

New scenes are created by copying `Assets/Scenes/ExperimentTemplate.unity` — the template defines
the standard camera rig, lighting, and audio setup used by every Sollertia task.

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
   empty scene with only the template's base GameObjects.
3. **Verify:**
   ```text
   list_scenes_tool()
   ```
   Confirm the new scene appears in the list.
4. **Open it:**
   ```text
   open_scene_tool(scene_path="Assets/Scenes/<scene-name>.unity")
   ```

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

## Path conventions

All tools require **project-relative** paths starting with `Assets/`. Absolute filesystem paths are
rejected by the Unity AssetDatabase.

| Asset                  | Canonical location                                  |
|------------------------|-----------------------------------------------------|
| Scenes                 | `Assets/Scenes/<name>.unity`                        |
| Task prefabs           | `Assets/InfiniteCorridorTask/Tasks/<name>.prefab`   |
| Segment prefabs        | `Assets/InfiniteCorridorTask/Prefabs/Segment_*.prefab` |
| Task template YAMLs    | `Assets/InfiniteCorridorTask/Configurations/<name>.yaml` |
| Scene base template    | `Assets/Scenes/ExperimentTemplate.unity`            |

---

## Troubleshooting

| Symptom                                             | Cause                                 | Resolution                                      |
|-----------------------------------------------------|---------------------------------------|-------------------------------------------------|
| `open_scene_tool` returns "scene not found"         | Scene path typo or missing file       | Call `list_scenes_tool` and copy the exact path |
| `create_scene_tool` fails with "template missing"   | `ExperimentTemplate.unity` absent     | Reinstall the `sollertia-unity-tasks` project   |
| `create_scene_tool` fails with "prefab not found"   | `task_prefab_path` is wrong           | Generate via `/task-prefabs` first; use project-relative path |
| `list_unity_assets_tool` returns empty list         | `asset_type` or `search_path` wrong   | Broaden `search_path="Assets"` and confirm type |
| Unity relay tools all fail                          | McpBridge down                        | `/unity-mcp-environment-setup`                        |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] list_scenes_tool returned a non-empty list before switching
- [ ] open_scene_tool resolved and the active scene changed
- [ ] create_scene_tool returned the expected scene_path under Assets/Scenes/
- [ ] list_unity_assets_tool returned the expected asset_type and count
```

---

## Related skills

| Skill                                         | Relationship                                              |
|-----------------------------------------------|-----------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                  |
| `/task-prefabs` (this plugin)                 | Upstream — generates the prefab seeded into a new scene   |
| `/scene-setup` (this plugin)                  | Consumer — configures the scene for runtime after opening |
| `/play-mode` (this plugin)                    | Consumer — typically entered after opening a target scene |
| assets plugin `/task-templates`               | Upstream — template filename defines the conventional scene name |
