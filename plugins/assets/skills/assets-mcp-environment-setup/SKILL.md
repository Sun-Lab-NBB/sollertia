---
name: assets-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-shared-assets MCP server connectivity issues (environment,
  `slsa` command availability, Python version, dependencies). Use when the MCP tools are
  unavailable, the server fails to start, or a new session needs the assets plugin's tools.
user-invocable: false
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
- sollertia-experiment MCP server (see `experiment:experiment-mcp-environment-setup`)
- sollertia-forgery MCP server (see `forging:forging-mcp-environment-setup`)
- Unity Editor or sollertia-virtual-reality installation (see the sollertia-virtual-reality README)
- Unity Editor relay (`McpBridge`) connectivity diagnostics (see the unity plugin's `unity:unity-mcp-environment-setup`)

---

## Architecture

sollertia-shared-assets exposes a single MCP server through the `slsa mcp` Click subcommand. The parent `slsa` entry
point is registered in `pyproject.toml`:

```toml
[project.scripts]
slsa = "sollertia_shared_assets.interfaces.cli:slsa_cli"
```

- **Server**: `sollertia-shared-assets`
- **CLI command**: `slsa mcp`
- **Purpose**: Discovery, read, write, and schema introspection of *shared* Sollertia configuration and runtime data
  files, meaning the assets consumed by multiple libraries. Configuration and runtime data files exclusive to
  `sollertia-experiment` and `sollertia-forgery` live in those packages and are served by their own MCP servers. This
  server also relays Unity Editor operations consumed by the unity plugin's skills.

The server accepts a `--transport` option, short form `-t`, whose two case-insensitive choices are `stdio` and
`streamable-http` and whose default is `stdio`. Under `stdio` the command calls `console.disable()`, because the
JSON-RPC message stream shares stdout with the console, so a manual `slsa mcp` smoke test prints nothing at all on
success. Run `slsa mcp -t streamable-http` instead when a visible confirmation is needed, and abort the command once its
startup line appears. The `streamable-http` transport runs with `json_response=True` and leaves the host, port, and path
at the mcp SDK defaults.

The assets plugin's `plugin.json` configures the Claude assistant to launch the server automatically:

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

| Component                                 | Distributed via                 | What it provides                          |
|-------------------------------------------|---------------------------------|-------------------------------------------|
| Skills (`assets:working-directory`, etc.) | sollertia assets plugin         | Workflow-guiding skill files              |
| MCP server registration                   | sollertia assets plugin         | Plugin entry that launches the server     |
| MCP server code (`slsa mcp`)              | sollertia-shared-assets pip pkg | The CLI command and server implementation |

Installing the plugin alone registers the MCP server, and the server fails to start unless `sollertia-shared-assets` is
installed in the active Python environment.

### Unity tools

The `slsa mcp` server also serves 15 Unity-relay tools, spanning seven families: task creation and deletion, prefab
inspection and zone cloning, asset listing and deletion, scene listing, opening, and inspection, play mode, task
parameters, and monitor refresh. Those tools depend on the Unity Editor running with the `McpBridge` plugin loaded.
**That diagnostic is owned by `unity:unity-mcp-environment-setup`.** This skill covers the slsa CLI and Python
environment side of the stack alone.

---

## Response contract

This section is the plugin-wide contract for every `slsa mcp` tool. Every other skill that calls one of those tools
references this section by pointer and MUST NOT restate it.

### Response envelope

Every tool on the `slsa mcp` server returns a `dict` and never raises. A successful call returns
`{"success": true, ...payload}` with the payload keys at the top level, and a failed call returns
`{"success": false, "error": "<message>"}`. Branch on `response["success"]` before reading any other key. Two tools
report a verdict rather than a failure. `validate_template_tool` and `validate_experiment_configuration_tool` return a
success envelope carrying `valid: false` and a single-element `issues` list when the document loads but fails
validation, and the error envelope only when the file does not exist. A batch tool can likewise return `success: true`
while individual items inside it failed, `inspect_sessions_tool` through per-session `status: "error"` reports and
`filter_sessions_tool` through `invalid_entries`.

### What a write tool validates

A `write_*` tool dumps the payload to a temporary sibling file, loads it back through the target dataclass, re-runs
`__post_init__` when the class defines one, and only then writes the destination. The loader runs with per-field type
checking disabled and does not reject unrecognized keys, so three things pass silently. A wrong-typed value is persisted
as written, a misspelled or unrecognized key is dropped, and a field omitted from the payload is written back at its
dataclass default. The rejections that do fire are a malformed payload, a field that the class requires and leaves
without a default, and a `__post_init__` raise. `TaskTemplate`, `Cue`, `TrialStructure`, `VREnvironment`, `SessionData`,
`MesoscopeExperimentConfiguration`, `MesoscopeWaterRewardTrial`, `MesoscopeGasPuffTrial`, and `DatasetData` define
`__post_init__` and therefore validate semantically. The four session descriptors, `MesoscopeHardwareState`,
`ExperimentState`, and `SurgeryData` define none, so for those the write is a shape check only. Because omissions are
silent, every amendment MUST be a read-mutate-write of the complete record, and the caller MUST re-read and diff the
result against the intended payload before reporting success. The tool also creates any missing parent directories, so a
mistyped `file_path` writes a stray file into a newly created tree instead of failing.

### Schema payload shape

Every `describe_*_schema_tool` returns its schema in one shape:
`{"class": "<Name>", "fields": {"<field>": {"type": "<annotation>", "default": <value>} }}`. A field with no default and
no working `default_factory` carries `"required": true` instead of `"default"`. A field whose type is itself a dataclass
carries a `"nested"` sub-schema of the same shape. A type already on the recursion path is rendered
`{"class": "<Name>", "recursive_reference": true}` with no `fields` key. Read `required` off this payload to learn which
fields a write payload must carry.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools. If the `sollertia-shared-assets` server is connected, the issue
lies outside the environment, so investigate tool-specific errors instead.

### Step 2: Read the platform environment status

Once the server connects, call `get_platform_environment_status_tool()` once. It returns `overall_ok` together with a
`components` map covering `working_directory` (`required: true`), `data_root`, `task_templates_directory`, and one
`<credentials_type>_credentials` entry per credentials category. Every component carries `required`, `configured`, `ok`,
and either `path` when it is configured or `error` when it is not. `overall_ok` reflects the required components only,
which today is `working_directory` alone, so an `overall_ok` of `true` does not mean the optional paths are set. The
tool never returns an error envelope. Route a false `ok` on any component to `assets:working-directory`, which owns the
field-level detail and the repair procedure.

### Step 3: Verify command availability

```bash
which slsa
```

### Step 4: Identify the environment type and resolve

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep sollertia-shared-assets
```

The conda, venv, and uv decision tree is identical to the other Sollertia and ataraxis MCP environment setup skills. The
package to install is:

```bash
pip install sollertia-shared-assets
# or
uv pip install sollertia-shared-assets
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating an environment
after the assistant has started does not make `slsa` available to the MCP subprocess.

### Step 5: Verify Python version compatibility

sollertia-shared-assets requires Python `>=3.14,<3.15`. If the Python version does not match, instruct the user to
create or activate a compatible environment.

### Step 6: Verify package integrity

```bash
slsa --help
```

If the command fails with an import error:

```bash
pip check sollertia-shared-assets 2>&1 | head -20
```

The dependency bounds a skewed environment violates:

| Dependency                 | Required bound | Triage note                                                                |
|----------------------------|----------------|----------------------------------------------------------------------------|
| `mcp`                      | `>=2,<3`       | Check this one first, because mcp 1.x breaks the import outright           |
| `ataraxis-data-structures` | `>=7.1,<8`     | 7.1 is the first release exporting `atomic_write` and `index_marker_files` |
| `ataraxis-base-utilities`  | `>=7,<8`       | Supplies `console`, `LogLevel`, and `ensure_directory_exists`              |
| `ataraxis-time`            | `>=7,<8`       | Supplies `get_timestamp` and `TimestampFormats` used by the data hierarchy |

An environment holding mcp 1.x makes `slsa mcp` die at import with
`ImportError: cannot import name 'MCPServer' from 'mcp.server'`, because the server module imports `MCPServer` and mcp
1.x named that class `FastMCP`. Remedy that skew with `pip install --upgrade 'mcp>=2,<3'`. The
`ataraxis-data-structures` floor is `7.1` rather than `7` because 7.1 is the first release exporting `atomic_write` and
`index_marker_files`, both of which the library calls.

Tool registration is a pure import side effect of the four `*_tools.py` modules, which the server module globs in
`sorted()` order at import time. A failure inside any one of them therefore takes down all 66 tools rather than a
subset. The corollary matters for triage: "some slsa tools are present and others are missing" is never an environment
fault, so investigate the tool names and the plugin registration instead.

`slsa --help` hides the traceback behind a generic failure. Reproduce it directly:

```bash
python -c "import sollertia_shared_assets.interfaces.mcp_server"
```

That command also surfaces the second start-time failure: a `RuntimeError` raised by the import-time registry contract
checks after an incomplete acquisition-system extension. Hand that case off to `assets:library-extension`, which owns
the registry contracts and the extension procedure.

### Step 7: Restart the MCP server

The user must restart the Claude assistant. The assets plugin reconnects automatically on the next session.

### Step 8: Hand off if the issue is Unity-specific

If the slsa server itself is healthy but a Unity-relay tool returns
`Unable to reach the Unity Editor at http://localhost:8090/`, hand off to the unity plugin's
`unity:unity-mcp-environment-setup`, which owns the McpBridge and localhost:8090 diagnostic.

A Unity-relay tool that instead reports that
`The Editor accepted the connection but did not answer within 30 seconds or dropped it mid-response` is a different
failure and MUST NOT be handed off. The bridge is reachable and the Editor main thread is busy with a long operation
such as a domain reload or an asset import, so the correct action is to wait for that operation to finish and retry the
call.

---

## Common issues and resolutions

| Symptom                               | Cause                                    | Resolution                                 |
|---------------------------------------|------------------------------------------|--------------------------------------------|
| `slsa: command not found`             | Environment not activated                | Activate conda/venv and restart            |
| `slsa: command not found`             | `sollertia-shared-assets` missing        | Install the package (see Step 4)           |
| `cannot import name 'MCPServer'`      | mcp 1.x in the active environment        | `pip install --upgrade 'mcp>=2,<3'`        |
| Import error on `slsa mcp`            | An ataraxis dependency is skewed         | Upgrade the offending package (see Step 6) |
| `as it has not been set` (directory)  | Working directory never configured       | Run `slsa configure directory`             |
| `as it has not been set` (templates)  | Templates directory never configured     | Run `slsa configure templates`             |
| `a file already exists at this path`  | `overwrite` was not passed               | Re-issue the write with `overwrite=True`   |
| `Unable to validate the payload as`   | Payload does not build the dataclass     | Fix it against `describe_*_schema_tool`    |
| `Unable to persist <Class> to <path>` | Destination suffix is not `.yaml`/`.yml` | Re-issue the write to a `.yaml` path       |
| `Unable to load <path> as <Class>`    | Invalid YAML from a previous edit        | Repair the file, then re-read it           |
| `Unable to reach the Unity Editor`    | McpBridge / Editor offline               | See `unity:unity-mcp-environment-setup`    |
| `did not answer within 30 seconds`    | Editor main thread busy                  | Wait for the Editor, then retry the call   |

The two path getters each raise `FileNotFoundError` in three conditions, not one: the cached path record does not exist
(`as it has not been set`), the record exists but is empty (`as the cached path record is empty`), and the record names
a directory that no longer exists on disk. `assets:working-directory` owns the full taxonomy and the repair procedure
for all three.

---

## Related skills

This skill is a prerequisite for **every** skill, in **any** plugin, that calls a `slsa mcp` tool. All of them depend on
the `sollertia-shared-assets` MCP server being reachable. That covers every other skill in the assets plugin, every
mesoscope plugin skill that reads or writes a shared descriptor, snapshot, or configuration, and every unity plugin
skill that consumes the Unity-relay tools. The cross-plugin map below covers only the relationships that the blanket
prerequisite does not already capture.

| Skill                                         | Relationship                                                                |
|-----------------------------------------------|-----------------------------------------------------------------------------|
| `assets:working-directory`                    | Owns the configured-path taxonomy behind every environment-status component |
| `assets:library-extension`                    | Owns the registry contracts whose import-time checks raise at server start  |
| `assets:datasets`                             | Consumer of the dataset tool family served by this same MCP server          |
| `mesoscope:mesoscope-vr`                      | Consumer that reads Mesoscope-VR descriptors and snapshots through this server |
| `unity:unity-mcp-environment-setup`           | Sibling that owns the McpBridge HTTP-relay diagnostic on the Unity side     |
| `experiment:experiment-mcp-environment-setup` | Peer carrying the equivalent diagnostic for the experiment plugin's MCP server |
| `forging:forging-mcp-environment-setup`       | Peer carrying the equivalent diagnostic for the forging plugin's MCP server |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and the `sollertia-shared-assets` MCP server is expected but unavailable
- Any slsa MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-shared-assets MCP server connectivity
- A Unity-relay tool fails and the slsa server itself is suspected, and otherwise hand off to
  `unity:unity-mcp-environment-setup`

---

## Verification checklist

```text
sollertia-shared-assets MCP environment setup:
- [ ] Checked sollertia-shared-assets MCP server connection status
- [ ] Read get_platform_environment_status_tool and routed any failing component to assets:working-directory
- [ ] Verified 'slsa' command is on PATH
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that the assistant must be restarted after environment changes
- [ ] Handed off to unity:unity-mcp-environment-setup if the issue is Unity-specific
```
