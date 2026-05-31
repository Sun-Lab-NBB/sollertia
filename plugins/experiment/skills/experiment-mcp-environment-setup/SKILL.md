---
name: experiment-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-experiment MCP server connectivity issues (environment,
  `sle mcp` command availability, Python version, dependencies). Use when the experiment
  MCP tools are unavailable, the server fails to start, or a new session needs
  sollertia-experiment tools.
user-invocable: false
---

# Sollertia experiment MCP environment setup

Diagnoses and resolves sollertia-experiment MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the `sollertia-experiment` MCP server is reachable
- Diagnosing why the `sle` command (and its `mcp` subcommand) is unavailable
- Checking Python version compatibility
- Validating sollertia-experiment package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows

**Does not cover:**
- MCP tool usage for Zaber discovery/configuration (see `/zaber-interface`)
- MCP tool usage for session preprocessing/migration (see `/data-management`)
- sollertia-shared-assets `slsa mcp` server (see the assets plugin's
  `/assets-mcp-environment-setup`)
- sollertia-experiment package development workflows

---

## Architecture

sollertia-experiment exposes a **single** MCP server, reached via the `mcp` subcommand of the
Click CLI entry point defined in `pyproject.toml`:

```toml
[project.scripts]
sle = "sollertia_experiment.interfaces.entry_points:sle_cli"
```

`sle` is a Click group with three children: the `mcp` command, the `get` command group
(hardware-agnostic discovery), and the `mesoscope` command group (Mesoscope-VR configuration,
runtime, and session management). The single `sle mcp` server exposes the tools that back **both**
the `get` and `mesoscope` groups to AI agents — there is no longer a separate server per group.

| Server                  | CLI command | Purpose                                                                                  |
|-------------------------|-------------|------------------------------------------------------------------------------------------|
| `sollertia-experiment`  | `sle mcp`   | All experiment tools: Zaber discovery/config, mount checks, system configuration read/write/validate, position snapshots, session preprocess/delete/migrate, CRC checksums |

The server accepts a `--transport` option (defaults to `stdio`). The experiment plugin's `plugin.json`
configures the Claude assistant to launch it automatically:

```json
{
  "mcpServers": {
    "sollertia-experiment": { "command": "sle", "args": ["mcp"] }
  }
}
```

The `sle` CLI must be on PATH when the Claude assistant starts. A single
`pip install sollertia-experiment` provides the `sle` command and all of its subcommands.

### Dual-distribution model

| Component                                          | Distributed via              | What it provides                                       |
|----------------------------------------------------|------------------------------|--------------------------------------------------------|
| Skills (`/zaber-interface`, `/data-management`, …) | sollertia experiment plugin  | Skill files that guide agents through workflows       |
| MCP server registration                            | sollertia experiment plugin  | The plugin entry that tells the assistant how to start the server |
| MCP server code (`sle mcp`)                        | sollertia-experiment pip pkg | The actual CLI command and server implementation     |

Installing the plugin alone registers the MCP server and makes skills available, but the server will
fail to start because the CLI command is not present. The pip package must also be installed in the
active Python environment.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the `sollertia-experiment`
MCP server is connected. If it is connected, the issue is not environmental — investigate
tool-specific errors instead.

### Step 2: Verify command availability

```bash
which sle
```

`sle` is the single parent binary that owns the `mcp`, `get`, and `mesoscope` subcommands. If the
binary is not found, proceed to step 3. The binary should resolve to the active Python environment's
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
conda environment after the assistant has started does not make the `sle` command available to the
MCP server subprocess.

### Step 4: Verify Python version compatibility

```bash
python --version
```

sollertia-experiment requires Python `>=3.14,<3.15`. If the Python version does not match, instruct the
user to create or activate an environment with a compatible version.

### Step 5: Verify package integrity

```bash
sle --help
sle mcp --help
```

If either command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check sollertia-experiment 2>&1 | head -20
```

Report any missing or incompatible dependencies to the user. Common causes include version skew with
`sollertia-shared-assets`, `ataraxis-base-utilities`, or `ataraxis-data-structures`.

### Step 6: Restart the MCP server

After resolving the environment issue, the user must restart the Claude assistant. The experiment
plugin will automatically reconnect the server on the next session.

---

## Common issues and resolutions

| Symptom                                  | Cause                                           | Resolution                                                          |
|------------------------------------------|-------------------------------------------------|---------------------------------------------------------------------|
| `sle: command not found`                 | Environment not activated                       | Activate conda/venv, restart the assistant                         |
| `sle: command not found`                 | sollertia-experiment not installed              | `pip install sollertia-experiment` in the active environment       |
| Import error on `sle mcp`                | Version skew with sollertia-shared-assets       | `pip install --upgrade sollertia-experiment sollertia-shared-assets` |
| Python version mismatch                  | Wrong environment activated                     | Activate environment with Python >=3.14,<3.15                      |
| Tools fail with "no system configuration"| `slsa` working directory not initialized        | Run `/working-directory` from the assets plugin                    |
| Tool fails with Zaber connection error   | Not an environment issue                        | Check `/zaber-interface` for hardware troubleshooting              |

---

## Related skills

| Skill                                       | Relationship                                                       |
|---------------------------------------------|--------------------------------------------------------------------|
| `/zaber-interface`                          | Requires the `sollertia-experiment` MCP for device discovery and settings |
| `/data-management`                          | Requires the `sollertia-experiment` MCP for preprocess/delete/migrate |
| `/session-snapshots`                        | Requires the `sollertia-experiment` MCP for position snapshot read/write |
| `/mesoscope-vr`                             | Requires the `sollertia-experiment` MCP for system configuration authoring |
| `/system-health-check`                      | Uses the server as part of the pre-session validation sweep        |
| `/pipeline`                                 | Orchestrates all phases that depend on MCP server connectivity     |
| assets plugin `/assets-mcp-environment-setup` | Equivalent diagnostic for the `slsa mcp` server                  |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and `sollertia-experiment` MCP tools are expected but unavailable
- Any `sle` MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-experiment MCP server connectivity

---

## Verification checklist

```text
sollertia-experiment MCP environment setup:
- [ ] Checked sollertia-experiment server connection status
- [ ] Verified 'sle' command is on PATH
- [ ] Verified 'sle mcp' starts without import errors
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that the assistant must be restarted after environment changes
```
