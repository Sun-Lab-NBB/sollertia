---
name: experiment-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-experiment MCP server connectivity issues. Covers environment
  verification, command availability for sle get and sle manage, Python version checks, dependency
  validation, and conda/pip/uv environment configuration. Use when sollertia-experiment MCP tools are
  unavailable, when the server fails to start, or when starting a session that requires sle get or
  sle manage MCP tools.
user-invocable: true
---

# Sollertia experiment MCP environment setup

Diagnoses and resolves sollertia-experiment MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the `sollertia-experiment-get` and `sollertia-experiment-manage` MCP servers are reachable
- Diagnosing why the `sle get` or `sle manage` commands are unavailable
- Checking Python version compatibility
- Validating sollertia-experiment package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows

**Does not cover:**
- MCP tool usage for Zaber discovery/configuration (see `/zaber-interface`)
- MCP tool usage for session preprocessing/migration (see `/data-management`)
- sollertia-shared-assets `slsa mcp` server (see the assets plugin's
  `/experiment-mcp-environment-setup`)
- sollertia-experiment package development workflows

---

## Architecture

sollertia-experiment exposes **two** MCP servers, both reached via subcommands of the single
Click CLI entry point defined in `pyproject.toml`:

```toml
[project.scripts]
sle = "sl_experiment.command_line_interfaces.entry_points:cli"
```

`get` and `manage` are subcommands of the `sle` Click group; `mcp` is in turn a subcommand of
each, so the full server invocations are `sle get mcp` and `sle manage mcp`.

| Server                  | CLI command   | Purpose                                                                              |
|-------------------------|---------------|--------------------------------------------------------------------------------------|
| `sollertia-experiment-get`     | `sle get mcp`  | Read-only tools: Zaber device discovery, settings inspection, CRC checksums         |
| `sollertia-experiment-manage`  | `sle manage mcp` | Mutating tools: Zaber config, session preprocessing, deletion, animal migration   |

Both servers accept a `--transport` option (defaults to `stdio`). The experiment plugin's `plugin.json`
configures the Claude assistant to launch both automatically:

```json
{
  "mcpServers": {
    "sollertia-experiment-get": { "command": "sle", "args": ["get", "mcp"] },
    "sollertia-experiment-manage": { "command": "sle", "args": ["manage", "mcp"] }
  }
}
```

The `sle` CLI must be on PATH when the Claude assistant starts. Both `get` and `manage` are
subcommands of the same `sollertia-experiment` Python package, so a single
`pip install sollertia-experiment` provides them.

### Dual-distribution model

| Component                                          | Distributed via              | What it provides                                       |
|----------------------------------------------------|------------------------------|--------------------------------------------------------|
| Skills (`/zaber-interface`, `/data-management`, …) | sollertia experiment plugin  | Skill files that guide agents through workflows       |
| MCP server registrations                           | sollertia experiment plugin  | Plugin entries that tell the assistant how to start the servers |
| MCP server code (`sle get mcp`, `sle manage mcp`)    | sollertia-experiment pip pkg | The actual CLI commands and server implementation     |

Installing the plugin alone registers the MCP servers and makes skills available, but the servers will
fail to start because the CLI commands are not present. The pip package must also be installed in the
active Python environment.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the `sollertia-experiment-get`
and/or `sollertia-experiment-manage` MCP servers are connected. If both are connected, the issue is not
environmental — investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which sle
```

`sle` is the single parent binary that owns both `get` and `manage` subcommands. If the binary
is not found, proceed to step 3. The binary should resolve to the active Python environment's
`bin/` directory. (Subcommand-level health is checked in step 5 via `--help` invocations.)

### Step 3: Identify the environment type and resolve

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep sollertia-experiment
```

Based on the output, guide the user through the appropriate resolution. The pattern is identical to
the ataraxis MCP environment setup workflow — see `ataraxis@communication:communication-mcp-environment-setup` for
the full conda / venv / uv decision tree. The only substitution is the package name:

```bash
pip install sollertia-experiment
# or
uv pip install sollertia-experiment
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating a
conda environment after the assistant has started does not make the `sle get` / `sle manage` commands
available to MCP server subprocesses.

### Step 4: Verify Python version compatibility

```bash
python --version
```

sollertia-experiment requires Python `>=3.14,<3.15`. If the Python version does not match, instruct the
user to create or activate an environment with a compatible version.

### Step 5: Verify package integrity

```bash
sle get --help
sle manage --help
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
| `sle: command not found`                 | Environment not activated                       | Activate conda/venv, restart the assistant                         |
| `sle: command not found`                 | sollertia-experiment not installed              | `pip install sollertia-experiment` in the active environment       |
| Only one of two servers connects         | Plugin manifest stale                           | Update the experiment plugin from the sollertia marketplace        |
| Import error on `sle get mcp`             | Version skew with sollertia-shared-assets       | `pip install --upgrade sollertia-experiment sollertia-shared-assets` |
| Python version mismatch                  | Wrong environment activated                     | Activate environment with Python >=3.14,<3.15                      |
| Tools fail with "no system configuration"| `slsa` working directory not initialized| Run the working-directory skill from the assets plugin      |
| Tool fails with Zaber connection error   | Not an environment issue                        | Check `/zaber-interface` for hardware troubleshooting              |

---

## Related skills

| Skill                                       | Relationship                                                       |
|---------------------------------------------|--------------------------------------------------------------------|
| `/zaber-interface`                          | Requires `sollertia-experiment-get` MCP for device discovery and settings |
| `/data-management`                          | Requires `sollertia-experiment-manage` MCP for preprocess/delete/migrate  |
| `/system-health-check`                      | Uses both servers as part of the pre-session validation sweep      |
| `/pipeline`                                 | Orchestrates all phases that depend on MCP server connectivity     |
| assets plugin's MCP env setup        | Equivalent diagnostic for the `slsa mcp` server            |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and `sollertia-experiment-get` or `sollertia-experiment-manage` MCP tools are expected but unavailable
- Any sle get or sle manage MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-experiment MCP server connectivity

---

## Verification checklist

```text
sollertia-experiment MCP environment setup:
- [ ] Checked sollertia-experiment-get connection status
- [ ] Checked sollertia-experiment-manage connection status
- [ ] Verified 'sle get' command is on PATH
- [ ] Verified 'sle manage' command is on PATH
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that the assistant must be restarted after environment changes
```
