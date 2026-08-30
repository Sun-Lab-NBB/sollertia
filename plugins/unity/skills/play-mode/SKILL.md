---
name: play-mode
description: >-
  Controls Unity Editor Play Mode for sollertia-virtual-reality via the sollertia-shared-assets MCP server's Unity
  relay. Owns enter_play_mode_tool, exit_play_mode_tool, and get_play_state_tool. Use when manually exercising a task
  prefab, checking play state, or gating Unity operations on it.
user-invocable: false
---

# Sollertia Unity Play Mode

Drives the Unity Editor's Play Mode for the `sollertia-virtual-reality` project using the Unity relay exposed by `slsa
mcp`. This skill is the **exclusive** owner of `enter_play_mode_tool`, `exit_play_mode_tool`, and `get_play_state_tool`.
No other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Entering Play Mode (`enter_play_mode_tool`)
- Exiting Play Mode (`exit_play_mode_tool`)
- Reading the current play state and active scene (`get_play_state_tool`)
- Gating other Unity operations on the Editor's play state

**Does not cover:**
- The acquisition-runtime side of Play Mode. `sollertia-experiment`'s `VRTaskDriver` owns the runtime enter/exit
  sequence and its handshake (see `experiment:vr-driver-interface`). The platform-general runtime pattern inside which
  it sits is `experiment:acquisition-system-runtime`, and each acquisition system's own CLI is owned by that system's
  plugin
- The automated Play Mode test suite. `Assets/Tests/PlayMode/` (assembly `Sollertia.Tests.PlayMode`) runs under the Test
  Runner or headlessly via `Unity -batchmode -nographics -projectPath . -runTests -testPlatform PlayMode`. The Test
  Runner drives its own Play Mode transitions and MUST NOT be interleaved with these tools (see `/unity-tests`)
- Generating or inspecting prefabs (see `/task-prefabs`)
- Opening or creating scenes (see `/task-scenes`)
- Unity Editor bridge diagnostics (see `/unity-mcp-environment-setup`)

Play Mode is **not** Editor-only. The acquisition runtime drives the exact same three bridge tools:
`VRTaskDriver._arm_unity` calls `enter_play_mode` and waits for the scene's `SessionStart` broadcast, `_stop_unity`
calls `exit_play_mode`, and the pre-session display check `_verify_vr_display` calls `get_play_state` (all three in
`sollertia-experiment/src/sollertia_experiment/vr_task/driver.py`). This skill covers the **interactive** use of those
tools. The runtime-owned path is documented by `experiment:vr-driver-interface`.

---

## Play-state values

The `state` field appears in every **successful** response from these tools (`success == true`). A failed call, whether
the bridge was unreachable, the request timed out, or the payload was unparseable, returns only `{"success": false,
"error": "..."}` with no `state` key, so you MUST check `success` before reading `state`. `get_play_state_tool` is the
authoritative read and uses the first three values. The transition tools return a transient value when they actually
issue the transition and the matching steady-state value when they short-circuit (already in the target state).
`get_play_state_tool` also returns `active_scene`, which is the active scene's *name* (not its asset path).

`get_play_state_tool` cannot represent an in-flight transition. `GetPlayState` (`McpBridge.cs`) derives the value from
`EditorApplication.isPlaying` and `EditorApplication.isCompiling` alone, never from `isPlayingOrWillChangePlaymode`, so
a poll taken while Unity is still tearing the domain down for Play Mode entry answers `edit`.

| State                | Returned by                                           | Meaning                                                                              |
|----------------------|-------------------------------------------------------|--------------------------------------------------------------------------------------|
| `edit`               | `get_play_state_tool`, `exit_play_mode_tool` (no-op)  | Editor is in the default edit mode, the scene is mutable, and no runtime loop runs   |
| `compiling`          | `get_play_state_tool`                                 | Editor is recompiling scripts                                                        |
| `playing`            | `get_play_state_tool`, `enter_play_mode_tool` (no-op) | Editor is currently in Play Mode, and the runtime loop is executing                  |
| `entering_play_mode` | `enter_play_mode_tool`                                | Transition was issued, so poll `get_play_state_tool` to confirm it reaches `playing` |
| `exiting_play_mode`  | `exit_play_mode_tool`                                 | Transition was issued, so poll `get_play_state_tool` to confirm it reaches `edit`    |

You MUST call `get_play_state_tool` before calling `enter_play_mode_tool` or `exit_play_mode_tool` to avoid issuing
redundant transitions.

---

## MCP tool surface

| Tool                   | Purpose                                                                 | Response                                                                                                                 |
|------------------------|-------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `enter_play_mode_tool` | Starts the Editor's Play Mode (exclusive)                               | `{success, state, message}`, with `entering_play_mode` on transition and `playing` if already in mode                    |
| `exit_play_mode_tool`  | Stops the Editor's Play Mode (exclusive)                                | `{success, state, message}`, with `exiting_play_mode` on transition and `edit` if already in edit mode                   |
| `get_play_state_tool`  | Returns current state and active scene (exclusive, and a natural share) | `{success, state, active_scene}`, where `state` is `edit` / `compiling` / `playing` and `active_scene` is the scene name |

`get_play_state_tool` is read-only and other skills may call it as a natural share when they need to confirm the Editor
is in `edit` before performing mutating Unity operations.

---

## Workflows

### Exercise a task prefab in Play Mode

1. **Verify prerequisites:**
   - Unity Editor running with McpBridge reachable (else `/unity-mcp-environment-setup`).
   - The target scene is open. Hand off to `/task-scenes` (`open_scene_tool`) if not.
   - The scene carries no unsaved edits. `/task-scenes` reads `is_dirty` through `inspect_scene_tool` and clears it
     through `save_scene_tool`, which refuses to run in Play Mode and on a scene that has never been saved.
   - No acquisition session is armed against this Editor (see the interaction contract below).
   - The scene's Actor references the `Simulated Linear` controller. Only `SimulatedLinearTreadmill` reads keyboard
     input. `Linear` subscribes to the hardware `Motion` MQTT topic and ignores the keyboard entirely
     (`LinearTreadmill.Start`). Hand off to `/task-parameters` to write `actor.controller = "Simulated Linear"`, or
     to `/scene-setup`, before entering Play Mode.
2. **Read current state:**
   ```text
   get_play_state_tool()
   ```
   If `state == "playing"`, skip to step 4. If `state == "compiling"`, you SHOULD wait and re-poll rather than issuing
   the call. `EnterPlayMode` (`McpBridge.cs`) guards only on `EditorApplication.isPlaying`, so the call is accepted and
   Unity defers the transition until the recompile finishes (see the interaction contract).
3. **Enter Play Mode:**
   ```text
   enter_play_mode_tool()
   ```
   The response's `state` is `entering_play_mode` when the bridge has issued the transition (or `playing` if the editor
   was already in Play Mode). The tool does not block on completion, so poll `get_play_state_tool` about once a second
   for up to 30 s to confirm `playing` before exercising the scene. Entry runs a full domain and scene reload
   (`m_EnterPlayModeOptionsEnabled: 0` in `ProjectSettings/EditorSettings.asset`), so an `edit` reading inside that
   window means the transition is still running. Conclude that Unity refused entry only once the window expires.

   Play Mode entry also connects the scene's `MQTTClient` to the broker configured in the MQTT section
   (`MQTTConnectorObject.OnEnable`). If the active scene hosts no client singleton, the connector logs `Unable to
   connect to the MQTT broker on enable. ... MQTTClient.Instance is null.` and never attempts the connection
   (`MQTTConnectorObject.cs`). Otherwise it calls `MQTTClient.Connect`, whose two failure branches both log an error
   opening `Unable to connect to the MQTT broker at <ip>:<port>.` A connect task that faults continues `... The broker
   must accept an MQTT 5.0 connection, but the attempt failed with: ...` and typically reports in milliseconds, which
   is the usual outcome when nothing is bound to the port. A connect task still unresolved when the
   `ConnectTimeoutMilliseconds` budget expires continues `... The connection attempt must resolve within 10000
   milliseconds, but it did not.`, having blocked the Editor main thread for the full 10 s, and most often means an
   unreachable or filtered host. Which network condition lands in which branch is OS network-stack behavior rather
   than anything `MQTTClient.Connect` guarantees (`MQTTClient.cs`), so read either message as the same outcome: no
   broker was reached. A keyboard-only run needs no broker: `MQTTClient.Publish` falls back to in-process delivery
   and logs a warning opening `Unable to deliver '<topic>' to the MQTT broker at <ip>:<port>. ... the message reaches
   in-process subscribers only ...` once per topic (`MQTTClient.cs`). Treat that warning as expected during an
   interactive Play Mode run, not as a defect.
4. **Ask the user to exercise the task:** The developer drives the scene from the Game view with the keyboard.
   `Movement` (forward/back) advances the simulated treadmill and `Jump` (spacebar) publishes a synthetic `Interaction`
   (`SimulatedLinearTreadmill.Update` and `SimulatedLinearTreadmill.GetSimulatedInput`). Claude cannot control animal
   movement from the MCP layer.
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

Scene changes and prefab regeneration are unsafe while the Editor is `playing`. You MUST check the state before issuing
them:

```text
state = get_play_state_tool()
if state["state"] == "playing":
    exit_play_mode_tool()   # then re-poll until "edit"
