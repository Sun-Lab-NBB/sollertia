---
name: configuration-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-shared-assets MCP server connectivity issues. Covers environment
  verification, command availability for sl-configure, Python version checks, dependency validation, and
  conda/pip/uv environment configuration. Use when sollertia-shared-assets MCP tools are unavailable, when
  the server fails to start, or when starting a session that requires the configuration MCP tools.
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

**Does not cover:**
- MCP tool usage for any specific configuration task (see other configuration plugin skills)
- sollertia-experiment `sl-get` / `sl-manage` MCP servers (see the experiment plugin's MCP env setup)

---

## Architecture

sollertia-shared-assets exposes a single MCP server through the `sl-configure mcp` Click subcommand
defined in `pyproject.toml`:

```toml
[project.scripts]
sl-configure = "sollertia_shared_assets.interfaces.configure:configure"
```

| Server                       | CLI command         | Purpose                                                              |
|------------------------------|---------------------|----------------------------------------------------------------------|
| `sollertia-shared-assets`    | `sl-configure mcp`  | Discovery, read, write, and schema introspection of all Sollertia configuration and runtime data files |

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

| Component                                | Distributed via                  | What it provides                                  |
|------------------------------------------|----------------------------------|---------------------------------------------------|
| Skills (`/working-directory`, etc.)      | sollertia configuration plugin   | Skill files that guide agents through workflows  |
| MCP server registration                  | sollertia configuration plugin   | Plugin entry that tells the assistant how to start the server |
| MCP server code (`sl-configure mcp`)     | sollertia-shared-assets pip pkg  | The actual CLI command and server implementation |

Installing the plugin alone registers the MCP server but the server will fail to start if
`sollertia-shared-assets` is not installed in the active Python environment.

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

---

## Common issues and resolutions

| Symptom                                         | Cause                                         | Resolution                                                  |
|-------------------------------------------------|-----------------------------------------------|-------------------------------------------------------------|
| `sl-configure: command not found`               | Environment not activated                     | Activate conda/venv, restart the assistant                  |
| `sl-configure: command not found`               | sollertia-shared-assets not installed         | `pip install sollertia-shared-assets`                       |
| Import error on `sl-configure mcp`              | ataraxis-data-structures version skew         | `pip install --upgrade sollertia-shared-assets`             |
| Tools fail with "no working directory"          | Working directory not initialized             | Use `/working-directory` to set it                          |
| Tools fail with "task templates directory not set" | Task templates path not configured         | Use `/working-directory` to set the templates path          |
| MCP server connected but write tools fail       | Invalid YAML schema produced by previous edit | Use `discover_*` and `read_*` tools to inspect actual state |

---

## Related skills

This skill is a prerequisite for **every** other skill in the configuration plugin — they all depend
on the `sollertia-shared-assets` MCP server being reachable.

| Skill                          | Asset it owns                                                  |
|--------------------------------|----------------------------------------------------------------|
| `/working-directory`           | Working directory, Google credentials, task templates dir path |
| `/system-configuration`        | `MesoscopeSystemConfiguration`                                 |
| `/server-configuration`        | `ServerConfiguration`                                          |
| `/task-templates`              | `TaskTemplate` + trial primitives                              |
| `/experiment-configuration`    | `MesoscopeExperimentConfiguration` + `ExperimentState`         |
| `/project-hierarchy`           | Projects (`create_project_tool`)                               |
| `/session-data`                | `SessionData` + `SessionTypes`                                 |
| `/session-descriptors`         | The 4 per-session-type descriptors                             |
| `/session-snapshots`           | Frozen `MesoscopeHardwareState` / `ZaberPositions` / `MesoscopePositions` |
| `/subject-metadata`            | `SubjectData` / `SurgeryData` / `ImplantData` / `InjectionData` / `DrugData` |
| `/datasets`                    | `DatasetData` / `DatasetSession`                               |
| experiment plugin `/mcp-environment-setup` | Equivalent diagnostic for `sl-get` / `sl-manage` MCP servers |

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
```
