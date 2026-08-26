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
- Diagnosing why the `sle` command and its `mcp` subcommand are unavailable
- Checking Python version compatibility
- Validating sollertia-experiment package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows
- The transport selection, tool-module discovery, and response conventions the server applies

**Does not cover:**
- MCP tool usage for Zaber discovery and configuration (see `/zaber-interface`)
- MCP tool usage for the shared session lifecycle (see `/data-management`)
- The CLI and MCP registration seams a new acquisition system adds (see `/library-extension`)
- The sollertia-shared-assets `slsa mcp` server (see `assets:assets-mcp-environment-setup`)
- sollertia-experiment package development workflows

---

## Architecture

sollertia-experiment exposes a single MCP server, started through the `mcp` command of the `sle` Click entry point
declared in the `[project.scripts]` table of `pyproject.toml`:

```toml
[project.scripts]
sle = "sollertia_experiment.interfaces.entry_points:sle_cli"
```

`sle` carries three surfaces, the `get` hardware-agnostic discovery group, one command group per registered acquisition
system, and the `mcp` command that starts the server (`sle_cli` in `interfaces/entry_points.py`). `AcquisitionSystems`
holds a single member, so Mesoscope-VR is currently the only registered acquisition system and supplies the only
per-system group. See `mesoscope:mesoscope-vr` for that system.

`_register_subcommands()` attaches both groups through two hardcoded imports and two `add_command` calls, and it runs at
module import (`interfaces/entry_points.py`). The CLI side carries no discovery mechanism, unlike the tool side
described below, so a new system's group arrives through an edit to that helper.

| Server                 | CLI command | Purpose                                                                     |
|------------------------|-------------|-----------------------------------------------------------------------------|
| `sollertia-experiment` | `sle mcp`   | Every registered tool, both the hardware-agnostic set and each system's set |

The experiment plugin's `plugin.json` configures the Claude assistant to launch the server automatically:

```json
{
  "mcpServers": {
    "sollertia-experiment": { "command": "sle", "args": ["mcp"] }
  }
}
```

The `sle` CLI must be on PATH when the Claude assistant starts. A single `pip install sollertia-experiment` provides the
`sle` command and all of its subcommands.

### Transport selection

`sle mcp` takes one option, `-t/--transport`, a case-insensitive `click.Choice` over `stdio` and `streamable-http` that
defaults to `stdio` (the `mcp` command in `interfaces/entry_points.py`). The `stdio` branch of that command calls
`console.disable()`, because stdio carries the JSON-RPC stream on stdout and a single logged line renders the message
it interleaves with unparsable. The `streamable-http` branch additionally passes `json_response=True` through
`run_server` (`interfaces/mcp_server.py`), which frames each response as one JSON body instead of an event stream.

### Server identity and tool discovery

One shared `MCPServer(name="sollertia-experiment")` is constructed at import of `mcp_instance`, and every tool module
registers on that module-level `mcp` instance (`interfaces/mcp_instance.py`). The MCP dependency is pinned to
`mcp==2.0.0` in the `dependencies` list of `pyproject.toml`.

`_register_tool_modules()` runs at import of `mcp_server`, globs `*_tools.py` inside `interfaces/` in `sorted()` order,
and imports each match (`interfaces/mcp_server.py`). Tools register purely as an import side effect of the
`@mcp.tool()` decorators. The call is `glob` rather than `rglob`, so a tool module sits directly in `interfaces/`, and
the `_tools.py` filename suffix is load-bearing.

The server registers 22 tools, the 7 hardware-agnostic tools of `interfaces/get_tools.py` and the 15 tools of the
current worked example in `interfaces/mesoscope_vr_tools.py`. Each tool name is its function name verbatim,
because `@mcp.tool()` is called with no `name=` argument.

### Dual-distribution model

| Component                                          | Distributed via              | What it provides                                                  |
|----------------------------------------------------|------------------------------|-------------------------------------------------------------------|
| Skills (`/zaber-interface`, `/data-management`, …) | sollertia experiment plugin  | Skill files that guide agents through workflows                   |
| MCP server registration                            | sollertia experiment plugin  | The plugin entry that tells the assistant how to start the server |
| MCP server code (`sle mcp`)                        | sollertia-experiment pip pkg | The actual CLI command and server implementation                  |

Installing the plugin alone registers the MCP server and makes skills available, and the server still fails to start
until the pip package is installed in the active Python environment.

---

## Response contract

Reading a failure correctly separates an environment fault from a tool-level refusal.

- A dict-returning tool signals failure with a single `"error"` key holding a human-readable string, as
  `read_system_configuration_tool` and `validate_system_configuration_tool` do in `interfaces/mesoscope_vr_tools.py`.
- A string-returning tool signals failure with a leading `"Error: "` prefix, as `get_zaber_devices_tool` and
  `get_checksum_tool` do in `interfaces/get_tools.py`.
- A destructive or hardware-mutating tool takes a tri-state confirmation `Literal["yes", "no"] | None` instead of a
  boolean, so a falsy default never reaches the mutation. `set_zaber_device_setting_tool` takes `confirm`
  (`interfaces/get_tools.py`) and `delete_session_tool` takes `confirm_deletion`
  (`interfaces/mesoscope_vr_tools.py`). An omitted confirmation returns an `Error:` string carrying the preview or the
  warning the caller shows the user, which is expected behavior rather than a server fault.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the `sollertia-experiment` MCP server is
connected. If it is connected, the issue is not environmental, so investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which sle
```

`sle` is the single parent binary that owns the `mcp`, `get`, and per-system subcommands. If the binary is not found,
proceed to step 3. The binary should resolve to the active Python environment's `bin/` directory. Server-side import
health is checked in step 5 by hand-launching the server.

### Step 3: Identify the environment type and resolve

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep sollertia-experiment
```

