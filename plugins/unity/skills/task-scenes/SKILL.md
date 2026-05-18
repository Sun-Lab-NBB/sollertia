---
name: task-scenes
description: >-
  Manages Unity task scenes and asset enumeration for sollertia-unity-tasks via the
  sollertia-shared-assets MCP server's Unity relay. Owns list_scenes_tool, open_scene_tool,
  delete_task_tool, list_assets_tool, and inspect_scene_tool. Use when listing, switching,
  inspecting, or deleting task scenes, or when enumerating project assets. Task creation and
  deletion (template → prefab + scene) are owned by /task-prefabs (create_task_tool,
  delete_task_tool).
user-invocable: true
---

# Sollertia Unity task scenes

Manages Unity task scenes and enumerates Unity assets for the `sollertia-unity-tasks` project
using the Unity relay exposed by `slsa mcp`. This skill is the **exclusive** owner of
`list_scenes_tool`, `open_scene_tool`, `delete_task_tool`, `list_assets_tool`, and
`inspect_scene_tool` — no other skill in the marketplace may call these. Task creation and
deletion are owned by `/task-prefabs` (`create_task_tool`, `delete_task_tool`), which produce or
remove the prefab and its matching scene together.

## Task vs scene

In this project, the two terms are **not interchangeable** but they are 1:1 by construction:

- A **task** is the abstract behavior defined by a YAML template under
  `Assets/InfiniteCorridorTask/Configurations/<name>.yaml` plus the generated artifacts that
  materialize it (the task prefab and the segment prefabs).
- A **task scene** is the Unity scene file at `Assets/Scenes/<name>.unity` that wraps one task
  prefab instance inside the runtime infrastructure (`Actors`, `Controllers`, `MQTT Client`,
  default `Actor` + `Display`) so the task can be exercised in the Editor or Play Mode.

Every scene under `Assets/Scenes/` is a task scene of this form — the only exception is the
hand-authored `Assets/Scenes/ExperimentTemplate.unity`, which is the base template the bridge
copies from when building new task scenes. Each task owns exactly one scene; each scene exercises
exactly one task. Both share the template basename (`MF_Reward.yaml` → `MF_Reward.prefab` →
`MF_Reward.unity`), so "the MF_Reward task" and "the MF_Reward scene" refer to the same task
end-to-end. This skill operates on the **scene** side of that pair; `/task-prefabs` owns the
task-level lifecycle (create / delete the whole bundle).

---

## Scope

**Covers:**
- Listing every Unity scene in the project and identifying the active one
- Opening an existing scene in the Editor with explicit handling of unsaved edits
- Deleting a scene under `Assets/Scenes/` together with its per-scene companion in one call
  (`delete_task_tool`)
- Inspecting the active scene's root GameObjects, components, and dirty state for pre-flight verification
- Enumerating Unity assets by type (Prefab, Scene, Material, Texture2D, …) under a search path

**Does not cover:**
- Creating scenes from templates — owned by `/task-prefabs` (`create_task_tool`)
- Generating or validating task prefabs (see `/task-prefabs`)
- Entering / exiting Play Mode (see `/play-mode`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)
- Authoring YAML task templates or experiment configurations (see assets plugin)

---

## The task-scene chain

A task scene is the **third tier** of the Sollertia task asset chain:

```text
Assets/InfiniteCorridorTask/Configurations/<name>.yaml      template (abstract YAML description)
                                │
                                │  /task-prefabs (create_task_tool — single call)
                                ▼
Assets/InfiniteCorridorTask/Tasks/<name>.prefab             task prefab (runtime corridor hierarchy)
Assets/Scenes/<name>.unity                                  task scene (this skill)
```

All three artifacts share the **same basename** by construction. `create_task_tool` auto-resolves
both the prefab path (`Tasks/<template-basename>.prefab`) and the scene path
(`Scenes/<template-basename>.unity`) from the supplied `template_name`, so a single name is
greppable end to end. The basename convention is enforced by the tool itself — agents cannot
choose a different scene name. See `/task-prefabs` for the canonical hierarchy description.

