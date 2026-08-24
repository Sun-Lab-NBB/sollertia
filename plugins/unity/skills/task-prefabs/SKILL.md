---
name: task-prefabs
description: >-
  Creates, deletes, and inspects Unity tasks for sollertia-virtual-reality from YAML task templates.
  Owns create_task_tool (single-step template → prefab + scene), delete_task_tool (single-step
  removal of every generated artifact for a task), inspect_prefab_tool, and delete_asset_tool
  (individual cue / material cleanup). Use when a template needs a matching task built or removed,
  or when auditing prefab hierarchy and colliders.
user-invocable: false
---

# Sollertia Unity task prefabs

Creates, deletes, and inspects Unity tasks for the `sollertia-virtual-reality` project through the
Unity relay exposed by `slsa mcp` — the **exclusive** owner of `create_task_tool`,
`delete_task_tool`, and `delete_asset_tool`, which no other skill in the marketplace may call. This
skill's inspection tool (`inspect_prefab_tool`) is read-only and may be called as a **natural
share** by any other skill that needs to read a prefab's hierarchy.

---

## Scope

**Covers:**
- Creating a Unity task end-to-end from a YAML task template — task prefab plus matching scene
  in one call (`create_task_tool`)
- Deleting a Unity task end-to-end — scene plus per-scene companion plus task prefab plus every
  segment prefab in one call (`delete_task_tool`)
- Inspecting a prefab's hierarchy, components, transforms, and colliders (`inspect_prefab_tool`)
- Deleting a regenerable cue prefab or cue material individually (`delete_asset_tool`)
- Template naming, header, and commenting conventions required for correct task creation