Based on the output, guide the user through the appropriate resolution. The pattern is identical to the ataraxis MCP
environment setup workflow. See `communication:communication-mcp-environment-setup` (ataraxis marketplace) for the full
conda, venv, and uv decision tree. The only substitution is the package name:

```bash
pip install sollertia-experiment
# or
uv pip install sollertia-experiment
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating a conda environment
after the assistant has started leaves the `sle` command unavailable to the MCP server subprocess.

### Step 4: Verify Python version compatibility

```bash
python --version
```

sollertia-experiment requires Python `>=3.14,<3.15` (the `requires-python` field of `pyproject.toml`). If the Python
version does not match, instruct the user to create or activate an environment with a compatible version.

### Step 5: Verify package integrity

```bash
sle --help
sle mcp -t streamable-http
```

`sle --help` proves the CLI package imports. It does NOT prove the MCP server starts, because `sle mcp` defers
`from .mcp_server import run_server` into the command body, which `--help` never reaches. `sle mcp -t streamable-http`
is the real smoke test: the `streamable-http` branch echoes
`Starting the sollertia-experiment MCP server with the streamable-http transport.` (`interfaces/entry_points.py`) and
then blocks. A startup line followed by a blocking process means the server is healthy and the fault lies in the
assistant's launch environment. A traceback instead of the startup line means a broken dependency. Tell the user to
interrupt it with Ctrl+C. Use `streamable-http` rather than the `stdio` default, because a silent `stdio` server is
indistinguishable from a hung one.

If either command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check 2>&1 | head -20
```

Report any missing or incompatible dependencies to the user. Every runtime dependency is pinned to an exact version, so
a sibling library upgraded on its own surfaces here. Common causes include version skew with `sollertia-shared-assets`,
`ataraxis-base-utilities`, or `ataraxis-data-structures`. Never name a pinned sibling alongside `sollertia-experiment`
in an `--upgrade` invocation. The exact pin means upgrading `sollertia-experiment` alone installs the sibling version
it requires, while naming the sibling asks pip to break that pin and can backtrack `sollertia-experiment` to an older
release.

### Step 6: Restart the MCP server

After resolving the environment issue, the user must restart the Claude assistant. The experiment plugin reconnects the
server on the next session.

---

## Common issues and resolutions

| Symptom                                                                   | Cause                                                        | Resolution                                                     |
|---------------------------------------------------------------------------|--------------------------------------------------------------|----------------------------------------------------------------|
| `sle: command not found`                                                  | Environment not activated                                    | Activate conda/venv, restart the assistant                     |
| `sle: command not found`                                                  | sollertia-experiment not installed                           | `pip install sollertia-experiment` in the active environment   |
| Import error on `sle mcp`                                                 | Version skew with sollertia-shared-assets                    | `pip install --upgrade --force-reinstall sollertia-experiment` |
| Python version mismatch                                                   | Wrong environment activated                                  | Activate environment with Python >=3.14,<3.15                  |
| Tool error: "working directory ... has not been set"                      | `slsa` working directory not initialized                     | Run `assets:working-directory` from the assets plugin          |
| Tool error: "Expected exactly one '*_system_configuration.yaml'"          | Host not bound to an acquisition system, or bound to several | Run `sle <system> configure system`                            |
| Tool error: "the host-machine belongs to the ... data acquisition system" | Host bound to a different acquisition system                 | Run `sle <system> configure system` to rebind                  |
| Tool fails with Zaber connection error                                    | Not an environment issue                                     | Check `/zaber-interface` for hardware troubleshooting          |

In the two configuration rows, `<system>` is the host's `AcquisitionSystems` value. Resolve it to its owning skill
through the Supported acquisition systems registry in `/acquisition-system-setup`, which this skill does not duplicate.

Two module-level preambles run before the `click` import and shape what an operator sees. `warnings.warn` and
`warnings.warn_explicit` are monkeypatched to no-ops, so dependency deprecation warnings raised during the import phase
stay silent (`interfaces/entry_points.py`). `QT_LOGGING_RULES` is set through `setdefault` before any subprocess
spawns, so every child inherits it and a value the operator exported survives (`interfaces/entry_points.py`).
Treat `pip check` as the way to surface a dependency problem, because an import-time deprecation warning stays hidden.

---

## Related skills

| Skill                                 | Relationship                                                                             |
|---------------------------------------|------------------------------------------------------------------------------------------|
| `/zaber-interface`                    | Requires the `sollertia-experiment` MCP for device discovery and settings                |
| `/data-management`                    | Requires the `sollertia-experiment` MCP for the shared session lifecycle primitives      |
| `/library-extension`                  | Owns the CLI and MCP registration seams a new acquisition system adds to this server     |
| `/system-health-check`                | Uses the server as part of the pre-session validation sweep                              |
| `/pipeline`                           | Orchestrates all phases that depend on MCP server connectivity                           |
| `mesoscope:mesoscope-vr`              | The current worked example, whose tools this server registers alongside the agnostic set |
| `mesoscope:mesoscope-vr-snapshots`    | Requires the `sollertia-experiment` MCP for position snapshot read and write             |
| `assets:assets-mcp-environment-setup` | Equivalent diagnostic for the `slsa mcp` server                                          |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and `sollertia-experiment` MCP tools are expected but unavailable
- Any `sle` MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-experiment MCP server connectivity

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

sollertia-experiment MCP environment setup:
- [ ] Checked sollertia-experiment server connection status
- [ ] Verified 'sle' command is on PATH
- [ ] Hand-launched 'sle mcp -t streamable-http' and saw the startup line
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Distinguished a tool-level refusal carrying 'error' or 'Error: ' from a server connectivity fault
- [ ] Informed user that the assistant must be restarted after environment changes
```
