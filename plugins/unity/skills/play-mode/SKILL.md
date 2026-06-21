---
name: play-mode
description: >-
  Controls Unity Editor Play Mode for sollertia-virtual-reality via the sollertia-shared-assets
  MCP server's Unity relay. Owns enter_play_mode_tool, exit_play_mode_tool, and
  get_play_state_tool. Use when manually exercising a task prefab, checking play state, or
  gating Unity operations on it.
user-invocable: false
---

# Sollertia Unity Play Mode

Drives the Unity Editor's Play Mode for the `sollertia-virtual-reality` project using the Unity relay
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
- Opening or creating scenes (see `/task-scenes`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

Play Mode here is an **Editor-side** convenience for developers iterating on a task prefab. It is
not the path production data acquisition takes.

---

## Play-state values

The `state` field appears in every response from these tools. `get_play_state_tool` is the
authoritative read and uses the first three values. The transition tools return a transient
value when they actually issue the transition and the matching steady-state value when they
short-circuit (already in the target state). `get_play_state_tool` also returns `active_scene`,
which is the active scene's *name* (not its asset path).

| State                | Returned by                                           | Meaning                                                                            |
|----------------------|-------------------------------------------------------|------------------------------------------------------------------------------------|
| `edit`               | `get_play_state_tool`, `exit_play_mode_tool` (no-op)  | Editor is in the default edit mode; scene is mutable, no runtime loop              |
| `compiling`          | `get_play_state_tool`                                 | Editor is recompiling scripts                                                      |
| `playing`            | `get_play_state_tool`, `enter_play_mode_tool` (no-op) | Editor is currently in Play Mode; the runtime loop is executing                    |
| `entering_play_mode` | `enter_play_mode_tool`                                | Transition was issued; poll `get_play_state_tool` to confirm it reaches `playing`  |
| `exiting_play_mode`  | `exit_play_mode_tool`                                 | Transition was issued; poll `get_play_state_tool` to confirm it reaches `edit`     |

You MUST call `get_play_state_tool` before calling `enter_play_mode_tool` or `exit_play_mode_tool`
to avoid issuing redundant transitions.

---

## MCP tool surface

| Tool                   | Purpose                                                                     | Response                                                                                                |
|------------------------|-----------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `enter_play_mode_tool` | Starts the Editor's Play Mode (exclusive)                                   | `{state, message}` — `entering_play_mode` on transition, `playing` if already in mode                   |
| `exit_play_mode_tool`  | Stops the Editor's Play Mode (exclusive)                                    | `{state, message}` — `exiting_play_mode` on transition, `edit` if already in edit mode                  |
| `get_play_state_tool`  | Returns current state and active scene (exclusive — may be a natural share) | `{state, active_scene}` — `state` is `edit` / `compiling` / `playing`; `active_scene` is the scene name |

`get_play_state_tool` is read-only and other skills may call it as a natural share when they need
to confirm the Editor is in `edit` before performing mutating Unity operations.

---

## Workflows

### Exercise a task prefab in Play Mode

1. **Verify prerequisites:**
   - Unity Editor running with McpBridge reachable (else `/unity-mcp-environment-setup`).
   - The target scene is open — hand off to `/task-scenes` (`open_scene_tool`) if not.
2. **Read current state:**
   ```text
   get_play_state_tool()
   ```
   If `state == "playing"`, skip to step 4.
   If `state == "compiling"`, wait and re-poll. You MUST NOT attempt to enter Play Mode during
   compilation.
3. **Enter Play Mode:**
   ```text
   enter_play_mode_tool()
   ```
   The response's `state` is `entering_play_mode` when the bridge has issued the transition (or
   `playing` if the editor was already in Play Mode). The tool does not block on completion —
   poll `get_play_state_tool` to confirm `playing` before exercising the scene.
4. **Ask the user to exercise the task:** The Editor's input system is how the developer drives
   the scene. Claude cannot control animal movement from the MCP layer.
5. **Exit Play Mode:**
   ```text
   exit_play_mode_tool()
   ```
6. **Verify the state reset:**
   ```text
   get_play_state_tool()
   ```
   You MUST confirm `state == "edit"` before handing off to other Unity workflows.