**Does not cover:**
- Authoring the YAML task template itself (see `assets:task-templates`)
- Authoring per-project experiment configurations (see `assets:experiment-configuration`)
- Listing, opening, or inspecting scenes (see `/task-scenes`)
- Entering / exiting Play Mode (see `/play-mode`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

---

## Asset chain on disk

A single task is three name-aligned files on disk. The base name (`MF_Reward`, `SSO_Reversal`, …)
is greppable across all three:

```text
Assets/InfiniteCorridorTask/Configurations/<name>.yaml      template          (authored by assets:task-templates)
                                │
                                │  create_task_tool (single call)
                                ▼
Assets/InfiniteCorridorTask/Tasks/<name>.prefab             task prefab
Assets/Scenes/<name>.unity                                  scene
```

`create_task_tool` is the **only** path that produces the prefab + scene pair, and the basename
convention is enforced by the tool itself: `template_name="MF_Reward"` produces
`Tasks/MF_Reward.prefab` and `Scenes/MF_Reward.unity` unconditionally. Both MCP `create_task_tool`
and the `CreateTask → New Task` Editor menu reject templates outside the `Configurations/`
directory so the cross-template cue-texture preflight, the runtime config-path resolver, and
downstream tooling all see a single canonical home — use `assets:working-directory`
(`set_task_templates_directory_tool`) to configure the MCP-side path.

Two derived artifact tiers sit below the task prefab:

- **Segment prefabs** under `Assets/InfiniteCorridorTask/Prefabs/<TemplateName>-<TrialName>.prefab` —
  one per template `trial_structures` entry, template-owned (every trial gets its own segment
  prefab even when two trials in different templates have identical geometry). The name is built by
  `CreateTask.CanonicalSegmentName`, and the separator is a **hyphen**: `ConfigLoader` restricts
  both the template name and the trial name to ASCII letters, digits, and underscores, so the
  hyphen appears exactly once and the filename splits back into one template and one trial.
- **Cue prefabs** under `Assets/InfiniteCorridorTask/Cues/Cue_<cuename>_<length>cm.prefab` —
  **shared** across every template that declares the same `(name, length_cm)` cue identity.

For the abstract template model (cue / segment / trial vocabulary, transition graph,
sliding-window corridor traversal), see `assets:task-templates`. For the
`CreateTask.CreateFromTemplate` pipeline internals (cue / segment build passes, shader chain,
zone placement math, hand-authored protected assets), see `/task-generator`.

**Scene scope.** `create_task_tool` produces a new **corridor** scene by copying `ExperimentTemplate.unity` — that is
the only scene topology this skill creates, and creating new corridor scenes this way is fully agent-autonomous. A
scene with a different Display rig or a non-corridor topology has no author-derived recipe; escalate to the human
supervisor (see `/scene-setup`). You MUST NOT hand-author a scene to work around this.

---

## MCP tool surface

| Tool                  | Purpose                                                                                                       |
|-----------------------|---------------------------------------------------------------------------------------------------------------|
| `create_task_tool`    | Builds a task prefab and the matching scene from a template in one call (exclusive, dirty-scene gated)        |
| `delete_task_tool`    | Removes the scene, the per-scene companion, the task prefab, and every segment prefab in one call (exclusive) |
| `inspect_prefab_tool` | Returns a prefab's recursive GameObject tree (read-only natural share)                                        |
| `delete_asset_tool`   | Removes a regenerable cue prefab or cue material (exclusive)                                                  |

`list_assets_tool` (owned by `/task-scenes`) is callable as a natural share when enumerating
prefabs before inspection. `inspect_prefab_tool` stays owned by this skill and is callable as a
natural share in the other direction. For example, `/zone-prefabs` runs it to validate a cloned
zone prefab. `delete_asset_tool` is scoped to individual assets under
`Assets/InfiniteCorridorTask/Tasks/`, `Prefabs/`, `Cues/`, and `Materials/` — templates under
`Configurations/` and imported textures under `Textures/` are outside the allowed roots; the handler
also rejects scene paths so scene cleanup goes through `delete_task_tool` and preserves the
per-scene `savedFullScreenViews` companion cascade.
`delete_task_tool` is the inverse of `create_task_tool`.

`create_task_tool` takes a second argument, `unsaved_changes` (`"save"` | `"discard"` | omitted).
Scene generation opens the new scene, which would discard unsaved edits in the active one, so the
bridge applies the same dirty-scene gate `open_scene_tool` applies — see `/task-scenes`
"Unsaved-changes policy". `delete_task_tool` takes no such argument.

---

## Template conventions (required for generation)

### File naming

```text
ProjectAbbreviation_TaskDescription.yaml
```

| Abbreviation | Project           |
|--------------|-------------------|
| MF           | MaalstroomicFlow  |
| SSO          | StateSpaceOdyssey |

- Use `_Base` suffix for single-segment training configurations.
- Capitalize each word in the task description.
- Template filename (without `.yaml`) is reused verbatim as the generated Unity scene / prefab name.

The `[A-Za-z0-9_]` character set is **hard-enforced**, not a convention. `ConfigLoader.LoadTemplate`
throws `Template filename '<name>' is invalid. Template names must contain only ASCII letters,
digits, and underscores…` for anything else, and the same restriction gates every cue name and
trial name. A hyphen in particular is forbidden, because it is the segment-prefab separator.

### Template header

Every template must begin with a YAML comment header:

```yaml
# Project: [Full project name]
# Purpose: [Single sentence describing the task structure]
# Layout:  [Segment names with cue letters and zone placements]
# History: [Optional — provenance of a reproduced historical geometry]
# Related: [Related template file (parenthetical explanation)]
```

Multi-line values align continuation text with the first character after the field name:

```yaml
# Project: MaalstroomicFlow
# Purpose: Extends the base MF_Reward trial structure to also include an aversive stimulus.
# Layout:  Segment ABCD with occupancy zone at cue C and aversive stimulus trigger zone in cue D.
#          Segment EFGH with the rewarding stimulus trigger zone in cue H.
# Related: MF_Reward (the base version of this task that only includes the reward zone)
```

Header guidelines:
- **Project:** Full project name, not the abbreviation.
- **Purpose:** Single sentence starting with a verb (Defines, Extends, Teaches).
- **Layout:** Segment names with cue letters, zone types, and stimulus clarifications.
- **History:** Optional. Present only on templates that reproduce a historical geometry, where it
  records the provenance and the exact numeric values the reproduction targets.
- **Related:** Parenthetical explanation of the relationship.

Stimulus modality is an acquisition-system concern, not a Unity one — the Unity wire contract is
valence-agnostic (`StimulusMessage` carries only the trial name, whether the stimulus was delivered,
and the cause), so keep template headers to the modality-free `rewarding stimulus` / `aversive
stimulus` wording and leave the concrete hardware mapping to the acquisition system's own skills.

### Inline comments

Add inline YAML comments to clarify non-obvious values:

```yaml
cues:
  - name: "Gray"  # Placeholder — task does not use Gray cues.
    code: 0
    length_cm: 1.0
    texture: "Gray Cue 2x1.png"
```

`texture` is mandatory on every cue, placeholder cues included: `ConfigLoader` rejects a missing or
empty value and additionally requires the named file to exist under
`Assets/InfiniteCorridorTask/Textures/`.

Templates that do not follow the **soft** conventions above — the abbreviation registry, word
capitalization, the `_Base` suffix, the header block, and the inline comments — may still generate
prefabs, but downstream tooling (session preprocessing, dataset forging) relies on the naming
contract. The `[A-Za-z0-9_]` character set is not among them; it is enforced at load.

---

## Generation workflow

### Step 1: Verify prerequisites

- `slsa mcp` server connected (else `assets:assets-mcp-environment-setup`).
- Unity Editor running with McpBridge listening (else `/unity-mcp-environment-setup` in this plugin).
- Template exists under `Assets/InfiniteCorridorTask/Configurations/<template-name>.yaml`. If not,
  hand off to `assets:task-templates` to author it first.

### Step 2: Create the task

```text
create_task_tool(template_name="<template-name>", unsaved_changes="save"|"discard")
```

A single call builds the task prefab at
`Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab` and the matching scene at
`Assets/Scenes/<template-name>.unity`. Both paths are auto-resolved from the template basename
and cannot be overridden — every artifact of one task is greppable by one name. The tool
delegates to Unity's CreateTask pipeline (cue prefabs, segment prefabs, corridor hierarchy)
followed by `CreateSceneFromTemplate` (copies `ExperimentTemplate.unity`, instantiates the new
prefab, runs `EnsureControllers` / `EnsureMqttDefaults` / `SyncDisplayBrightnessToSettings`).

`unsaved_changes` is optional but is **not** a value to guess. Generation opens the new scene, which
discards unsaved edits in the active one, so when the active scene is dirty and the argument is
omitted the bridge errors before any asset is written. You MUST ask the user whether to save or
discard, then retry with their answer.

The success response carries `template_name`, `prefab_path`, `scene_path`,
`simulated_controller_added` (true when `EnsureControllers` created a `SimulatedLinearTreadmill` the
copied template scene did not already carry — the shipped `ExperimentTemplate.unity` carries none, so
a fresh generation reports true; false means one was already present and `/scene-setup` has nothing
to add), and `message`.

If a scene already exists at the resolved path, the tool refuses and points at `delete_task_tool`
(this skill); a regeneration cycle is always `delete_task_tool` → `create_task_tool`.

### Step 3: Inspect the result

```text
inspect_prefab_tool(prefab_path="Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab")
```

Verify the hierarchy matches the template — cue count, segment order, trial zones.

### Step 4: Hand off

- The scene already exists at `Assets/Scenes/<template-name>.unity` — `create_task_tool` produced
  it in step 2. For navigation between scenes, hand off to `/task-scenes`.
- For runtime testing: hand off to `/play-mode`.
- For per-project experiment configuration: hand off to `assets:experiment-configuration`.

---

## Deleting a task

`delete_task_tool` is the inverse of `create_task_tool`. A single call removes every Unity
artifact that `create_task_tool` produced for a template:

```text
delete_task_tool(template_name="<template-name>")
```

Removes the scene at `Assets/Scenes/<template-name>.unity` (plus its
`savedFullScreenViews.asset` companion via the same cascade `delete_task_tool` uses), the task
prefab at `Assets/InfiniteCorridorTask/Tasks/<template-name>.prefab`, and every segment prefab
under `Assets/InfiniteCorridorTask/Prefabs/` that `<template-name>` owns — the sweep matches the
literal prefix `Assets/InfiniteCorridorTask/Prefabs/<template-name>-`, and because `ConfigLoader`
bars hyphens from both the template basename and the trial name, that prefix can only match this
template's own segments even when one basename nests another (e.g. `SSO_Merging` vs
`SSO_Merging_Base`). Sweeping by prefix rather than by current trial name also reclaims orphans left
by trials since removed from the template. The template YAML and the shared cue prefabs / materials
are preserved — cues live in `Assets/InfiniteCorridorTask/Cues/` and are referenced by sibling
tasks, so individual cue cleanup goes through `delete_asset_tool`.

