---
name: unity-mcp-environment-setup
description: >-
  Diagnoses and resolves Unity Editor relay connectivity issues for the sollertia-virtual-reality
  `McpBridge` (HTTP listener on 127.0.0.1:8090, [::1]:8090, and localhost:8090; Editor running;
  script compiled). Use when Unity relay tools fail with "Unity Editor is not reachable" or when
  starting a session that needs the Unity tools.
user-invocable: false
---

# Sollertia Unity MCP environment setup

Diagnoses and resolves the **Unity-side** wiring of the Unity Editor relay used by every
Unity-family tool exposed by `sollertia-shared-assets` — the `slsa mcp` server itself is owned by
`assets:assets-mcp-environment-setup`.

---

## Scope

**Covers:**
- Verifying the Unity Editor is running with the `sollertia-virtual-reality` project open
- Verifying the `McpBridge` HTTP listener is active on `127.0.0.1:8090`, `[::1]:8090`, and
  `localhost:8090`
- Testing the relay from the command line
- Diagnosing why Unity relay tools return "Unity Editor is not reachable"

**Does not cover:**
- Diagnosing `slsa` CLI / `slsa mcp` availability (see `assets:assets-mcp-environment-setup`)
- Sollertia working directory setup (see `assets:working-directory`)
- Unity Editor installation or project setup (see the `sollertia-virtual-reality` README)
- Prefab, scene, Task Parameters, or Play Mode workflows (see `/task-prefabs`, `/task-scenes`,
  `/task-parameters`, `/play-mode`)

---

## Architecture

Unity tools are served by the **same** `slsa mcp` MCP server as every other shared-assets tool. The
server delegates Unity operations over HTTP to an editor-side plugin called `McpBridge`:

```text
Claude ↔ slsa mcp (stdio) ↔ HTTP POST to {127.0.0.1, [::1], localhost}:8090 ↔ Unity Editor McpBridge
```

The `McpBridge` editor plugin ships with `sollertia-virtual-reality`. It starts the HTTP listener on
three loopback prefixes automatically when the Editor loads the project — registering all three
because `HttpListener` performs exact host-header matching. A client requesting `localhost` is
rejected by a `127.0.0.1` prefix even though they resolve to the same socket. The shipped Python
wrapper (`sollertia-shared-assets/.../interfaces/unity_tools.py:19`) hard-codes
`http://localhost:8090/` as `_UNITY_BRIDGE_URL`, so the `127.0.0.1` and `[::1]` prefixes exist as
defensive coverage for ad-hoc `curl` callers — operationally, only the `localhost` prefix needs to
be reachable for the relay to work. The 13 relayed tools are:

| Tool                         | Owning skill       |
|------------------------------|--------------------|
| `create_task_tool`           | `/task-prefabs`    |
| `delete_task_tool`           | `/task-prefabs`    |
| `inspect_prefab_tool`        | `/task-prefabs`    |
| `delete_asset_tool`          | `/task-prefabs`    |
| `list_assets_tool`           | `/task-scenes`     |
| `list_scenes_tool`           | `/task-scenes`     |
| `open_scene_tool`            | `/task-scenes`     |
| `inspect_scene_tool`         | `/task-scenes`     |
| `enter_play_mode_tool`       | `/play-mode`       |
| `exit_play_mode_tool`        | `/play-mode`       |
| `get_play_state_tool`        | `/play-mode`       |
| `read_task_parameters_tool`  | `/task-parameters` |
| `write_task_parameters_tool` | `/task-parameters` |

All 13 tools require **both** the `slsa mcp` MCP server to be connected **and** the Unity Editor
to be running with `sollertia-virtual-reality` open.

---

## Diagnostic workflow

You MUST follow these steps in order when a Unity relay tool returns "Unity Editor is not reachable".

### Step 1: Confirm the slsa MCP server is connected

If the `sollertia-shared-assets` MCP server itself is disconnected, no Unity tool can reach the
bridge. Hand off to `assets:assets-mcp-environment-setup` first.

