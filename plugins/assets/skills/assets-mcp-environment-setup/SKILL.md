---
name: assets-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-shared-assets MCP server connectivity issues (environment,
  `slsa` command availability, Python version, dependencies). Use when the MCP tools are
  unavailable, the server fails to start, or a new session needs the assets plugin's tools.
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
- sollertia-experiment MCP servers (see the experiment plugin's `/experiment-mcp-environment-setup`)
- sollertia-forgery MCP server (see the forging plugin's `/forging-mcp-environment-setup`)
- Unity Editor or sollertia-unity-tasks installation (see the sollertia-unity-tasks README)
- Unity Editor relay (`McpBridge`) connectivity diagnostics (see the unity plugin's
  `/unity-mcp-environment-setup`)

---

## Architecture

sollertia-shared-assets exposes a single MCP server through the `slsa mcp` Click subcommand. The
parent `slsa` entry point is registered in `pyproject.toml`:

```toml
[project.scripts]
slsa = "sollertia_shared_assets.interfaces.cli:slsa_cli"
```

- **Server**: `sollertia-shared-assets`
- **CLI command**: `slsa mcp`
- **Purpose**: Discovery, read, write, and schema introspection of *shared* Sollertia configuration
  and runtime data files — the assets consumed by multiple libraries. Configuration and runtime data
  files exclusive to `sollertia-experiment` and `sollertia-forgery` live in those packages and are
  served by their own MCP servers. Also relays Unity Editor operations consumed by the unity
  plugin's skills.

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

The `slsa mcp` server also serves a family of Unity-relay tools (prefab, scene, play-mode,
task-parameters). Those tools depend on the Unity Editor running with the `McpBridge` plugin loaded.
**That diagnostic is owned by the unity plugin's `/unity-mcp-environment-setup`** — this skill only
covers the slsa CLI / Python environment side of the stack.

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

Common version-skew sources are `ataraxis-base-utilities` (must be `>=6,<7`),
`ataraxis-data-structures` (must be `>=6,<7`), and `mcp` (must be `>=1,<2`).

### Step 6: Restart the MCP server

The user must restart the Claude assistant. The assets plugin will automatically reconnect on
the next session.

### Step 7: Hand off if the issue is Unity-specific

If the slsa server itself is healthy but a Unity-relay tool (prefab / scene / play-mode /
task-parameters) returns "Unity Editor is not reachable", hand off to the unity plugin's
`/unity-mcp-environment-setup` — that skill owns the McpBridge / localhost:8090 diagnostic.

---

## Common issues and resolutions

| Symptom                           | Cause                             | Resolution                                      |
|-----------------------------------|-----------------------------------|-------------------------------------------------|
| `slsa: command not found`         | Environment not activated         | Activate conda/venv and restart                 |
| `slsa: command not found`         | `sollertia-shared-assets` missing | Install the package (see Step 3)                |
| Import error on `slsa mcp`        | `ataraxis-data-structures` skew   | Upgrade the package (see Step 5)                |
| `"working directory ... not set"` | Working directory not initialized | Run `/working-directory` to set it              |
| `"task templates ... not set"`    | Task templates path not set       | Run `/working-directory`                        |
| Write tools fail after connect    | Invalid YAML from a previous edit | Use `discover_*` / `read_*` tools               |
| "Unity Editor is not reachable"   | McpBridge / Editor offline        | See unity plugin `/unity-mcp-environment-setup` |

---

## Related skills

This skill is a prerequisite for **every** other skill in the assets plugin and **every** Unity
plugin skill that consumes the `slsa mcp` Unity-relay tools — they all depend on the
`sollertia-shared-assets` MCP server being reachable. The cross-plugin map below covers only the
relationships that are not already captured by that blanket prerequisite.

| Skill                                                 | Relationship                                                         |
|-------------------------------------------------------|----------------------------------------------------------------------|
| unity plugin `/unity-mcp-environment-setup`           | Sibling — owns the McpBridge HTTP-relay diagnostic (Unity side)      |
| experiment plugin `/experiment-mcp-environment-setup` | Peer — equivalent diagnostic for the experiment plugin's MCP servers |
| forging plugin `/forging-mcp-environment-setup`       | Peer — equivalent diagnostic for the forging plugin's MCP server     |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and the `sollertia-shared-assets` MCP server is expected but unavailable
- Any slsa MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-shared-assets MCP server connectivity
- A Unity-relay tool fails and the slsa server itself is suspected (otherwise hand off to the
  unity plugin's `/unity-mcp-environment-setup`)

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
- [ ] Handed off to unity plugin's /unity-mcp-environment-setup if the issue is Unity-specific
```