elif state["state"] == "compiling":
    wait and re-poll get_play_state_tool()  # exit_play_mode_tool no-ops here
```

You MUST branch on the two non-edit states separately. `ExitPlayMode` guards on `EditorApplication.isPlaying`
(`McpBridge.cs`), which is false while compiling, so calling it against `compiling` short-circuits to `edit`
without waiting for the recompile to finish.

You MUST hand off to the owning skill (`/task-scenes`, `/task-prefabs`) only after the Editor returns to `edit`.

---

## Interaction contract

- **You MUST NOT** call `exit_play_mode_tool` while an acquisition session is armed against this Editor, and MUST NOT
  exit and re-enter Play Mode. The scene's `SessionStop` broadcast is read by `VRTaskDriver.cycle` as
  `UNITY_TERMINATED`, which sets `terminated = True` and forces the runtime into an emergency pause requiring
  `resume_after_unity_restart` (`sollertia-experiment/src/sollertia_experiment/vr_task/driver.py`). Confirm no
  session is running before issuing any transition.
- **You SHOULD NOT** call `enter_play_mode_tool` while `state == "compiling"`. The bridge does not gate this.
  `EnterPlayMode` (`McpBridge.cs`) only guards on `EditorApplication.isPlaying` and forwards the call to
  `EditorApplication.EnterPlaymode`, which Unity then defers until compilation finishes. The transition is queued rather
  than ambiguous, but the deterministic result is not visible until a follow-up `get_play_state_tool` poll.
- **Calling a transition tool in its target state is safe.** `enter_play_mode_tool` against `playing` (or
  `exit_play_mode_tool` against `edit`) short-circuits with the canonical steady-state `state` and an `Already in Play
  Mode.` / `Not in Play Mode.` message. You SHOULD still avoid these calls to keep the transcript focused.
- **You MUST** confirm the final state with `get_play_state_tool` after issuing a transition. The transition tools
  acknowledge the request (`entering_play_mode` / `exiting_play_mode`) but do not block on completion, so an `edit`
  reading taken right after `enter_play_mode_tool` is ambiguous. It means either that the reload is still running or
  that Unity refused entry because the active scene has compile errors. Re-poll for up to 30 s, then read the refusal
  with `read_console_tool(level="error")`.
- **Play Mode publishes on the configured broker.** Entering Play Mode makes the scene's `MQTTClient` connect to the
  IP/port in the MQTT section and broadcast `SessionStart`. Exiting broadcasts `SessionStop` (`MQTTClient.Start` and
  `MQTTClient.OnApplicationQuit`). Trigger zones publish `Stimulus` and `Delay` while playing. If an acquisition
  runtime is attached to that broker, an interactive Play Mode run injects real messages into the live session. Point
  the MQTT section at an isolated broker, or at a closed port, before exercising a task interactively. Entry against a
  port with no broker behind it can block the Editor main thread until the connect attempt resolves, for up to the full
  10 s budget.
- **Task Parameters re-opens on Play Mode entry.** `MainWindow.RegisterAutoOpen` registers an
  `EditorApplication.playModeStateChanged` hook that calls `EnsureWindowOpen` when the editor reaches
  `PlayModeStateChange.EnteredPlayMode`. The exception is a batch-mode Editor, where `RegisterAutoOpen` returns before
  subscribing any hook (`MainWindow.cs`), so a headless `-runTests -testPlatform PlayMode` run never opens the window.
  The MQTT section, the Task section, and the Camera Mapping `Show Full-Screen Views` control are disabled at runtime,
  so you SHOULD flip the Task flags via MQTT (`/mqtt-contract`) instead of `/task-parameters` during a Play Mode run.
  Note the GUI disable is cosmetic from the agent's side. The bridge itself has no play-state guard on
  `write_task_parameters`. `EditorApplication.isPlaying` is consulted only by `EnterPlayMode`, `ExitPlayMode`,
  `GetPlayState`, and `SaveScene` (`McpBridge.cs`). Thus a write issued during Play Mode is accepted and applied to the
  runtime scene instance. The scene-component values (the Task flags) revert when Play Mode exits, so that half of the
  write is silently lost. The MQTT ip/port values additionally persist to EditorPrefs (`McpBridge.ApplyMqttSection`),
  which `MainWindow.EnsureMqttDefaults` and `MQTTClient.Awake` re-apply to the scene on the next window init or Play
  Mode entry. The MQTT half of the write therefore survives the exit, and the Task-flag half does not. This is why MQTT
  is the correct path for runtime flag changes.

---

## Troubleshooting

| Symptom                                                                          | Cause                                                                                                              | Resolution                                                                                        |
|----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| `enter_play_mode_tool` returns `entering_play_mode` but follow-up poll is `edit` | The domain and scene reload has not landed yet, or Unity refused entry because the active scene has compile errors | Re-poll on step 3's schedule first, then read the refusal with `read_console_tool(level="error")` |
| `enter_play_mode_tool` returns `entering_play_mode` while `compiling`            | Script recompile in progress, and Unity will run the transition once it finishes                                   | Wait and re-poll `get_play_state_tool`                                                            |
| `exit_play_mode_tool` returns `state == "edit"` immediately                      | Editor already in `edit`, and the handler short-circuits with `Not in Play Mode.`                                  | Expected, no further action needed                                                                |
| A poll fails with `Unable to complete the request to the Unity Editor ...`       | Play Mode entry triggered a domain reload and the Editor main thread is not draining the bridge queue              | Wait for the Editor to settle, then re-poll, because this is not a bridge outage                  |
| Console logs `Unable to connect to the MQTT broker on enable ...`                | The active scene hosts no `MQTTClient` component, so `MQTTConnectorObject.OnEnable` returns before connecting      | Open `Window → Task Parameters` to rerun `InitializeScene`, which recreates it (`/scene-setup`)   |
| Console logs `Unable to connect to the MQTT broker ... failed with: ...`         | The connect task faulted, typically in milliseconds, most often because nothing is bound to the IP/port            | Expected for keyboard-only runs, otherwise start the broker or fix IP/port via `/task-parameters` |
| Console logs `Unable to connect to the MQTT broker ... must resolve within ...`  | The connect attempt did not resolve within `ConnectTimeoutMilliseconds`, most often an unreachable/filtered host   | Expected without a broker, otherwise check the host address and firewall, then fix the IP/port    |
| Console logs `Unable to deliver '<topic>' to the MQTT broker ...`                | `MQTTClient.Publish` fell back to in-process delivery, warning once per topic (`MQTTClient.cs`)                    | Expected while playing without a broker, rather than a defect                                     |
| Active scene is not the one expected                                             | A different scene was opened previously                                                                            | Hand off to `/task-scenes` (`open_scene_tool`)                                                    |
| All tools fail with `Unable to reach the Unity Editor at http://localhost:8090/` | McpBridge down                                                                                                     | `/unity-mcp-environment-setup`                                                                    |

