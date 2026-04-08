---
name: experiment-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-experiment MCP server connectivity issues. Covers environment
  verification, command availability for sl-get and sl-manage, Python version checks, dependency
  validation, and conda/pip/uv environment configuration. Use when sl-experiment MCP tools are
  unavailable, when the server fails to start, or when starting a session that requires sl-get or
  sl-manage MCP tools.
user-invocable: true
---

# Sollertia experiment MCP environment setup

Diagnoses and resolves sollertia-experiment MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the `sl-experiment-get` and `sl-experiment-manage` MCP servers are reachable
- Diagnosing why the `sl-get` or `sl-manage` commands are unavailable
- Checking Python version compatibility
- Validating sollertia-experiment package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows

**Does not cover:**
- MCP tool usage for Zaber discovery/configuration (see `/zaber-interface`)
- MCP tool usage for session preprocessing/migration (see `/data-management`)
- sollertia-shared-assets `sl-configure mcp` server (see the configuration plugin's
  `/configuration-mcp-environment-setup`)
- sollertia-experiment package development workflows

---

## Architecture

sollertia-experiment exposes **two** MCP servers, both via the Click CLI entry points defined in
`pyproject.toml`:

```toml
[project.scripts]
sl-get = "sl_experiment.command_line_interfaces.entry_points:get_cli"
sl-manage = "sl_experiment.command_line_interfaces.entry_points:manage_cli"
```

| Server                  | CLI command   | Purpose                                                                              |
|-------------------------|---------------|--------------------------------------------------------------------------------------|
| `sl-experiment-get`     | `sl-get mcp`  | Read-only tools: Zaber device discovery, settings inspection, CRC checksums         |
| `sl-experiment-manage`  | `sl-manage mcp` | Mutating tools: Zaber config, session preprocessing, deletion, animal migration   |

Both servers accept a `--transport` option (defaults to `stdio`). The experiment plugin's `plugin.json`
configures the Claude assistant to launch both automatically:

```json
{
  "mcpServers": {
    "sl-experiment-get": { "command": "sl-get", "args": ["mcp"] },
    "sl-experiment-manage": { "command": "sl-manage", "args": ["mcp"] }
  }
}
```

Both `sl-get` and `sl-manage` must be on PATH when the Claude assistant starts. They are installed by
the same `sollertia-experiment` (formerly `sl-experiment`) Python package, so a single
`pip install sollertia-experiment` provides both.

### Dual-distribution model

| Component                                          | Distributed via              | What it provides                                       |
|----------------------------------------------------|------------------------------|--------------------------------------------------------|
| Skills (`/zaber-interface`, `/data-management`, …) | sollertia experiment plugin  | Skill files that guide agents through workflows       |
| MCP server registrations                           | sollertia experiment plugin  | Plugin entries that tell the assistant how to start the servers |
| MCP server code (`sl-get mcp`, `sl-manage mcp`)    | sollertia-experiment pip pkg | The actual CLI commands and server implementation     |

Installing the plugin alone registers the MCP servers and makes skills available, but the servers will
fail to start because the CLI commands are not present. The pip package must also be installed in the
active Python environment.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the `sl-experiment-get`
and/or `sl-experiment-manage` MCP servers are connected. If both are connected, the issue is not
environmental — investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which sl-get
which sl-manage
```

If either command is not found, proceed to step 3. Both should resolve to the same Python environment's
`bin/` directory.

### Step 3: Identify the environment type and resolve

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep -E "sollertia-experiment|sl-experiment"
```

Based on the output, guide the user through the appropriate resolution. The pattern is identical to
the ataraxis MCP environment setup workflow — see `ataraxis@communication:mcp-environment-setup` for
the full conda / venv / uv decision tree. The only substitution is the package name:

```bash
pip install sollertia-experiment
# or
uv pip install sollertia-experiment
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating a
conda environment after the assistant has started does not make the `sl-get` / `sl-manage` commands
available to MCP server subprocesses.

### Step 4: Verify Python version compatibility

```bash
python --version
```

sollertia-experiment requires Python `>=3.14,<3.15`. If the Python version does not match, instruct the
user to create or activate an environment with a compatible version.

### Step 5: Verify package integrity

```bash
sl-get --help
sl-manage --help
```

If either command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check sollertia-experiment 2>&1 | head -20
```

Report any missing or incompatible dependencies to the user. Common causes include version skew with
`sollertia-shared-assets`, `ataraxis-base-utilities`, or `ataraxis-data-structures`.

### Step 6: Restart the MCP servers

After resolving the environment issue, the user must restart the Claude assistant. The experiment
plugin will automatically reconnect both servers on the next session.

---

## Common issues and resolutions

| Symptom                                  | Cause                                           | Resolution                                                          |
|------------------------------------------|-------------------------------------------------|---------------------------------------------------------------------|
| `sl-get: command not found`              | Environment not activated                       | Activate conda/venv, restart the assistant                         |
| `sl-get: command not found`              | sollertia-experiment not installed              | `pip install sollertia-experiment` in the active environment       |
| Only one of two servers connects         | Plugin manifest stale                           | Update the experiment plugin from the sollertia marketplace        |
| Import error on `sl-get mcp`             | Version skew with sollertia-shared-assets       | `pip install --upgrade sollertia-experiment sollertia-shared-assets` |
| Python version mismatch                  | Wrong environment activated                     | Activate environment with Python >=3.14,<3.15                      |
| Tools fail with "no system configuration"| `sl-configure` working directory not initialized| Run the working-directory skill from the configuration plugin      |
| Tool fails with Zaber connection error   | Not an environment issue                        | Check `/zaber-interface` for hardware troubleshooting              |

---

## Related skills

| Skill                                       | Relationship                                                       |
|---------------------------------------------|--------------------------------------------------------------------|
| `/zaber-interface`                          | Requires `sl-experiment-get` MCP for device discovery and settings |
| `/data-management`                          | Requires `sl-experiment-manage` MCP for preprocess/delete/migrate  |
| `/system-health-check`                      | Uses both servers as part of the pre-session validation sweep      |
| `/pipeline`                                 | Orchestrates all phases that depend on MCP server connectivity     |
| configuration plugin's MCP env setup        | Equivalent diagnostic for the `sl-configure mcp` server            |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and `sl-experiment-get` or `sl-experiment-manage` MCP tools are expected but unavailable
- Any sl-get or sl-manage MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-experiment MCP server connectivity

---

## Verification checklist

```text
sollertia-experiment MCP environment setup:
- [ ] Checked sl-experiment-get connection status
- [ ] Checked sl-experiment-manage connection status
- [ ] Verified 'sl-get' command is on PATH
- [ ] Verified 'sl-manage' command is on PATH
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that the assistant must be restarted after environment changes
```
