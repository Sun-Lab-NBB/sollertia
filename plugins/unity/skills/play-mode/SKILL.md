---
name: play-mode
description: >-
  Controls Unity Editor Play Mode for the sollertia-unity-tasks project via the slsa MCP server's
  Unity relay. Owns enter_play_mode_tool, exit_play_mode_tool, and get_play_state_tool. Use when
  manually exercising a task prefab inside the Editor, checking whether the Editor is currently
  playing, or gating other Unity operations on the Editor's play state.
user-invocable: true
---

# Sollertia Unity Play Mode

Drives the Unity Editor's Play Mode for the `sollertia-unity-tasks` project using the Unity relay
exposed by `slsa mcp`. This skill is the **exclusive** owner of `enter_play_mode_tool`,
`exit_play_mode_tool`, and `get_play_state_tool` — no other skill in the marketplace may call
these.

---

## Scope

**Covers:**
- Entering Play Mode (`enter_play_mode_tool`)
- Exiting Play Mode (`exit_play_mode_tool`)
- Reading the current play state and active scene (`get_play_state_tool`)
- Gating other Unity operations on the Editor's play state

**Does not cover:**
- Runtime data acquisition outside the Editor — that is driven by the `sle run` CLI in
  `sollertia-experiment`, not by the Editor
- Generating or inspecting prefabs (see `/task-prefabs`)
- Opening or creating scenes (see `/scenes`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

Play Mode here is an **Editor-side** convenience for developers iterating on a task prefab. It is
not the path production data acquisition takes.

---

## Play-state values

`get_play_state_tool` returns one of three states in the `state` field:

| State       | Meaning                                                               |
|-------------|-----------------------------------------------------------------------|
| `edit`      | Editor is in the default edit mode; scene is mutable, no runtime loop |
| `compiling` | Editor is recompiling scripts; Play Mode cannot be entered right now  |
| `playing`   | Editor is currently in Play Mode; the runtime loop is executing       |

You MUST call `get_play_state_tool` before calling `enter_play_mode_tool` or `exit_play_mode_tool`
to avoid issuing redundant or no-op transitions.

---

## MCP tool surface

| Tool                     | Purpose                                                                         |
|--------------------------|---------------------------------------------------------------------------------|
| `enter_play_mode_tool`   | Starts the Editor's Play Mode (exclusive)                                       |
| `exit_play_mode_tool`    | Stops the Editor's Play Mode (exclusive)                                        |
| `get_play_state_tool`    | Returns current `state` and `active_scene` (exclusive — may be a natural share) |

`get_play_state_tool` is read-only and other skills may call it as a natural share when they need
to confirm the Editor is in `edit` before performing mutating Unity operations.

---

## Workflows

### Exercise a task prefab in Play Mode

1. **Verify prerequisites:**
   - Unity Editor running with McpBridge reachable (else `/unity-mcp-environment-setup`).
   - The target scene is open — hand off to `/scenes` (`open_scene_tool`) if not.
2. **Read current state:**
   ```text
   get_play_state_tool()
   ```
   If `state == "playing"`, skip to step 4.
   If `state == "compiling"`, wait and re-poll — do **not** attempt to enter Play Mode during
   compilation.
3. **Enter Play Mode:**
   ```text
   enter_play_mode_tool()
   ```
4. **Ask the user to exercise the task** — the Editor's input system is how the developer drives
   the scene. Claude cannot control animal movement from the MCP layer.
5. **Exit Play Mode:**
   ```text
   exit_play_mode_tool()
   ```
6. **Verify the state reset:**
   ```text
   get_play_state_tool()
   ```
   Confirm `state == "edit"` before handing off to other Unity workflows.

### Gate a mutating operation on edit mode

Some Unity operations (scene changes, prefab regeneration) should not run while the Editor is
playing. Before calling them, check the state:

```text
state = get_play_state_tool()
if state["state"] != "edit":
    exit_play_mode_tool()
```

Hand off to the owning skill (`/scenes`, `/task-prefabs`) only after the Editor returns to `edit`.

---

## Interaction contract

- **Never** call `enter_play_mode_tool` while `state == "compiling"`. The transition will either
  fail or be silently deferred, leaving the system in an ambiguous state.
- **Never** call `enter_play_mode_tool` when already `playing`, or `exit_play_mode_tool` when
  already `edit` — these are no-ops but pollute the transcript.
- Always confirm the final state with `get_play_state_tool` after a transition — Play Mode entry
  can fail silently if the active scene has compile errors.
- Play Mode is **Editor-scoped**. It does not interact with `sle run`, `sle manage`, or the
  acquisition rig. Data generated during Play Mode is not recorded anywhere.

---

## Troubleshooting

| Symptom                                        | Cause                                           | Resolution                                 |
|------------------------------------------------|-------------------------------------------------|--------------------------------------------|
| `enter_play_mode_tool` returns but state stays `edit` | Scene has compile errors                 | Ask the user to fix errors in the Console  |
| `enter_play_mode_tool` returns while `compiling`     | Script recompile in progress              | Wait and re-poll `get_play_state_tool`     |
| `exit_play_mode_tool` has no effect            | Editor already in `edit`                        | Expected — re-poll to confirm              |
| Active scene is not the one expected           | A different scene was opened previously         | Hand off to `/scenes` (`open_scene_tool`)  |
| All tools fail "Unity Editor is not reachable" | McpBridge down                                  | `/unity-mcp-environment-setup`                   |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] get_play_state_tool was called before enter_play_mode_tool / exit_play_mode_tool
- [ ] The target scene was open before entering Play Mode
- [ ] State returned to "edit" after exit_play_mode_tool
- [ ] No mutating Unity operation was issued while state was "playing" or "compiling"
```

---

## Related skills

| Skill                                  | Relationship                                      |
|----------------------------------------|---------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable          |
| `/scenes`                              | Upstream — opens the scene to exercise in Play Mode |
| `/task-prefabs`                        | Upstream — generates the prefab under test        |
| assets plugin `/assets-mcp-environment-setup` | Upstream — owns the slsa MCP server diagnostic    |