`read_console_tool` reads the Console rows above without a human at the Editor. Its buffer holds the last 500 entries
logged since the Editor loaded, and a domain reload starts it empty, so entries predating a Play Mode entry are gone
once the state reaches `playing` (`ReadConsole` in `McpBridge.cs`).

---

## Related skills

| Skill                                        | Relationship                                                                                                                                                                             |
|----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/unity-mcp-environment-setup` (this plugin) | Run first if Unity Editor is unreachable, and owns `read_console_tool` for reading Play Mode Console output                                                                              |
| `/task-scenes` (this plugin)                 | Upstream, opens the scene to exercise in Play Mode and saves it before entry (`save_scene_tool`)                                                                                         |
| `/scene-setup` (this plugin)                 | Upstream, must pass the pre-Play Mode checklist first                                                                                                                                    |
| `/task-prefabs` (this plugin)                | Upstream, generates the prefab under test                                                                                                                                                |
| `/task-parameters` (this plugin)             | Upstream, sets Actor / Task / Display fields in edit mode before entering Play Mode                                                                                                      |
| `/unity-tests` (this plugin)                 | Owns the automated EditMode / PlayMode suite, which drives its own Play Mode transitions                                                                                                 |
| `/mqtt-contract` (this plugin)               | Reference for topics that drive runtime behavior and runtime alternatives to Task Parameters writes                                                                                      |
| `experiment:vr-driver-interface`             | Counterpart, owns the runtime-side use of these same three bridge tools (`_arm_unity` / `_stop_unity`)                                                                                   |
| `experiment:acquisition-system-runtime`      | Reference for the platform-general runtime state machine and per-cycle loop into which the VR driver plugs, and it delegates the Unity driver itself to `experiment:vr-driver-interface` |
| `assets:assets-mcp-environment-setup`        | Upstream, owns the slsa MCP server diagnostic                                                                                                                                            |

---

## Verification checklist

```text
- [ ] Unity Editor is running and McpBridge is reachable
- [ ] No acquisition session was armed against this Editor when the transition was issued
- [ ] get_play_state_tool was called before enter_play_mode_tool / exit_play_mode_tool
- [ ] The response's "success" was checked before its "state" was read
- [ ] The target scene was open before entering Play Mode
- [ ] The scene's Actor referenced the "Simulated Linear" controller
- [ ] The scene was saved (is_dirty == false) before enter_play_mode_tool
- [ ] Console errors were read with read_console_tool when the entry poll never reached "playing"
- [ ] State returned to "edit" after exit_play_mode_tool
- [ ] No mutating Unity operation was issued while state was "playing" or "compiling"
```
