---
name: unity-mcp-environment-setup
description: >-
  Diagnoses and resolves Unity Editor relay connectivity issues for the sollertia-virtual-reality `McpBridge`, covering
  its HTTP listener on 127.0.0.1:8090, [::1]:8090, and localhost:8090, a running Editor, and a compiled script. Also
  owns the contract for adding a new tool to the bridge. Use when Unity relay tools fail with "Unable to reach" or
  "Unable to complete the request to" the Unity Editor, when extending the bridge tool surface, or when starting a
  session that needs the Unity tools.
user-invocable: false
---

# Sollertia Unity MCP environment setup

Diagnoses and resolves the **Unity-side** wiring of the Unity Editor relay used by every Unity-family tool exposed by
`sollertia-shared-assets`. The `slsa mcp` server itself is owned by `assets:assets-mcp-environment-setup`.

---

## Scope

**Covers:**
- Verifying the Unity Editor is running with the `sollertia-virtual-reality` project open
- Verifying the `McpBridge` HTTP listener is active on `127.0.0.1:8090`, `[::1]:8090`, and `localhost:8090`
- Verifying the macOS / Linux monitor enumeration helper on which the Camera Mapping tools depend
- Testing the relay from the command line
- Diagnosing why Unity relay tools return "Unable to reach the Unity Editor at ..." or "Unable to complete the request
  to the Unity Editor at ..."
- Adding a new tool to the bridge: the `Dispatch` case, the handler contract, and the matching `@mcp.tool()` wrapper in
  `sollertia-shared-assets`

**Does not cover:**
- Diagnosing `slsa` CLI / `slsa mcp` availability (see `assets:assets-mcp-environment-setup`)
- Sollertia working directory setup (see `assets:working-directory`)
- Unity Editor installation or project setup (see the `sollertia-virtual-reality` README)
- The Unity test suite and the assembly-definition catalog (see `/unity-tests`)
- Prefab, zone, scene, Task Parameters, or Play Mode workflows (see `/task-prefabs`, `/zone-prefabs`, `/task-scenes`,
  `/task-parameters`, `/play-mode`)

---

## Architecture

Unity tools are served by the **same** `slsa mcp` MCP server as every other shared-assets tool. The server delegates
Unity operations over HTTP to an editor-side plugin called `McpBridge`:

```text
Claude ↔ slsa mcp (stdio) ↔ HTTP POST to {127.0.0.1, [::1], localhost}:8090 ↔ Unity Editor McpBridge
```