### Step 2: Confirm the Unity Editor is running

Ask the user to confirm the Unity Editor is open with the `sollertia-virtual-reality` project loaded.
If not, instruct them to open it and wait for the project to finish loading.

### Step 3: Confirm the McpBridge initialized

In the Unity Console, look for the listener log emitted by `McpBridge`'s static constructor:

```text
McpBridge: Listening on http://127.0.0.1:8090/, http://[::1]:8090/, and http://localhost:8090/
```

If it is absent:
- The Editor may still be compiling — wait for compilation to finish.
- The `McpBridge` script may have failed to compile — check the Console for errors and ask the user
  to resolve them.
- The Editor may have disabled the plugin — ask the user to re-enable it in `Edit → Preferences →
  External Tools` (or the McpBridge settings panel, depending on the project version).

### Step 4: Test the bridge from the command line

```bash
curl -s -X POST http://localhost:8090/ \
  -H "Content-Type: application/json" \
  -d '{"tool": "get_play_state", "args": {}}' | python -m json.tool
```

Expected outcomes:

| Response                     | Meaning                                  | Next step                             |
|------------------------------|------------------------------------------|---------------------------------------|
| JSON with `"success": true`  | Bridge healthy                           | Re-run the failing Unity tool         |
| JSON with `"success": false` | Bridge reachable but rejected the tool   | Inspect the error message, fix inputs |
| Connection refused / timeout | Listener is not running                  | Return to Step 3                      |
| HTML / unexpected text       | Port 8090 is held by a different process | Kill the other process, restart Unity |

### Step 5: Verify by retrying a read-only Unity tool

Once curl succeeds, confirm from Claude by invoking `get_play_state_tool` — the cheapest Unity tool
that exercises the relay. If it returns a structured response, Unity-dependent tools are ready.

---

## Common issues and resolutions

| Symptom                                        | Cause                                                                                        | Resolution                                        |
|------------------------------------------------|----------------------------------------------------------------------------------------------|---------------------------------------------------|
| "Unity Editor is not reachable"                | Editor not running                                                                           | Open the Editor with `sollertia-virtual-reality`      |
| "Unity Editor is not reachable"                | McpBridge not loaded                                                                         | Wait for compile, verify Console for listener log |
| "Unity Editor is not reachable"                | Port 8090 taken by another process                                                           | Free the port, restart the Editor                 |
| "Unity Editor is not reachable" after ~30s     | Hard timeout on Python wrapper's `urllib.urlopen(timeout=30)`; Editor busy or hanging        | Retry after the Editor finishes compiling         |
| "Unity bridge returned invalid JSON"           | McpBridge produced a non-JSON response                                                       | Restart the Unity Editor                          |
| "Unity bridge returned non-object payload"     | Response parsed as JSON but is not a dict (very rare; should not occur in practice)          | Restart the Unity Editor and file an issue        |
| Slow first call, then works                    | Editor warming up after project load                                                         | Expected — retry after ~30 seconds                |
| Tools work, but prefab/scene paths 404         | Paths are not project-relative                                                               | Use `Assets/...` paths, never absolute paths      |

---

## Editor-side foot-guns

These failure modes are silent from the MCP side — the relay simply appears unreachable. Check them before deeper
debugging when the Unity tools stop responding after a recently healthy session.

### Silent failure after a compile error

`McpBridge` is declared `[InitializeOnLoad]`, so it restarts every time Unity reloads its assemblies. If **any** script
in the project fails to compile (including a file unrelated to the bridge), the reload aborts and the listener never
starts. From the MCP side this looks identical to "Editor not running."

- Check the Unity Console for compile errors — fix them first.
- The listener log will reappear on the next successful reload — the full three-prefix line is
  `McpBridge: Listening on http://127.0.0.1:8090/, http://[::1]:8090/, and http://localhost:8090/`.

### Moving McpBridge.cs under an assembly definition

