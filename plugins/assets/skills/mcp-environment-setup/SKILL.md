---
name: mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-shared-assets MCP server connectivity issues. Covers environment
  verification, command availability for slsa, Python version checks, dependency validation, and
  conda/pip/uv environment configuration. Use when sollertia-shared-assets MCP tools are
  unavailable, when the server fails to start, or when starting a session that requires the assets
  plugin's MCP tools.
user-invocable: true
---

# Sollertia assets MCP environment setup

Diagnoses and resolves sollertia-shared-assets MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the `sollertia-shared-assets` MCP server is reachable
- Diagnosing why the `slsa` command is unavailable
- Checking Python version compatibility
- Validating sollertia-shared-assets package installation and dependencies

**Does not cover:**
- MCP tool usage for any specific configuration task (see other assets plugin skills)
- sollertia-experiment `sl-get` / `sl-manage` MCP servers (see the experiment plugin's MCP env setup)
- Unity Editor or sollertia-unity-tasks installation (see the sollertia-unity-tasks README)
- Unity Editor relay (`McpBridge`) connectivity diagnostics (see the unity plugin's
  `/mcp-environment-setup`)

---

## Architecture

sollertia-shared-assets exposes a single MCP server through the `slsa mcp` Click subcommand
defined in `pyproject.toml`:

```toml
[project.scripts]
slsa = "sollertia_shared_assets.interfaces.cli:slsa_cli"
```

- **Server**: `sollertia-shared-assets`
- **CLI command**: `slsa mcp`
- **Purpose**: Discovery, read, write, and schema introspection of all Sollertia configuration and
  runtime data files. Also relays Unity Editor operations for the unity plugin's tools.

The server accepts a `--transport` option (defaults to `stdio`). The assets plugin's `plugin.json`
configures the Claude assistant to launch the server automatically:

```json
{
  "mcpServers": {
    "sollertia-shared-assets": {
      "command": "slsa",
      "args": ["mcp"]
    }
  }
}
```

`slsa` must be on PATH when the Claude assistant starts.

### Dual-distribution model

| Component                            | Distributed via                 | What it provides                          |
|--------------------------------------|---------------------------------|-------------------------------------------|
| Skills (`/working-directory`, etc.)  | sollertia assets plugin         | Workflow-guiding skill files              |
| MCP server registration              | sollertia assets plugin         | Plugin entry that launches the server     |
| MCP server code (`slsa mcp`)         | sollertia-shared-assets pip pkg | The CLI command and server implementation |

Installing the plugin alone registers the MCP server but the server will fail to start if
`sollertia-shared-assets` is not installed in the active Python environment.

### Unity tools

The `slsa mcp` server also serves a family of Unity-relay tools (prefab, scene, play-mode). Those
tools depend on the Unity Editor running with the `McpBridge` plugin loaded. **That diagnostic is
owned by the unity plugin's `/mcp-environment-setup`** — this skill only covers the slsa CLI /
Python environment side of the stack.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools. If the `sollertia-shared-assets` server is
connected, the issue is not environmental — investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which slsa
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
an environment after the assistant has started does not make `slsa` available to the MCP
subprocess.

### Step 4: Verify Python version compatibility

sollertia-shared-assets requires Python `>=3.14,<3.15`. If the Python version does not match, instruct
the user to create or activate a compatible environment.

### Step 5: Verify package integrity

```bash
slsa --help
```

If the command fails with an import error:

```bash
pip check sollertia-shared-assets 2>&1 | head -20
```

Common version-skew sources are `ataraxis-base-utilities`, `ataraxis-data-structures` (must be `>=6,<7`),
and `mcp` (must be `>=1,<2`).

### Step 6: Restart the MCP server

The user must restart the Claude assistant. The assets plugin will automatically reconnect on
the next session.

### Step 7: Hand off if the issue is Unity-specific

If the slsa server itself is healthy but a Unity-relay tool (prefab / scene / play-mode) returns
"Unity Editor is not reachable", hand off to the unity plugin's `/mcp-environment-setup` — that
skill owns the McpBridge / localhost:8090 diagnostic.

---

## Common issues and resolutions

| Symptom                              | Cause                             | Resolution                                       |
|--------------------------------------|-----------------------------------|--------------------------------------------------|
| `slsa: command not found`            | Environment not activated         | Activate conda/venv and restart                  |
| `slsa: command not found`            | `sollertia-shared-assets` missing | Install the package (see Step 3)                 |
| Import error on `slsa mcp`           | `ataraxis-data-structures` skew   | Upgrade the package (see Step 5)                 |
| Tools fail "no working directory"    | Working directory not initialized | Run `/working-directory` to set it               |
| Tools fail "templates not set"       | Task templates path not set       | Run `/working-directory`                         |
| Write tools fail after connect       | Invalid YAML from a previous edit | Use `discover_*` / `read_*` tools                |
| "Unity Editor is not reachable"      | McpBridge / Editor offline        | Hand off to unity plugin `/mcp-environment-setup`|

---

## Related skills

This skill is a prerequisite for **every** other skill in the assets plugin and the unity plugin —
they all depend on the `sollertia-shared-assets` MCP server being reachable.

| Skill                                      | Asset it owns                                          |
|--------------------------------------------|--------------------------------------------------------|
| `/working-directory`                       | Working directory, credentials, templates dir path     |
| `/task-templates`                          | `TaskTemplate` + trial primitives                      |
| `/experiment-configuration`                | `MesoscopeExperimentConfiguration` + `ExperimentState` |
| `/project-hierarchy`                       | Projects (`create_project_tool`)                       |
| `/session-data`                            | `SessionData` + `SessionTypes` + session status tools  |
| `/session-descriptors`                     | The 4 per-session-type descriptors                     |
| `/subject-metadata`                        | All animal-scoped subject record types                 |
| unity plugin `/mcp-environment-setup`      | Unity Editor / McpBridge diagnostic                    |
| unity plugin `/task-prefabs`               | Task prefab generation, inspection, validation         |
| unity plugin `/scenes`                     | Scene and Unity asset management                       |
| unity plugin `/play-mode`                  | Unity Editor Play Mode control                         |
| experiment plugin `/system-configuration`  | `MesoscopeSystemConfiguration` (moved out)             |
| experiment plugin `/session-snapshots`     | Frozen hardware state / Zaber / mesoscope positions    |
| experiment plugin `/system-health-check`   | Owns `get_acquisition_environment_status_tool`         |
| forging plugin `/server-configuration`     | `ServerConfiguration` (moved out of this plugin)       |
| forging plugin `/datasets`                 | `DatasetData` / `DatasetSession` (moved out)           |
| experiment plugin `/mcp-environment-setup` | Equivalent diagnostic for `sl-get` / `sl-manage`       |

---

## Verification checklist

```text
sollertia-shared-assets MCP environment setup:
- [ ] Checked sollertia-shared-assets MCP server connection status
- [ ] Verified 'slsa' command is on PATH
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that the assistant must be restarted after environment changes
- [ ] Handed off to unity plugin's /mcp-environment-setup if the issue is Unity-specific
```