The `McpBridge` editor plugin ships with `sollertia-virtual-reality`. It starts the HTTP listener on three loopback
prefixes automatically when the Editor loads the project, registering all three because `HttpListener` performs exact
host-header matching. A client requesting `localhost` is rejected by a `127.0.0.1` prefix even though they resolve to
the same socket, and the explicit numeric prefixes additionally work around Mono's IPv6-only resolution of `localhost`
(the `Listener.Prefixes.Add` calls in `McpBridge`'s static constructor, `McpBridge.cs`). Two production clients
depend on the listener: the shipped Python wrapper hard-codes `http://localhost:8090/` as `_UNITY_BRIDGE_URL` in
`sollertia-shared-assets/.../interfaces/unity_tools.py`, while the acquisition runtime's `UnityBridgeClient` defaults
to `127.0.0.1` through `_BRIDGE_HOST` in `sollertia-experiment/.../vr_task/bridge.py`. Both of those prefixes are
load-bearing in production, and `[::1]` is load-bearing on Mono, where the `localhost` prefix alone can bind only the
IPv6 stack. The 18 relayed tools are:

| Tool                         | Owning skill                   |
|------------------------------|--------------------------------|
| `create_task_tool`           | `/task-prefabs`                |
| `delete_task_tool`           | `/task-prefabs`                |
| `inspect_prefab_tool`        | `/task-prefabs`                |
| `delete_asset_tool`          | `/task-prefabs`                |
| `clone_zone_prefab_tool`     | `/zone-prefabs`                |
| `list_assets_tool`           | `/task-scenes`                 |
| `refresh_assets_tool`        | `/task-scenes`                 |
| `list_scenes_tool`           | `/task-scenes`                 |
| `open_scene_tool`            | `/task-scenes`                 |
| `save_scene_tool`            | `/task-scenes`                 |
| `inspect_scene_tool`         | `/task-scenes`                 |
| `enter_play_mode_tool`       | `/play-mode`                   |
| `exit_play_mode_tool`        | `/play-mode`                   |
| `get_play_state_tool`        | `/play-mode`                   |
| `read_task_parameters_tool`  | `/task-parameters`             |
| `write_task_parameters_tool` | `/task-parameters`             |
| `refresh_monitors_tool`      | `/task-parameters`             |
| `read_console_tool`          | `/unity-mcp-environment-setup` |

Four read-only tools serve as **natural shares** that skills beyond their owner may call. `inspect_prefab_tool`
inspects a prefab hierarchy, `get_play_state_tool` confirms the Editor sits in `edit` before a mutating call,
`list_assets_tool` enumerates prefabs for `/task-prefabs`, and `read_console_tool` reads the Unity Console that every
skill prescribing a Console check depends on. Every other tool in the table is owned exclusively by the listed skill.

All 18 tools require **both** the `slsa mcp` MCP server to be connected **and** the Unity Editor to be running with
`sollertia-virtual-reality` open.

---

## Host prerequisites

### Monitor enumeration helper

The Camera Mapping surface, made up of `read_task_parameters_tool`, `write_task_parameters_tool`, and
`refresh_monitors_tool`, enumerates the host's displays through `Monitor.EnumerateMonitors`
(`Assets/Gimbl/Scripts/Displays/Monitor.cs`), which needs an OS-specific helper:

- **macOS** uses [displayplacer](https://github.com/jakehilborn/displayplacer), installed with `brew install
  displayplacer`. `ResolveDisplayPlacerPath` (`Monitor.cs`) tries `/opt/homebrew/bin/displayplacer` (Apple
  Silicon), then `/usr/local/bin/displayplacer` (Intel), then the bare `displayplacer` name resolved through `PATH`.
- **Linux** uses `xrandr` from the X11 server utilities, resolved through `PATH` by the Linux branch of
  `Monitor.EnumerateMonitors` (`Monitor.cs`). Verify with `command -v xrandr`.
- **Windows** enumerates monitors through the operating system and needs no helper.

A missing helper is a Console **warning** rather than an exception. `EnumerateViaSubprocess` (`Monitor.cs`) logs
`Monitor enumeration: failed to start '<command>'.` and carries an `Install it with 'brew install displayplacer'.` hint
on macOS. The relay keeps returning `"success": true`, so the symptom is an empty `state.camera_mapping`, and a
`camera_mapping` write is refused by `ValidateCameraMappingWrites` (`McpBridge.cs`) with "Cannot write camera_mapping:
no monitors were detected on this host." Install the helper first, then call `refresh_monitors_tool`, because refreshing
alone never fixes it.

---

## Diagnostic workflow

You MUST follow these steps in order when a Unity relay tool returns an error message that starts with "Unable to reach
the Unity Editor at". That is the `URLError` branch of `_unity_relay`: the connection was never established, so the
listener is down.

A message starting with "Unable to complete the request to the Unity Editor at" is the sibling `OSError` branch and
means the opposite. The listener IS up and accepted the connection, but the Editor's main thread did not answer within
30 seconds or dropped the response mid-flight. Steps 2 and 3 will pass for it, so go straight to "Domain reload
mid-tool-call" and "First Task Parameters call after a scene change is slow" under "Editor-side foot-guns" instead.

### Step 1: Confirm the slsa MCP server is connected

If the `sollertia-shared-assets` MCP server itself is disconnected, no Unity tool can reach the bridge. Hand off to
`assets:assets-mcp-environment-setup` first.

### Step 2: Confirm the Unity Editor is running

Ask the user to confirm the Unity Editor is open with the `sollertia-virtual-reality` project loaded. If not, instruct
them to open it and wait for the project to finish loading.

### Step 3: Confirm the McpBridge initialized

In the Unity Console, look for the listener log emitted by `McpBridge`'s static constructor:

```text
The MCP bridge is listening on http://127.0.0.1:8090/, http://[::1]:8090/, and http://localhost:8090/
```

If it is absent:
- The Editor may still be compiling. Wait for compilation to finish.
- The `McpBridge` script, or any other script in the project, may have failed to compile. Check the Console for errors
  and ask the user to resolve them.
- Another process may hold port 8090. The Console then carries an error whose text begins
  `Unable to start the MCP bridge HTTP listener.` and ends with the OS-specific exception text that follows
  `but the bind failed with:`. Free the port and reload the project.

If the listener log **is** present but calls still fail, the listener started and then degraded. Search the Console for
`Unable to re-arm the MCP bridge HTTP listener` or `Unable to capture an incoming bridge request` (the latter names
`EndGetContext` in its body text), both logged by `OnContextReceived` (`McpBridge.cs`). After a re-arm failure the
listener accepts no further requests, and that is indistinguishable from "listener never started" on the relay side, so
restart the Editor. An `Unable to deliver the bridge response.` warning from `HandleRequest` (`McpBridge.cs`) is benign
by comparison: the client gave up on a long call before the bridge answered.

`McpBridge` is declared `[InitializeOnLoad]` and its static constructor starts the listener on every assembly reload, so
a missing log line points at an in-progress compile, a compile failure, or a port conflict.

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

Once curl succeeds, confirm from Claude by invoking `get_play_state_tool`, the cheapest Unity tool that exercises the
relay. If it returns a structured response, Unity-dependent tools are ready.

---

## Common issues and resolutions

| Symptom                                           | Cause                                                                                      | Resolution                                        |
|---------------------------------------------------|--------------------------------------------------------------------------------------------|---------------------------------------------------|
| "Unable to reach the Unity Editor at ..."         | Editor not running                                                                         | Open the Editor with `sollertia-virtual-reality`  |
| "Unable to reach the Unity Editor at ..."         | `McpBridge` not loaded                                                                     | Wait for compile, verify Console for listener log |
| "Unable to reach the Unity Editor at ..."         | Port 8090 taken by another process                                                         | Free the port, restart the Editor                 |
| "Unable to complete the request to the Unity ..." | `OSError` branch, Editor busy past the 30 s budget or dropped the response mid-flight      | Wait for the Editor to become responsive, retry   |
| "Unable to parse the Unity bridge response: ..."  | Message ends `the payload is not valid UTF-8 encoded JSON.`, or `UnicodeDecodeError`       | Restart the Unity Editor                          |
| "Unable to parse the Unity bridge response: ..."  | Message ends `the payload is a valid JSON value but not an object.`, very rare in practice | Restart the Unity Editor and file an issue        |
| Slow first call, then works                       | Editor warming up after project load, or the per-scene monitor enumeration being rebuilt   | Expected, retry after ~30 seconds                 |
| `state.camera_mapping` comes back empty           | Monitor enumeration helper missing on macOS / Linux                                        | Install the helper, then `refresh_monitors_tool`  |
| Prefab or scene tool returns `"success": false`   | Paths are not project-relative                                                             | Use `Assets/...` paths, never absolute paths      |

---

## Editor-side foot-guns

These failure modes are silent from the MCP side, because the relay simply appears unreachable. Check them before deeper
debugging when the Unity tools stop responding after a recently healthy session.

### Silent failure after a compile error

`McpBridge` is declared `[InitializeOnLoad]`, so it restarts every time Unity reloads its assemblies. If **any** script
in the project fails to compile (including a file unrelated to the bridge), the reload aborts and the listener never
starts. From the MCP side this looks identical to "Editor not running."

- Check the Unity Console for compile errors and fix them first.
- The listener log will reappear on the next successful reload, and the full three-prefix line is `The MCP bridge is
  listening on http://127.0.0.1:8090/, http://[::1]:8090/, and http://localhost:8090/`.

### Breaking the McpBridge assembly references

`McpBridge.cs` compiles into `Sollertia.InfiniteCorridorTask.Editor`, declared by
`Assets/InfiniteCorridorTask/Scripts/Editor/Sollertia.InfiniteCorridorTask.Editor.asmdef` in the same folder. That
assembly sets `"rootNamespace": "SL.Tasks"`, restricts itself to `"includePlatforms": ["Editor"]`, and references
exactly three assemblies: `Sollertia.Gimbl`, `Sollertia.Gimbl.Editor`, and `Sollertia.InfiniteCorridorTask`. Every
script in the project now compiles into a named assembly, and nothing lands in Unity's predefined `Assembly-CSharp`.

- The bridge declares `namespace SL.Tasks` and imports, through the `using` directives at the top of `McpBridge.cs`:
  `Gimbl`, `SL.Config`, `UnityEditor`, `UnityEditor.SceneManagement`, `UnityEngine`, and `UnityEngine.SceneManagement`
  (plus the BCL `System.*` namespaces, which are always available). `Gimbl` spans two of them, because `Monitor` and
  `FullScreenViewManager` come from `Sollertia.Gimbl` while `MainWindow`, which `AcquireFullScreenManager`
  (`McpBridge.cs`) resolves, comes from `Sollertia.Gimbl.Editor`. `SL.Config` resolves through the
  `Sollertia.InfiniteCorridorTask` reference.
- The live foot-gun is the inverse of adding an `.asmdef`: removing or narrowing one of those three references, or
  dropping `"Editor"` from `includePlatforms`, breaks the bridge's imports, the editor assembly fails to compile, and
  the listener never starts. From the MCP side this is indistinguishable from "Editor not running".
- Adding a `using` for a type that lives in an unreferenced assembly has the same effect. Add the owning assembly to the
  `references` array in the same change.
- A new Editor-side script joins this assembly by sitting inside its subtree. A brand-new folder declares its own
  `.asmdef` and is referenced from every assembly that consumes it. `/unity-tests` owns the eight-assembly catalog and
  the test-assembly constraints.

### Second Unity instance stealing the port

`HttpListener` claims `localhost:8090` exclusively. If a second Unity Editor starts with the same project path or a copy
of the repo, its bridge call fails silently in the second Editor's Console. The log line always begins
`Unable to start the MCP bridge HTTP listener.`, and the OS-specific exception text follows its
`but the bind failed with:` clause. That text is typically a "conflicts with an existing registration" wording on
.NET-Windows, or an "address already in use" / `EADDRINUSE` wording on Linux and Mac.

The MCP tools continue to work against the **first** Editor, which is rarely what the user intended. Close the duplicate
Editor, because only one Unity Editor may own the bridge at a time.

### Headless test run holds the project lock

Unity holds a per-project lock, so a headless test run, `Unity -batchmode -nographics -projectPath . -runTests
-testPlatform EditMode -testResults out.xml`, requires the interactive Editor to be **closed** on that project. With the
Editor closed the `[InitializeOnLoad]` bridge never constructs, the listener is down, and every relay tool returns
"Unable to reach the Unity Editor at ...".

- This is expected, not a fault. Do not restart-and-retry into it: confirm with the user whether a headless run is in
  progress before working through the diagnostic steps.
- To keep a bridge-hosting Editor session alive, copy the project directory and run the headless suite against the copy.
  `/unity-tests` owns the suite itself.

### Domain reload mid-tool-call

Unity triggers a domain reload after scripts recompile. If a mutating tool (e.g., `create_task_tool`,
`delete_task_tool`, `write_task_parameters_tool`) is invoked during a reload, the HTTP request may succeed from the OS
but the response never arrives. Wait for `get_play_state_tool` to return `state == "edit"` (not `"compiling"`) before
issuing mutating tool calls.

### First Task Parameters call after a scene change is slow

`McpBridge` caches one `FullScreenViewManager` per scene in `_cachedFullScreenManager` (`McpBridge.cs`), and its
static constructor clears that cache on every `EditorSceneManager.activeSceneChangedInEditMode`. Constructing a manager
runs `Monitor.EnumerateMonitors`, which spawns an OS subprocess on Linux and macOS, bounded at 5000 ms per wait by
`SubprocessTimeoutMilliseconds` (`Monitor.cs`), and opens one short-lived popup window per detected monitor.

- The first `read_task_parameters_tool`, `write_task_parameters_tool`, or `refresh_monitors_tool` call after
  `open_scene_tool` or `create_task_tool` therefore re-pays that cost. On a busy Editor it can push the request toward
  the 30-second relay budget and surface as "Unable to complete the request to the Unity Editor at ...".
- Retry rather than restarting the Editor. The second call hits the cache.
- The popup windows are the enumeration working as designed, not a crash.

### Editor lost focus while headless

On Linux specifically, if the Editor loses OS focus at the moment a `Poll()` tick would have run, the listener can fall
behind. The current implementation uses `EditorApplication.update` which is throttled while unfocused. Tools still work,
but first-call latency can stretch to several seconds. Re-focus the Editor window if calls time out.

---

## Adding a bridge tool

`sollertia-virtual-reality`'s extension table routes the `McpBridge` tool row here, so this section is the contract. A
new tool is **two** changes in **two** repositories, and shipping only the Unity half leaves the tool unreachable from
Claude.

### Unity side

1. **Add the `Dispatch` case.** `Dispatch` (`McpBridge.cs`) is one `switch` expression whose eighteen arms map a wire
   tool name to a handler, with a `_` arm returning `Error($"Unable to dispatch '{tool}'. It must be a declared bridge
   tool, but it is not.")`. Add one arm, in the order the README's bridge table lists it. The wire name is snake_case
   and carries **no** `_tool` suffix, because that suffix belongs to the Python wrapper alone.
2. **Write the handler.** A handler is `private static string`, takes `Dictionary<string, object> arguments` when the
   tool has inputs and nothing when it does not, and returns `Ok(payload)` or `Error(message)`. `Ok` (`McpBridge.cs`)
   stamps `success = true` onto the payload dictionary, and `Error` (`McpBridge.cs`) emits `{"success": false, "error":
   message}`. Every response therefore carries a `success` boolean. Never hand-build a response dictionary and never
   return a bare JSON string.
3. **Serialize through `MiniJson` only.** `MiniJson.Deserialize` parses the request body in `HandleRequest`, and
   `MiniJson.Serialize` writes every response from `Ok` and `Error`, all in `McpBridge.cs`. It covers dictionaries,
   sequences, strings, numbers, booleans, and null, and serializes anything else as its quoted `ToString`. Do not
   reach for `JsonUtility`: it cannot serialize the `Dictionary<string, object>` payload every handler builds.
4. **Call Unity APIs freely.** `OnContextReceived` (`McpBridge.cs`) runs on a thread-pool thread and only enqueues the
   context onto a `ConcurrentQueue`, and `Poll` (`McpBridge.cs`) drains that queue on the editor thread from
   `EditorApplication.update`. Handlers therefore already run where Unity APIs are legal. Do not add
   threading of your own.
5. **Fold a scene-touching tool into the shared walk.** `AcquireSceneComponents` (`McpBridge.cs`) performs the single
   scene walk the Task Parameters endpoints share, and `BuildSnapshot` (`McpBridge.cs`) turns it into the response
   payload. A tool that reads or writes scene state adds its component to the `SceneComponents` struct and its field to
   `BuildSnapshot` instead of running a second `FindAnyObjectByType` pass, which is why `RefreshMonitors`
   (`McpBridge.cs`) is three lines long.
6. **Respect the deletion bounds** whenever the tool removes anything:
   - A new deletable asset root joins `DeleteAllowedPrefixes` (`McpBridge.cs`).
   - A new hand-authored asset joins `DeleteProtectedPaths` (`McpBridge.cs`), which `delete_asset` consults through
     `IsDeleteAllowed` and which `delete_task` consults itself in its `DestroyTask` handler before removing a scene.
   - Scenes are deliberately absent from `DeleteAllowedPrefixes`. They are removed only through `delete_task`, so the
     per-scene companion cascade can never be bypassed, and a new per-scene companion joins
     `TryDeleteScenePerSceneCompanions` (`McpBridge.cs`) in the same change.
   - Build every asset path as a **forward-slash literal**. `Path.Combine` emits backslashes on Windows, and every
     prefix and protected-path comparison above is ordinal against forward-slash strings.

### Python side, in sollertia-shared-assets

The Unity half is unreachable from Claude until a matching wrapper exists in
`sollertia-shared-assets/src/sollertia_shared_assets/interfaces/unity_tools.py`:

- Add an `@mcp.tool()`-decorated function named `<wire_name>_tool` whose body is `return
  _unity_relay(tool="<wire_name>", arguments={...})`, or `_unity_relay(tool="<wire_name>")` when the tool takes no
  arguments.
- `_unity_relay` owns transport and error translation, covering the 30-second budget, both reachability branches, and
  the parse-failure branch. Never POST to the bridge from a tool body directly.
- The docstring is the agent-facing contract: state the return shape and note that the tool requires the Unity Editor
  running with the `McpBridge` plugin active.

### Finish the change

- Add a row to the tool-ownership table in this skill's "Architecture" section naming the owning skill. Then bump all
  three counts ("The 18 relayed tools are:", "All 18 tools require ...", and the "eighteen arms" count in "Adding a
  bridge tool"). An unowned tool breaks the plugin's exclusive-ownership lattice.
- Update the `sollertia-virtual-reality` README's "Editor MCP Bridge" table **and** its `The bridge dispatches **18
  tools**:` count in the same change, because that README is the catalog.
- Update the `sollertia-shared-assets` README's MCP tool table and the Unity-tool list in the `***Note,***` paragraph
  that follows it, because agents read that README as the wrapper catalog.
- Bump the remaining counts: the `sollertia-virtual-reality` README's "18 Editor operations" bullet, its `CLAUDE.md`
  "dispatches 18 tools" line, the `thirteen of the eighteen` counts in `/unity-tests`, the `Unity Editor relay` row
  and the `eighteen Unity tools` phrase in `assets:cli-reference`, and the `<remarks>` count on
  `Dispatch_DeclaredToolName_DoesNotFallThroughToUnknownTool` in `McpBridgeTests.cs`.
- Bump the count in `plugins/assets/skills/assets-mcp-environment-setup/SKILL.md`, which says "18 Unity-relay tools,
  spanning eight families". Its family grouping must still partition the full roster: a new tool either joins one of
  the eight families or makes a ninth, so update the family count in the same edit.
- Document the tool in the owning skill's own tool surface, and register the handler with whichever bridge fixture its
  prerequisites allow, meaning `McpBridgeTests`, `McpBridgeTaskParametersTests`, or `McpBridgePlayModeTests` (see
  `/unity-tests`).

---

## Related skills

| Skill                                 | Relationship                                          |
|---------------------------------------|-------------------------------------------------------|
| `assets:assets-mcp-environment-setup` | Run first, owns the slsa MCP server diagnostic        |
| `/task-prefabs` (this plugin)         | Consumer, prefab generation / inspection / validation |
| `/zone-prefabs` (this plugin)         | Consumer, zone prefab cloning and wiring              |
| `/task-scenes` (this plugin)          | Consumer, scene and asset management                  |
| `/play-mode` (this plugin)            | Consumer, runtime control                             |
| `/task-parameters` (this plugin)      | Consumer, Task Parameters read / write                |
| `/scene-setup` (this plugin)          | Consumer, Editor-time scene configuration             |
| `/task-generator` (this plugin)       | Reference for the `CreateTask` pipeline internals     |
| `/mqtt-contract` (this plugin)        | Reference for MQTT topics crossing this relay's tools |
| `/gimbl-framework` (this plugin)      | Reference for the GIMBL VR framework                  |
| `/unity-tests` (this plugin)          | Owns the test suite and the assembly catalog          |
| `experiment:vr-driver-interface`      | Peer client, reaches the same listener on `127.0.0.1` |
| `assets:task-templates`               | Upstream, prefabs are generated from templates        |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A Unity relay tool fails with "Unable to reach the Unity Editor at" or "Unable to complete the request to the Unity
  Editor at"
- A session begins that needs the Unity relay tools and the Editor is not confirmed running
- The user reports trouble with the Unity Editor relay, the `McpBridge` listener, or port 8090
- A new tool is being added to the bridge, since the `Dispatch` case, the handler, and the `@mcp.tool()` wrapper are
  one contract
- The `slsa` server rather than the Editor is the suspect, in which case hand off to
  `assets:assets-mcp-environment-setup`

---

## Verification checklist

You MUST verify this checklist before declaring the Unity relay healthy or handing off to any downstream Unity-tool
skill.

```text
Unity MCP Environment Compliance:
- [ ] slsa mcp server is connected (assets:assets-mcp-environment-setup)
- [ ] No headless test run is holding the project lock (that run requires the Editor closed)
- [ ] Unity Editor is running with sollertia-virtual-reality open
- [ ] Unity Console shows "The MCP bridge is listening on http://127.0.0.1:8090/, http://[::1]:8090/, and
      http://localhost:8090/"
- [ ] Unity Console shows no "Unable to re-arm the MCP bridge HTTP listener" or "Unable to capture an incoming
      bridge request" line after that
- [ ] curl POST to localhost:8090 returns a JSON success response
- [ ] get_play_state_tool returns a structured response from Claude
- [ ] Camera Mapping work only: displayplacer (macOS) or xrandr (Linux) resolves on the host
```