The response carries `template_name`, `deleted` (always
`true` on success), `deleted_paths` (the scene, the task prefab, and every segment prefab removed —
the per-scene companion is **not** among them), `message`, and two conditional keys:
`companion_deleted` (the saved-views asset, present only when one was removed) and
`companion_delete_failed` (present only when the cascade could not remove an existing companion).
`companion_delete_failed` still accompanies a successful delete, so check for it: the owning scene
is already gone and the orphaned companion under `Assets/VRSettings/Displays/` has to be removed by
hand. When no artifacts exist for the supplied template the call returns
`No artifacts found for template '<name>'.`

**Protected-asset refusal.** The tool checks `McpBridge.DeleteProtectedPaths` for both the resolved
scene path and the resolved task-prefab path before deleting anything, so a template name that
collides with a hand-authored asset is refused rather than honored:
`delete_task_tool(template_name="ExperimentTemplate")` returns `Refusing to delete task
'ExperimentTemplate'. Its scene or task prefab is a protected hand-authored asset that the
generation pipeline loads by hardcoded path.`

**Active-scene swap.** When the scene being deleted is the Editor's active scene, the tool opens
`Assets/Scenes/ExperimentTemplate.unity` in single mode first, so the Editor is left on the
template scene once the delete finishes. The swap is conditional on that scene being the open one,
and the response reports nothing about it. You MUST re-open the scene you intend to work in through
`/task-scenes` before any skill that reads the active scene runs, including `/task-parameters`,
`/play-mode`, and `/task-scenes` itself.