A task scene built by `create_task_tool` contains the task prefab instance (which transitively
brings in every segment-prefab and cue-prefab the task prefab references) plus the auto-created
infrastructure (`Actors`, `Controllers`, `MQTT Client`, default `Actor` + `Display`) seeded by
`MainWindow.InitializeScene`. The hand-authored `Assets/Scenes/ExperimentTemplate.unity` is the
base scene the bridge copies — it ships with the standard camera rig, lighting, and audio setup
and **you MUST NOT** edit it as a regular working scene. `delete_task_tool` also refuses to
delete `ExperimentTemplate.unity` for the same reason.

---

## MCP tool surface

| Tool                     | Purpose                                                                                                                 |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------|
| `list_scenes_tool`       | Lists every scene in the project and flags the active one (exclusive)                                                   |
| `open_scene_tool`        | Opens a scene in the Editor with explicit unsaved-changes handling (exclusive)                                          |
| `delete_task_tool`      | Deletes a scene under `Assets/Scenes/` and its `savedFullScreenViews` companion in one atomic call (exclusive)          |
| `inspect_scene_tool`     | Returns the active scene's metadata, dirty flag, and recursive root hierarchy (exclusive)                               |
| `list_assets_tool` | Lists assets of a given type under a path (exclusive)                                                                   |

`list_assets_tool` is callable as a natural share by `/task-prefabs` when enumerating
prefabs before inspection. Scene **creation** is owned by `/task-prefabs` (`create_task_tool`),
which builds the task prefab and the matching scene in one call from a template — there is no
standalone "create scene" tool. Scene **deletion** is owned by this skill (`delete_task_tool`);
`delete_asset_tool` (owned by `/task-prefabs`) rejects scene paths to keep the per-scene
companion cascade from being bypassed.

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

Scene creation is owned by `/task-prefabs` (`create_task_tool`). Hand off to that skill — a single
`create_task_tool(template_name="<name>")` call produces both the task prefab and the matching
scene at conventional paths. There is no standalone "create scene" tool on this surface.

### Delete a scene (and its per-scene companion)

`delete_task_tool` is the canonical path for removing a scene under `Assets/Scenes/`. The tool
inlines every step of scene cleanup so the caller does not have to coordinate them:

1. **Validate the path** — refuses paths outside `Assets/Scenes/` or paths that don't end in
   `.unity`. Refuses `ExperimentTemplate.unity` (the hand-authored base scene).
2. **Deactivate if active** — if the scene to delete is currently the active scene, the tool
   opens `ExperimentTemplate.unity` first so Unity will accept the delete (Unity refuses to
   delete the active scene). Any unsaved edits in the active scene are discarded — deletion is
   destructive by intent.
3. **Delete the `.unity` asset.**
4. **Cascade-delete the per-scene companion** at
   `Assets/VRSettings/Displays/<scene>-savedFullScreenViews.asset` when it exists, so per-scene
   camera mappings do not outlive the scene.
5. **Refresh the AssetDatabase.**

```text
delete_task_tool(scene_path="Assets/Scenes/<name>.unity")
# Response when a companion existed:
# {success: true, scene_path: "...", deleted: true,
#  companion_deleted: "Assets/VRSettings/Displays/<name>-savedFullScreenViews.asset"}
```

`delete_asset_tool` (owned by `/task-prefabs`) rejects scene paths and points at this tool —
scene cleanup must always pass through `delete_task_tool` so the companion cascade fires.

A scene regeneration cycle is **`delete_task_tool` → `create_task_tool`**: `create_task_tool`
refuses to overwrite an existing scene, so the agent removes the existing scene (and its
companion) here before rebuilding via `/task-prefabs`. When you want to remove the **entire**
task (scene + task prefab + every segment prefab), use `delete_task_tool` (owned by
`/task-prefabs`) instead — this skill's `delete_task_tool` only touches the scene asset and its
companion.

`Assets/Scenes/ExperimentTemplate.unity` is the protected base scene; `delete_task_tool` refuses
to delete it. New per-scene companion assets should be added to
`McpBridge.TryDeleteScenePerSceneCompanions` in the same change that introduces them.

### Inspect the active scene

Use `inspect_scene_tool` to verify a scene is wired correctly before entering Play Mode. The tool
returns the active scene's path, name, dirty flag, root GameObject count, and a recursive
hierarchy with components and collider geometry for every root.