### Gate a mutating operation on edit mode

Scene changes and prefab regeneration are unsafe while the Editor is `playing`. You MUST check
the state before issuing them:

```text
state = get_play_state_tool()
if state["state"] != "edit":
    exit_play_mode_tool()
```

You MUST hand off to the owning skill (`/task-scenes`, `/task-prefabs`) only after the Editor returns
to `edit`.

---

## Interaction contract

- **You SHOULD NOT** call `enter_play_mode_tool` while `state == "compiling"`. The bridge does
  not gate this — `EnterPlayMode` (`McpBridge.cs:686-704`) only guards on
  `EditorApplication.isPlaying` and forwards the call to `EditorApplication.EnterPlaymode`,
  which Unity then defers until compilation finishes. The transition is queued rather than
  ambiguous, but the deterministic result is not visible until a follow-up `get_play_state_tool`
  poll.
- **Calling a transition tool in its target state is safe.** `enter_play_mode_tool` against
  `playing` (or `exit_play_mode_tool` against `edit`) short-circuits with the canonical
  steady-state `state` and an `Already in Play Mode.` / `Not in Play Mode.` message. You SHOULD
  still avoid these calls to keep the transcript focused.
- **You MUST** confirm the final state with `get_play_state_tool` after issuing a transition.
  The transition tools acknowledge the request (`entering_play_mode` / `exiting_play_mode`) but
  do not block on completion, and Unity refuses Play Mode entry when the active scene has
  compile errors — in that case `get_play_state_tool` keeps reporting `edit`.
- **Play Mode is Editor-scoped.** It does not interact with `sle run`, `sle manage`, or the
  acquisition rig. Data generated during Play Mode is not recorded anywhere.
- **Task Parameters re-opens on Play Mode entry.** `MainWindow.RegisterAutoOpen` registers an
  `EditorApplication.playModeStateChanged` hook that calls `EnsureWindowOpen` when the editor
  reaches `PlayModeStateChange.EnteredPlayMode`. The MQTT section, the Task section, and the
  Camera Mapping `Show Full Screen Views` control are disabled at runtime; you SHOULD flip the
  Task flags via MQTT (`/mqtt-contract`) instead of `/task-parameters` during a Play Mode run.

---

## Troubleshooting

| Symptom                                                                          | Cause                                                                         | Resolution                                     |
|----------------------------------------------------------------------------------|-------------------------------------------------------------------------------|------------------------------------------------|
| `enter_play_mode_tool` returns `entering_play_mode` but follow-up poll is `edit` | Unity refused the transition because the active scene has compile errors      | Ask the user to fix errors in the Console      |
| `enter_play_mode_tool` returns `entering_play_mode` while `compiling`            | Script recompile in progress; Unity will run the transition once it finishes  | Wait and re-poll `get_play_state_tool`         |
| `exit_play_mode_tool` returns `state == "edit"` immediately                      | Editor already in `edit`; the handler short-circuits with `Not in Play Mode.` | Expected — no further action needed            |
| Active scene is not the one expected                                             | A different scene was opened previously                                       | Hand off to `/task-scenes` (`open_scene_tool`) |
| All tools fail "Unity Editor is not reachable"                                   | McpBridge down                                                                | `/unity-mcp-environment-setup`                 |

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

| Skill                                         | Relationship                                                                                        |
|-----------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin)  | Run first if Unity Editor is unreachable                                                            |
| `/task-scenes` (this plugin)                  | Upstream — opens the scene to exercise in Play Mode                                                 |
| `/scene-setup` (this plugin)                  | Upstream — must pass the pre-Play Mode checklist first                                              |
| `/task-prefabs` (this plugin)                 | Upstream — generates the prefab under test                                                          |
| `/task-parameters` (this plugin)              | Upstream — set Actor / Task / Display fields in edit mode before entering Play Mode                 |
| `/mqtt-contract` (this plugin)                | Reference for topics that drive runtime behavior and runtime alternatives to Task Parameters writes |
| `assets:assets-mcp-environment-setup` | Upstream — owns the slsa MCP server diagnostic                                                      |