Unlike `create_task_tool` and `open_scene_tool`, `delete_task_tool` takes no `unsaved_changes`
argument and does not consult the unsaved-changes policy: the swap opens the template scene
unconditionally, so unsaved edits in the scene being deleted are discarded without a prompt. Save or
deliberately abandon them before calling it.

---

## Regenerating after template edits

The regeneration cycle is always **`delete_task_tool` → `create_task_tool`**: `create_task_tool`
refuses to overwrite an existing scene, so the agent removes the existing bundle (scene + per-scene
companion + task prefab + every segment prefab) before rebuilding. `create_task_tool` always
rebuilds the **task prefab** and every **segment prefab** the template owns regardless of which
files survived the deletion. The Unity-side `CreateTask` pipeline deletes the previous
`<template-name>-<trial-name>.prefab` files before regenerating them, so trial-parameter edits in the YAML
(cue sequence, zone math, trigger type) take effect on the next call without any manual cleanup
beyond the bundle delete. That wipe runs **after** the cue build pass, so a cue failure aborts with
the previous generation still intact on disk.

**Cue prefabs and materials are different.** They are keyed by cue name and length only
(`Cue_<name>_<length>cm.prefab` / `.mat`) and are shared across every template that declares a
matching cue, so `CreateTask` reuses them instead of rebuilding them per template. Editing a cue's
**texture** while keeping the same name and length therefore cannot propagate on its own — but it no
longer drifts silently either. `BuildCuePrefabs` loads the cached `Cue_<name>_<length>cm.mat` and
compares its `_MainTex` against the template's declared texture; on a mismatch it logs
`BuildCuePrefabs: Cue '<name>' at <n> cm declares texture '<file>', but the cached material
'<stem>.mat' was built from a different texture…` and returns false, so the call fails with
`Failed to build cue prefabs.` before any segment is wiped. Delete the cue prefab and its
material and regenerate, or give the cue a distinct name or length so it occupies its own asset
slot. (The separate cross-template cue-texture preflight covers the other case: two templates
declaring the same `(name, length_cm)` identity with conflicting textures.)

The skip that reuses a cue requires **both** its prefab and its material to be on disk. Deleting the
material alone is enough to force a rebuild — `BuildCuePrefabs` retires the surviving prefab and
writes both assets afresh, so the wall renderers point at the new material.

### Picking what to delete

