---
name: forging-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-forgery MCP server connectivity issues (environment,
  command availability, Python version, dependencies). Use when the MCP tools are
  unavailable, the server fails to start, or a new session needs sollertia-forgery tools.
user-invocable: false
---

# MCP environment setup

Diagnoses and resolves sollertia-forgery MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the sollertia-forgery MCP server is reachable and functional
- Diagnosing why the `slf` command is unavailable
- Checking Python version compatibility (`>=3.14,<3.15`)
- Validating sollertia-forgery package installation and core dependencies
- Environment-specific guidance for conda, pip, and uv workflows

**Does not cover:**
- MCP tool usage for session discovery (see `assets:session-discovery`)
- MCP tool usage for manifest reading and generation (see `/project-manifest`)
- MCP tool usage for checksum verification (see `/checksum-verification`)
- MCP tool usage for batch processing (see `/behavior-processing`)
- MCP tool usage for output verification (see `/behavior-results`)
- Input data preparation from upstream libraries (see `/behavior-input-format`)
- sollertia-forgery package development or contribution workflows

---

## Architecture

sollertia-forgery provides a single MCP server, started by the `mcp` subcommand of the `slf` console script defined in
`pyproject.toml`:

```toml
[project.scripts]
slf = "sollertia_forgery.interfaces.entry_points:slf_cli"
```

| Server              | CLI command | Purpose                                                       |
|---------------------|-------------|---------------------------------------------------------------|
| `sollertia-forgery` | `slf mcp`   | Session discovery, batch behavior processing, output querying |

The `mcp` subcommand takes one option, `-t/--transport`, which defaults to `stdio`. The sollertia forging plugin's
`plugin.json` configures the Claude assistant to launch the server automatically:

```json
{
  "mcpServers": {
    "sollertia-forgery": {
      "command": "slf",
      "args": ["mcp"]
    }
  }
}
```

The `slf` command must be on PATH when the Claude assistant starts. This means the Python environment
where sollertia-forgery is installed must be active before launching the assistant.

### Dual-distribution model

The sollertia forging plugin's Claude integration is split across two distribution channels:

| Component                                                         | Distributed via               | What it provides                                                     |
|-------------------------------------------------------------------|-------------------------------|----------------------------------------------------------------------|
| Skills (`assets:session-discovery`, `/behavior-processing`, etc.) | sollertia forging plugin      | Skill files that guide agents through workflows                      |
| MCP server registration                                           | sollertia forging plugin      | Plugin entry that tells the Claude assistant how to start the server |
| MCP server code (`slf mcp`)                                       | sollertia-forgery pip package | The actual CLI command and server implementation                     |

Installing the plugin alone registers the MCP server and makes skills available, but the server will fail
to start because the `slf` CLI command is not present. The pip package must also be installed in the
active Python environment for the MCP server to function.

This is the most common cause of MCP failures after initial setup: the plugin is installed but the pip
package is not, or the pip package is installed in a different Python environment than the one active when
the Claude assistant launches.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable or the server fails to start.

### Step 1: Test MCP tool availability

Attempt to call any sollertia-forgery MCP tool (for example `get_checksum_status_tool`). If the
call succeeds, the environment is healthy, and you can invoke the target skill. If the call returns a
connection error, continue to step 2.

### Step 2: Verify CLI command availability

```bash
which slf
```

Expected outcome: `which slf` prints a path inside the active environment (conda env, venv, or uv tool dir). If `which`
returns nothing, the package is not installed in the active environment, see "Installation workflows" below.
`slf --help` and `slf mcp --help` print usage without starting anything, but you MUST NOT run a bare `slf mcp` from a
shell: it launches the stdio MCP server and blocks until the client or the user terminates it.

### Step 3: Verify Python version

```bash
python --version
```

Expected: Python 3.14.x. sollertia-forgery requires `>=3.14,<3.15`. If the active environment is on a
different version, create a new environment at the correct version and reinstall.

