---
name: mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-shared-assets MCP server connectivity issues. Covers environment
  verification, command availability for sl-configure, Python version checks, dependency validation,
  conda/pip/uv environment configuration, and Unity Editor relay connectivity for Unity-dependent tools.
  Use when sollertia-shared-assets MCP tools are unavailable, when the server fails to start, when
  Unity relay tools fail with "Unity Editor is not reachable", or when starting a session that requires
  the configuration MCP tools.
user-invocable: true
---

# Sollertia configuration MCP environment setup

Diagnoses and resolves sollertia-shared-assets MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the `sollertia-shared-assets` MCP server is reachable
- Diagnosing why the `sl-configure` command is unavailable
- Checking Python version compatibility
- Validating sollertia-shared-assets package installation and dependencies
- Diagnosing Unity Editor relay connectivity for Unity-dependent tools

**Does not cover:**
- MCP tool usage for any specific configuration task (see other configuration plugin skills)
- sollertia-experiment `sl-get` / `sl-manage` MCP servers (see the experiment plugin's MCP env setup)
- Unity Editor or sollertia-unity-tasks installation (see the sollertia-unity-tasks README)

---

## Architecture

sollertia-shared-assets exposes a single MCP server through the `sl-configure mcp` Click subcommand
defined in `pyproject.toml`:

```toml
[project.scripts]
sl-configure = "sollertia_shared_assets.interfaces.configure:configure"
```

- **Server**: `sollertia-shared-assets`
- **CLI command**: `sl-configure mcp`
- **Purpose**: Discovery, read, write, and schema introspection of all Sollertia configuration and
  runtime data files

The server accepts a `--transport` option (defaults to `stdio`). The configuration plugin's `plugin.json`
configures the Claude assistant to launch the server automatically:

```json
{
  "mcpServers": {
    "sollertia-shared-assets": {
      "command": "sl-configure",
      "args": ["mcp"]
    }
  }
}
```

`sl-configure` must be on PATH when the Claude assistant starts.

### Dual-distribution model

| Component                            | Distributed via                 | What it provides                          |
|--------------------------------------|---------------------------------|-------------------------------------------|
| Skills (`/working-directory`, etc.)  | sollertia configuration plugin  | Workflow-guiding skill files              |
| MCP server registration              | sollertia configuration plugin  | Plugin entry that launches the server     |
| MCP server code (`sl-configure mcp`) | sollertia-shared-assets pip pkg | The CLI command and server implementation |

Installing the plugin alone registers the MCP server but the server will fail to start if
`sollertia-shared-assets` is not installed in the active Python environment.

### Unity Editor relay

A subset of tools registered by `sollertia-shared-assets` relay requests to the Unity Editor via HTTP.
The sollertia-unity-tasks project includes an `McpBridge` editor plugin that starts an HTTP listener on
`localhost:8090` when the Editor loads. The relay path is:

```text
Claude ↔ sl-configure mcp (stdio) ↔ HTTP POST to localhost:8090 ↔ Unity Editor McpBridge
```

The relayed tools are: `generate_task_prefab_tool`, `inspect_prefab_tool`,
`validate_prefab_against_template_tool`, `list_unity_assets_tool`, `list_scenes_tool`, `open_scene_tool`,
`create_scene_tool`, `enter_play_mode_tool`, `exit_play_mode_tool`, and `get_play_state_tool`. These tools
require **both** the `sollertia-shared-assets` MCP server to be connected **and** the Unity Editor to be
running with the sollertia-unity-tasks project open. All other `sollertia-shared-assets` tools (configuration
read/write, schema introspection, template authoring) work without the Unity Editor.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools. If the `sollertia-shared-assets` server is
connected, the issue is not environmental — investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which sl-configure
```

### Step 3: Identify the environment type and resolve

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep sollertia-shared-assets
```

The conda / venv / uv decision tree is identical to other Sollertia / ataraxis MCP environment setup
skills. The package to install is:

```bash
pip install sollertia-shared-assets
# or
uv pip install sollertia-shared-assets
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating
an environment after the assistant has started does not make `sl-configure` available to the MCP
subprocess.

### Step 4: Verify Python version compatibility

sollertia-shared-assets requires Python `>=3.14,<3.15`. If the Python version does not match, instruct
the user to create or activate a compatible environment.

### Step 5: Verify package integrity

```bash
sl-configure --help
```

If the command fails with an import error:

```bash
pip check sollertia-shared-assets 2>&1 | head -20
```

Common version-skew sources are `ataraxis-base-utilities`, `ataraxis-data-structures` (must be `>=6,<7`),
and `mcp` (must be `>=1,<2`).

### Step 6: Restart the MCP server

The user must restart the Claude assistant. The configuration plugin will automatically reconnect on
the next session.

### Step 7: Verify Unity Editor relay (Unity-dependent tools only)

This step is only required when a Unity relay tool fails with "Unity Editor is not reachable". It is
not part of the base MCP diagnostic and should be skipped when the issue is with non-Unity tools.

1. Confirm the Unity Editor is running and has the sollertia-unity-tasks project open.
2. Check the Unity Console for `McpBridge: Listening on http://localhost:8090/` — this log line confirms
   the bridge initialized successfully. If it is absent, the Editor may still be loading or the McpBridge
   script may have failed to compile.
3. Test connectivity from the command line:

```bash
curl -s -X POST http://localhost:8090/ \
  -H "Content-Type: application/json" \
  -d '{"tool": "get_play_state", "args": {}}' | python -m json.tool
```

If the request returns a JSON response with `"success": true`, the bridge is healthy and the issue is on
the `sollertia-shared-assets` relay side (return to Step 1). If the request times out or is refused, the
Unity Editor is not listening — restart the Editor and wait for the McpBridge log line.

---

## Common issues and resolutions

| Symptom                              | Cause                             | Resolution                                       |
|--------------------------------------|-----------------------------------|--------------------------------------------------|
| `sl-configure: command not found`    | Environment not activated         | Activate conda/venv and restart                  |
| `sl-configure: command not found`    | `sollertia-shared-assets` missing | Install the package (see Step 3)                 |
| Import error on `sl-configure mcp`   | `ataraxis-data-structures` skew   | Upgrade the package (see Step 5)                 |
| Tools fail "no working directory"    | Working directory not initialized | Run `/working-directory` to set it               |
| Tools fail "templates not set"       | Task templates path not set       | Run `/working-directory`                         |
| Write tools fail after connect       | Invalid YAML from a previous edit | Use `discover_*` / `read_*` tools                |
| "Unity Editor is not reachable"      | Editor not running                | Open the Unity Editor with sollertia-unity-tasks |
| "Unity Editor is not reachable"      | McpBridge not loaded              | Verify the Editor finished loading (see Step 7)  |
| "Unity bridge returned invalid JSON" | McpBridge returned malformed data | Restart the Unity Editor                         |

---

## Related skills

This skill is a prerequisite for **every** other skill in the configuration plugin — they all depend
on the `sollertia-shared-assets` MCP server being reachable.

| Skill                                      | Asset it owns                                          |
|--------------------------------------------|--------------------------------------------------------|
| `/working-directory`                       | Working directory, credentials, templates dir path     |
| `/system-configuration`                    | `MesoscopeSystemConfiguration`                         |
| `/server-configuration`                    | `ServerConfiguration`                                  |
| `/task-templates`                          | `TaskTemplate` + trial primitives                      |
| `/experiment-configuration`                | `MesoscopeExperimentConfiguration` + `ExperimentState` |
| `/project-hierarchy`                       | Projects (`create_project_tool`)                       |
| `/session-data`                            | `SessionData` + `SessionTypes`                         |
| `/session-descriptors`                     | The 4 per-session-type descriptors                     |
| `/session-snapshots`                       | Frozen hardware state, Zaber, mesoscope positions      |
| `/subject-metadata`                        | All animal-scoped subject record types                 |
| `/datasets`                                | `DatasetData` / `DatasetSession`                       |
| experiment plugin `/mcp-environment-setup` | Equivalent diagnostic for `sl-get` / `sl-manage`       |

---

## Verification checklist

```text
sollertia-shared-assets MCP environment setup:
- [ ] Checked sollertia-shared-assets MCP server connection status
- [ ] Verified 'sl-configure' command is on PATH
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that the assistant must be restarted after environment changes
- [ ] (Unity tools only) Verified Unity Editor is running with McpBridge on localhost:8090
```