| Template change                                                      | Delete                                                                                                                          |
|----------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `cues[].texture` for cue X (name and length unchanged)               | `Assets/InfiniteCorridorTask/Cues/Cue_X_<length>cm.prefab` **and** `Assets/InfiniteCorridorTask/Materials/Cue_X_<length>cm.mat` |
| `cues[].length_cm` for cue X                                         | Nothing — a new length yields a new `Cue_X_<new-length>cm.prefab` automatically                                                 |
| `trial_structures[T].cue_sequence` for trial T                       | Nothing — the segment prefab is regenerated automatically                                                                       |
| `trial_structures[T]` zone math for trial T                          | Nothing — the segment prefab is regenerated automatically                                                                       |
| `vr_environment.cm_per_unity_unit` (rescaled units affect every cue) | All cue prefabs *and* materials under `Cues/` / `Materials/` whose lengths are affected                                         |

The task prefab and the scene are rebuilt by every successful `create_task_tool` call.

### Workflow

1. **Remove the existing task bundle** (always required — `create_task_tool` refuses to overwrite
   an existing scene):
   ```text
   delete_task_tool(template_name="<template-name>")
   ```
   Cue prefabs and materials are deliberately preserved.
2. **Delete stale cue assets** (only when cue textures or shared cue geometry changed):
   ```text
   delete_asset_tool(asset_path="Assets/InfiniteCorridorTask/Cues/Cue_X_30cm.prefab")
   delete_asset_tool(asset_path="Assets/InfiniteCorridorTask/Materials/Cue_X_30cm.mat")
   ```
   Allowed roots and protected paths are summarized in
   [Troubleshooting](#troubleshooting). Because cue assets are shared across templates,
   deleting them forces every dependent template to regenerate the cue on its next
   `create_task_tool` call.
3. **Regenerate** — re-run `create_task_tool(template_name="<template-name>")`. The pipeline
   rebuilds the prefab, every segment prefab from scratch, any missing cue prefabs, and the scene.
4. **Spot-check** — run `inspect_prefab_tool` against the rebuilt task prefab to confirm the
   hierarchy matches the template (cue count, segment order, trial zones).

---

## End-to-end task authoring workflow

Use this composite flow for end-to-end task creation; each step is owned by a different skill.

| Step | Skill (owner)                     | Action                                                                                                |
|------|-----------------------------------|-------------------------------------------------------------------------------------------------------|
| 1    | `assets:task-templates`           | Author `Assets/InfiniteCorridorTask/Configurations/<name>.yaml`                                       |
| 2    | `/task-prefabs` (this skill)      | `create_task_tool(template_name="<name>", unsaved_changes="save"\|"discard")` — task prefab AND scene |
| 3    | `/task-prefabs` (this skill)      | `inspect_prefab_tool(prefab_path="Assets/InfiniteCorridorTask/Tasks/<name>.prefab")`                  |
| 4    | `/task-scenes`                    | `open_scene_tool(scene_path="Assets/Scenes/<name>.unity")` — the scene was created in step 2          |
| 5    | `/scene-setup`                    | Configure Display rig and optional `SimulatedLinearTreadmill`                                         |
| 6    | `/play-mode`                      | `enter_play_mode_tool()` → exercise → `exit_play_mode_tool()`                                         |
| 7    | `assets:experiment-configuration` | (Optional) Bind the template to a per-project experiment configuration                                |

Checkpoints between steps:

- **Step 2 → 3:** stop if `create_task_tool` returns `success: false`. Common causes are a mistyped
  `template_name`, a malformed YAML header, or an existing scene at the resolved path (regenerate via
  `delete_task_tool` → `create_task_tool`). An `unsaved changes` error means the active scene is
  dirty — ask the user save-vs-discard and retry with `unsaved_changes`, never pick for them.
- **Step 3 → 4:** stop if `inspect_prefab_tool` shows a hierarchy that contradicts the template (missing
  corridor count, wrong segment names). Fix the template and regenerate via the
  `delete_task_tool` → `create_task_tool` cycle; you MUST NOT hand-patch the prefab.
- **Step 5 → 6:** stop if a display panel is missing — Play Mode without displays throws runtime
  null-reference errors in `ActorObject.Display`.

---

## Reading inspect_prefab_tool output

`inspect_prefab_tool` returns the same recursive node tree that `inspect_scene_tool` does (each
node carries `name`, `position`, `rotation`, `scale`, `components`, optional `collider_*` keys,
optional `children`); the canonical shape contract — and the warning about silently-dropped
missing scripts and key-presence checks — lives in `/task-scenes` "Inspect the active scene".
This section covers only the **task-prefab-specific** interpretation: the top-level object is the
task prefab, its children are `Corridor<indices>` objects, and each first segment's stimulus zone
hierarchy varies by `trigger_type`.

The tool reports component **type names** and collider geometry only — it never returns a
serialized MonoBehaviour field value, so neither `triggerMode` nor `showBoundary` can be read back
from its output. Judge a generated zone by the root GameObject's name, its child shape, and its
`collider_size.z`, then confirm the intended mode from the template's `trigger_type`.

The five `trigger_type` modes (`interaction`, `collision`, `occupancy_disarm`, `occupancy_arm`,
`occupancy_trigger`), their per-mode hierarchy trees, the collider math behind each annotation, and
the key markers that separate a healthy prefab from a miswired one live in
[references/generated-prefab-anatomy.md](references/generated-prefab-anatomy.md). Read it whenever
an inspection result has to be judged against a template's `trigger_type`, or when a segment's zone
children look wrong.

---

## Troubleshooting

| Symptom                                                                                | Cause                                                                                                                                                                               | Resolution                                                                                                                                      |
|----------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| `create_task_tool` returns "Template not found"                                        | Template file missing from `Configurations/`                                                                                                                                        | Hand off to `assets:task-templates`                                                                                                             |
| `create_task_tool` returns "Template filename '…' is invalid"                          | The basename carries something outside `[A-Za-z0-9_]`; `ConfigLoader` rejects it at load                                                                                            | Rename the template file; hand off to `assets:task-templates`                                                                                   |
| `create_task_tool` returns "Scene already exists at: …"                                | The target scene exists; regeneration is an explicit two-step action                                                                                                                | Call `delete_task_tool` for the existing scene, then re-run `create_task_tool`                                                                  |
| `create_task_tool` returns "Active scene '…' has unsaved changes"                      | Generation opens the new scene, which would discard unsaved edits in the active one                                                                                                 | Ask the user save-vs-discard, then retry with `unsaved_changes="save"` or `"discard"`                                                           |
| `create_task_tool` returns "Cross-template cue-texture conflict detected"              | Two templates declare the same `(cue name, length_cm)` identity with different `texture` values                                                                                     | Rename, re-length, or unify the colliding cue via `assets:task-templates`, then re-run                                                          |
| `create_task_tool` returns "Failed to build cue prefabs."                              | `BuildCuePrefabs` aborted — the Console names a missing texture or a cached material built from another texture                                                                     | Import the missing texture, or delete the cue's `.prefab` **and** `.mat` (see "Regenerating after template edits")                              |
| `create_task_tool` returns "Generation requires hand-authored assets that are missing" | `ValidateHandAuthoredAssets` found a deleted hand-authored prefab or material                                                                                                       | Restore the named paths from git; the list lives in `/task-generator` "Required shared assets"                                                  |
| `create_task_tool` returns a track-length error naming the longest segment             | `ValidateTrackLengthCoversCorridor` — the longest segment does not fit `segments_per_corridor` times                                                                                | Shorten the longest cue sequence via `assets:task-templates`, or raise Track Length via `/task-parameters`                                      |
| `delete_task_tool` returns "Refusing to delete task '…'"                               | The resolved scene or task prefab is in `McpBridge.DeleteProtectedPaths`                                                                                                            | Nothing to fix — confirm the intended template name; you MUST NOT bypass the protection                                                         |
| `delete_task_tool` returns "No artifacts found for template '…'"                       | No scene, task prefab, or segment prefab exists for that basename                                                                                                                   | Check the spelling against `Configurations/`; the task may already be deleted                                                                   |
| `delete_task_tool` succeeds but returns `companion_delete_failed`                      | The scene was removed but its `savedFullScreenViews` companion was not, so it is orphaned                                                                                           | Remove the named asset under `Assets/VRSettings/Displays/` by hand                                                                              |
| `inspect_prefab_tool` returns "Prefab not found at: …"                                 | Prefab missing from `Tasks/` (deleted, or the generating call errored)                                                                                                              | Re-run `create_task_tool`; if it still does not appear, check the Console for `CreateTask` errors                                               |
| All Unity tools return "Unable to reach the Unity Editor at http://localhost:8090/ …"  | Nothing is listening: the Editor is closed or McpBridge is not loaded                                                                                                               | `/unity-mcp-environment-setup` in this plugin                                                                                                   |
| All Unity tools return "Unable to complete the request to the Unity Editor at …"       | The Editor accepted the connection but did not answer within 30 s — its main thread is busy                                                                                         | Wait for the Editor to become responsive and retry; this is not an environment fault                                                            |
| Generated zone hierarchy does not match the template's `trigger_type`                  | The template was edited after generation, or the prefab was hand-patched                                                                                                            | Regenerate via `delete_task_tool` → `create_task_tool`; you MUST NOT hand-patch the prefab                                                      |
| `delete_asset_tool` rejects the path with "Refusing to delete"                         | Path is outside the four allowed `InfiniteCorridorTask` roots, names a protected asset from `/task-generator` "Required shared assets", or contains `..`, is rooted, or ends in `/` | Reference a different name in the template; you MUST NOT bypass the protection. Restore the protected asset from git if it is genuinely missing |

---

## Related skills

| Skill                                        | Relationship                                                          |
|----------------------------------------------|-----------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable                              |
| `/task-scenes` (this plugin)                 | Consumer — opens / inspects the scene this skill produced             |
| `/play-mode` (this plugin)                   | Consumer — exercises the prefab at runtime                            |
| `/scene-setup` (this plugin)                 | Consumer — configures displays / controller before Play Mode          |
| `/task-parameters` (this plugin)             | Consumer — reads / writes the generated `Task` component fields       |
| `/zone-prefabs` (this plugin)                | Natural-share caller of `inspect_prefab_tool` for cloned zone prefabs |
| `/task-generator` (this plugin)              | Reference for the `CreateTask` pipeline this tool invokes             |
| `/mqtt-contract` (this plugin)               | Reference for MQTT topics wired by generated zone scripts             |
| `/gimbl-framework` (this plugin)             | Reference for `ActorObject` coordinate frame usage                    |
| `assets:task-templates`                      | Upstream — owns the YAML template the prefab is built from            |
| `assets:experiment-configuration`            | Downstream — per-project instantiation of the template                |
| `assets:assets-mcp-environment-setup`        | Run first — owns the slsa MCP server diagnostic                       |
| `experiment:vr-driver-interface`             | Host consumes the cues and zones in the generated prefab at runtime   |

---

## Verification checklist

You MUST verify your work against this checklist before submitting any change that calls
`create_task_tool`, `delete_task_tool`, `inspect_prefab_tool`, or `delete_asset_tool`.

```text
Task Prefabs Compliance:
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] slsa mcp server is connected
- [ ] Target template exists under Assets/InfiniteCorridorTask/Configurations/
- [ ] Template filename follows the ProjectAbbreviation_TaskDescription convention and uses only
      ASCII letters, digits, and underscores
- [ ] When the active scene is dirty, the user was asked save-vs-discard and create_task_tool was
      called with the unsaved_changes value they chose
- [ ] create_task_tool returns prefab_path and scene_path on success, and simulated_controller_added
      was read to confirm a simulated controller is present before keyboard testing
- [ ] inspect_prefab_tool hierarchy matches the template's cue / segment / trial counts
- [ ] Stale cue prefabs and materials are removed via delete_asset_tool before regeneration when
      a cue texture changes without a name or length rename
- [ ] delete_task_tool responses are checked for companion_delete_failed, and any orphaned
      savedFullScreenViews asset it names is removed by hand
- [ ] When delete_task_tool removed the scene that was open in the Editor, the intended scene is
      re-opened through /task-scenes before any skill reads the active scene
- [ ] Every cues[].texture in the template resolves to an existing file under
      Assets/InfiniteCorridorTask/Textures/; a missing texture is handed off to the user to supply (you cannot author
      binary image assets) and generation resumes only after they import it — never left to fail with "Failed to load
      texture"
- [ ] Generated prefabs are not hand-edited; regeneration goes through delete_task_tool →
      create_task_tool
```