### Step 4: Verify package installation

```bash
python -c "import sollertia_forgery; print(sollertia_forgery.__file__)"
```

Expected: a path inside the active environment's `site-packages`. If the import fails with
`ModuleNotFoundError`, the package is not installed.

### Step 5: Verify core dependencies

```bash
python -c "import mcp, polars, numpy, numba, scipy, ataraxis_time, ataraxis_base_utilities, \
ataraxis_data_structures, sollertia_shared_assets; print('ok')"
```

Expected: prints `ok`. Any `ImportError` indicates a missing or version-incompatible dependency — see
`pyproject.toml` for the authoritative version constraints.

---

## Installation workflows

### Conda / mamba

```bash
mamba create -n slf_dev python=3.14
mamba activate slf_dev
pip install -e .
```

Run inside the cloned `sollertia-forgery` repository directory. The editable installation wires the
`slf` entry point into the environment's `bin/` so Claude Code can launch it by name.

### uv

```bash
uv venv --python 3.14
source .venv/bin/activate
uv pip install -e .
```

### pip (system Python)

```bash
python3.14 -m venv .venv
source .venv/bin/activate
pip install -e .
```

On Windows, replace `source .venv/bin/activate` with `.venv\Scripts\activate` in the uv and pip
workflows above (the conda / mamba workflow is identical on all platforms).

---

## Common failure modes

| Symptom                                        | Diagnosis                                                   | Resolution                                                                                      |
|------------------------------------------------|-------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| `slf: command not found`                       | Package not installed, or wrong environment active          | Activate the correct environment and `pip install -e .`                                         |
| Skills available but MCP tools missing         | Plugin installed without pip package                        | Install `sollertia-forgery` into the active Python environment and restart the Claude assistant |
| `ModuleNotFoundError: sollertia_shared_assets` | Shared-assets dependency missing                            | `pip install "sollertia-shared-assets>=8.0.0rc1,<9"`                                            |
| `ModuleNotFoundError: mcp`                     | FastMCP dependency missing                                  | `pip install "mcp[cli]>=1,<2"`                                                                  |
| Python version mismatch                        | Active environment does not meet `>=3.14,<3.15` requirement | Recreate environment with Python 3.14                                                           |
| `slf mcp` starts but returns no tools          | Claude assistant connected to a stale process               | Restart the Claude assistant so the MCP client respawns the server                              |

---

## Related skills

| Skill                                | Relationship                                                       |
|--------------------------------------|--------------------------------------------------------------------|
| `assets:session-discovery`   | Downstream: session discovery once MCP is verified                 |
| `/project-manifest`                  | Downstream: manifest reading and generation require MCP tools      |
| `/checksum-verification`             | Downstream: checksum batch pipeline requires MCP tools             |
| `/behavior-input-format`             | Reference: input formats consumed by MCP-driven workflows          |
| `/behavior-processing`               | Downstream: batch processing operations require MCP tools          |
| `/behavior-results`                  | Downstream: output verification and querying require MCP tools     |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:
- A session begins and MCP tools from the sollertia-forgery server are expected but unavailable
- Any sollertia-forgery MCP tool call fails with a connection or server error
- The user mentions issues with sollertia-forgery MCP server connectivity or environment setup

---

## Verification checklist

```text
MCP Environment Setup:
- [ ] Checked MCP server connection status (sollertia-forgery)
- [ ] Verified `slf` command is on PATH (`which slf`)
- [ ] Confirmed Python version matches >=3.14,<3.15
- [ ] Confirmed sollertia-forgery package and core dependencies import successfully
- [ ] Confirmed sollertia forging plugin installed and registers the sollertia-forgery MCP server
- [ ] Informed user that the Claude assistant must be restarted after environment or plugin changes
- [ ] Confirmed at least one sollertia-forgery MCP tool call returns without connection errors
```