`McpBridge.cs` lives at `Assets/InfiniteCorridorTask/Scripts/Editor/` without an enclosing `.asmdef`, so it compiles
into the default editor assembly. Adding an `.asmdef` to that folder (or any ancestor) without also referencing every
assembly the bridge depends on will break its imports and the listener will fail to start.

- The bridge declares `namespace SL.Tasks` and imports (`McpBridge.cs:8-20`): `SL.Config`, `Gimbl`, `UnityEditor`,
  `UnityEditor.SceneManagement`, `UnityEngine`, and `UnityEngine.SceneManagement` (plus the BCL `System.*` namespaces,
  which are always available). Any introduced `.asmdef` must declare references covering each of those assemblies —
  notably `Gimbl` (the inlined GIMBL framework) and `SL.Config` (the schema-mirror namespace), both of which sit in
  separate logical assemblies.
- Easiest safe path: leave the Editor folder outside any `.asmdef` — the project has worked this way since bridge
  inception.

### Second Unity instance stealing the port

`HttpListener` claims `localhost:8090` exclusively. If a second Unity Editor starts with the same project path or a
copy of the repo, its bridge call fails silently in the second Editor's Console with a log line whose **prefix** is
always `McpBridge: Failed to start HTTP listener:` followed by the OS-specific exception text — typically a
"conflicts with an existing registration" wording on .NET-Windows or an "address already in use" / `EADDRINUSE`
wording on Linux and Mac.

The MCP tools continue to work against the **first** Editor, which is rarely what the user intended. Close the
duplicate Editor; only one Unity Editor may own the bridge at a time.

### Domain reload mid-tool-call

Unity triggers a domain reload after scripts recompile. If a mutating tool (e.g.,
`create_task_tool`, `delete_task_tool`, `write_task_parameters_tool`) is invoked during a reload, the HTTP request
may succeed from the OS but the response never arrives. Wait for `get_play_state_tool` to return
`state == "edit"` (not `"compiling"`) before issuing mutating tool calls.

### Editor lost focus while headless

On Linux specifically, if the Editor loses OS focus at the moment a `Poll()` tick would have run, the listener can
fall behind. The current implementation uses `EditorApplication.update` which is throttled while unfocused. Tools
still work, but first-call latency can stretch to several seconds. Re-focus the Editor window if calls time out.

---

## Related skills

| Skill                                         | Relationship                                           |
|-----------------------------------------------|--------------------------------------------------------|
| `assets:assets-mcp-environment-setup` | Run first — owns the slsa MCP server diagnostic        |
| `/task-prefabs` (this plugin)                 | Consumer — prefab generation / inspection / validation |
| `/task-scenes` (this plugin)                  | Consumer — scene and asset management                  |
| `/play-mode` (this plugin)                    | Consumer — runtime control                             |
| `/task-parameters` (this plugin)              | Consumer — Task Parameters read / write                |
| `/scene-setup` (this plugin)                  | Consumer — Editor-time scene configuration             |
| `/task-generator` (this plugin)               | Reference for the `CreateTask` pipeline internals      |
| `/mqtt-contract` (this plugin)                | Reference for MQTT topics crossing this relay's tools  |
| `/gimbl-framework` (this plugin)              | Reference for the GIMBL VR framework                   |
| `assets:task-templates`               | Upstream — prefabs are generated from templates        |

---

## Verification checklist

You MUST verify this checklist before declaring the Unity relay healthy or handing off to any
downstream Unity-tool skill.

```text
Unity MCP Environment Compliance:
- [ ] slsa mcp server is connected (assets:assets-mcp-environment-setup)
- [ ] Unity Editor is running with sollertia-virtual-reality open
- [ ] Unity Console shows "McpBridge: Listening on http://127.0.0.1:8090/, http://[::1]:8090/, and http://localhost:8090/"
- [ ] curl POST to localhost:8090 returns a JSON success response
- [ ] get_play_state_tool returns a structured response from Claude
```