```text
inspect_scene_tool()
```

Common pre-flight checks:

- The `Task` component is present on a root GameObject (set up by `create_task_tool` when a
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

`open_scene_tool` and `create_task_tool` both replace the active scene. When the active scene has
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
create_task_tool(scene_name="<name>", unsaved_changes="discard")
```

`inspect_scene_tool` reports the active scene's `is_dirty` flag, so you can pre-emptively check
before invoking `open_scene_tool` / `create_task_tool` and ask the user up-front when needed.

---

## Path conventions

All tools require **project-relative** paths starting with `Assets/`. Absolute filesystem paths are
rejected by the Unity AssetDatabase.

| Asset               | Canonical location                                              |
|---------------------|-----------------------------------------------------------------|
| Scenes              | `Assets/Scenes/<name>.unity`                                    |
| Task prefabs        | `Assets/InfiniteCorridorTask/Tasks/<name>.prefab`               |
| Segment prefabs     | `Assets/InfiniteCorridorTask/Prefabs/<template>_<trial>.prefab` |
| Task template YAMLs | `Assets/InfiniteCorridorTask/Configurations/<name>.yaml`        |
| Scene base template | `Assets/Scenes/ExperimentTemplate.unity`                        |

---

## Troubleshooting

| Symptom                                                                              | Cause                                                               | Resolution                                                                                                                                                                                                                   |
|--------------------------------------------------------------------------------------|---------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `open_scene_tool` returns "scene not found"                                          | Scene path typo or missing file                                     | Call `list_scenes_tool` and copy the exact path                                                                                                                                                                              |
| `open_scene_tool` / `create_task_tool` returns "Active scene … has unsaved changes" | Active scene is dirty, no policy passed                             | Ask the user save vs discard, retry with `unsaved_changes="save"` or `"discard"`                                                                                                                                             |
| `create_task_tool` fails with "Scene already exists at: …"                          | The target path is taken                                            | Delete the existing scene via `delete_asset_tool` (`/task-prefabs`) under `Assets/Scenes/` first, then retry — the MCP path refuses overwrite to keep automated callers from silently destroying a hand-authored scene |
| `create_task_tool` returns `warning: "task_prefab_not_found"`                       | A non-empty `task_prefab_path` did not resolve to a loadable prefab | Generate the prefab via `/task-prefabs`, or fix the path; the scene was created without the task hierarchy and can be re-seeded after generation                                                                             |
| `create_task_tool` fails with "Template scene not found"                            | `ExperimentTemplate.unity` absent                                   | Reinstall the `sollertia-unity-tasks` project                                                                                                                                                                                |
| `inspect_scene_tool` returns empty `root_objects`                                    | No scene loaded, or `ExperimentTemplate.unity` was opened empty     | Call `list_scenes_tool` and `open_scene_tool` to load a real scene first                                                                                                                                                     |
| `list_assets_tool` returns empty list                                          | `asset_type` or `search_path` wrong                                 | Broaden `search_path="Assets"` and confirm type                                                                                                                                                                              |
| Unity relay tools all fail                                                           | McpBridge down                                                      | `/unity-mcp-environment-setup`                                                                                                                                                                                               |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] list_scenes_tool returned a non-empty list before switching
- [ ] On dirty-scene error, asked the user save vs discard before retrying with unsaved_changes
- [ ] open_scene_tool resolved and the active scene changed
- [ ] create_task_tool returned the expected scene_path under Assets/Scenes/
- [ ] inspect_scene_tool confirmed expected root components present before entering Play Mode
- [ ] list_assets_tool returned the expected asset_type and count
```

---

## Related skills

| Skill                                        | Relationship                                                                                  |
|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable                                                      |
| `/task-prefabs` (this plugin)                | Upstream — generates the prefab seeded into a new scene                                       |
| `/scene-setup` (this plugin)                 | Consumer — configures the scene for runtime after opening                                     |
| `/task-parameters` (this plugin)             | Consumer — reads / writes Actor / MQTT / Display / Camera Mapping / Task fields after opening |
| `/play-mode` (this plugin)                   | Consumer — typically entered after opening a target scene                                     |
| assets plugin `/task-templates`              | Upstream — template filename defines the conventional scene name                              |
