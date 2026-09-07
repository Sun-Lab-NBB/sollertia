---
name: forging-mcp-environment-setup
description: >-
  Diagnoses and resolves sollertia-forgery MCP server connectivity issues (environment, `slf` command availability,
  Python version, dependencies, the macOS OpenMP runtime). Owns the plugin-wide response envelope, staged-read contract,
  and paging keys. Use when the forging MCP tools are unavailable, when the server fails to start, when every job fails
  on a macOS host, or when a skill needs the response contract every forging tool returns.
user-invocable: false
---

# Sollertia forging MCP environment setup

Diagnoses and resolves sollertia-forgery MCP server connectivity and environment configuration issues. It also owns the
two plugin-wide contracts every other forging skill references, the response envelope the `slf mcp` tools return and the
macOS OpenMP runtime prerequisite every parallel pipeline enforces. This skill owns no MCP tool of its own. It calls
`read_resource_model_tool` as a reachability probe alone, and `/job-planning` owns that tool and the model it reports.

---

## Scope

**Covers:**
- Verifying the `sollertia-forgery` MCP server is reachable
- Diagnosing why the `slf` command and its `mcp` subcommand are unavailable
- Checking Python version compatibility, package integrity, and dependency bounds
- The macOS OpenMP runtime prerequisite every parallel pipeline verifies before it does any work
- The response envelope, the three widening read stages, the paging keys, and the breakdown axis cap

**Does not cover:**
- The rest of the `slf` command surface and its options. Owned by `/cli-reference`.
- Each tool's own parameters, filters, and error strings. Owned by the skill that owns the tool.
- The declared resource model and the job types it reports. Owned by `/job-planning`.
- The registry contracts whose import-time coverage check aborts the server. Owned by `/library-extension`.
- The Sollertia platform working directory and the data root. Owned by `assets:working-directory`.
- The `sollertia-shared-assets` `slsa mcp` server. Owned by `assets:assets-mcp-environment-setup`.
- The `sollertia-experiment` `sle mcp` server. Owned by `experiment:experiment-mcp-environment-setup`.

---

## Architecture

sollertia-forgery exposes a single MCP server, started through the `mcp` command of the `slf` Click entry point
declared in the `[project.scripts]` table of `pyproject.toml`:

```toml
[project.scripts]
slf = "sollertia_forgery.interfaces.entry_points:slf_cli"
```

| Server              | CLI command | Purpose                                                                                                        |
|---------------------|-------------|----------------------------------------------------------------------------------------------------------------|
| `sollertia-forgery` | `slf mcp`   | All 28 tools, spanning planning, processing, orchestration, forging, project management, and the remote server |

The forging plugin's `plugin.json` configures the Claude assistant to launch the server automatically:

```json
{
  "mcpServers": {
    "sollertia-forgery": { "command": "slf", "args": ["mcp"] }
  }
}
```

`slf` must be on PATH when the Claude assistant starts. A single `pip install sollertia-forgery` provides the `slf`
command and every one of its subcommands.

### Transport selection

`slf mcp` takes one option, `-t/--transport`, a case-insensitive `click.Choice` over `stdio`, `sse`, and
`streamable-http` that defaults to `stdio` (`interfaces/entry_points.py::run_mcp_server_command`). The `stdio` branch
calls `console.disable()`, because stdio carries the JSON-RPC stream on stdout and the pipelines these tools drive echo
status lines that would land inside a message and leave it unparsable. Every other transport keeps the console on and
emits `Starting the sollertia-forgery MCP server with the <transport> transport.` at `LogLevel.INFO`. `run_server`
passes `json_response=True` for `streamable-http` alone, which frames each response as one JSON body rather than an
event stream (`interfaces/mcp_server.py`). The `sse` and `streamable-http` transports serve the same tools over the
network, which is how an agent client reaches a server running on a separate processing host.

### Server identity and tool discovery

One shared `MCPServer(name="sollertia-forgery")` is constructed at import of `mcp_instance`, and every tool module
registers on that module-level `mcp` instance (`interfaces/mcp_instance.py`). `_register_tool_modules()` runs at import
of `mcp_server`, globs `*_tools.py` inside `interfaces/` in `sorted()` order, and imports each match. Tools register
purely as an import side effect of the `@mcp.tool()` decorators, so the `_tools.py` filename suffix is load-bearing and
a tool module sits directly in `interfaces/` rather than in a subdirectory.

The server registers 28 tools across six modules, `forging_tools.py`, `management_tools.py`, `planning_tools.py`,
`server_tools.py`, `processing_tools.py`, and `orchestration_tools.py`. `interfaces/remote_tools.py` declares no
`@mcp.tool()` decorator despite its name. Two of its entry points reach MCP as the `host="remote"` branch of
`get_processing_status_tool` and `cancel_processing_tool`, and the third backs `retire_remote_batches_tool`, which takes
no host. `interfaces/server.py` imports the status and retire entry points directly, which is how `slf server batches`
and `slf server retire-batch` answer exactly as their tools do. Registration is one import per module, so a failure
inside any one of them takes down all 28 tools rather than a subset. The corollary matters for triage: "some slf tools
are present and others are missing" is never an environment fault, so investigate the tool names and the plugin
registration instead. `run_mcp_server_command` defers `from .mcp_server import run_server` into the command body, so
`slf --help` never imports a tool module and never proves that the server starts.

### Dual-distribution model

| Component                                     | Distributed via           | What it provides                                          |
|-----------------------------------------------|---------------------------|-----------------------------------------------------------|
| Skills (`/batch-processing`, `/job-planning`) | sollertia forging plugin  | Skill files that guide agents through workflows           |
| MCP server registration                       | sollertia forging plugin  | The plugin entry that tells the assistant how to start it |
| MCP server code (`slf mcp`)                   | sollertia-forgery pip pkg | The actual CLI command and server implementation          |

Installing the plugin alone registers the MCP server and makes the skills available, and the server fails to start
unless `sollertia-forgery` is installed in the active Python environment.

### OpenMP runtime prerequisite

`sollertia_forgery/__init__.py` sets the Numba threading layer to `omp` on macOS and to `tbb` on every other platform.
The macOS omppool extension records its dependency as `@rpath/libomp.dylib` and carries no `LC_RPATH` entry of its own.
The loader therefore expands that name against the entries the running interpreter carries, and the only directory it
reaches is that interpreter's own library directory. Six of the seven pipelines call
`shared_assets/openmp.py::verify_openmp_runtime` before doing any work, and that call raises `RuntimeError` beginning
`Unable to locate the OpenMP runtime (libomp.dylib) that the Numba threading layer loads on macOS.` whenever the runtime
does not load. The `manifest` pipeline runs single-threaded and is the one pipeline that does not check.

A macOS host that has never linked the runtime therefore fails every parallel job while the MCP server reports perfectly
healthy, because the server process itself never opens a threading layer. `slf omp` is the only remedy, and there is no
MCP tool for OpenMP discovery or linking. Have the operator run it from the environment that runs the pipelines, so that
environment receives the link. A conda environment grants that write without `sudo`, while a system-wide interpreter
needs it.

```bash
slf omp
```

A bare run is a dry-run report, and it changes nothing. It always prints one summary line, it prints the `searched:`
line only when discovery examined paths, and it prints the `runtime:` and `link:` lines only when discovery resolved a
runtime. `resolve_openmp_runtime` returns one of the four `OpenMPStatus` members.

| Status       | What it means                                               | What to tell the operator                          |
|--------------|-------------------------------------------------------------|----------------------------------------------------|
| `available`  | The runtime already loads, so nothing was changed           | Nothing to do. `-f/--force` links a runtime anyway |
| `previewed`  | A runtime was found and the link was resolved as a dry run  | Re-run with `-y/--yes` from that same interpreter  |
| `unresolved` | No examined path holds a `libomp.dylib` file                | `brew install libomp`, then re-run `slf omp`       |
| `linked`     | The runtime was linked into the interpreter's lib directory | Read the summary line to confirm the runtime loads |

`slf omp` exits `1` on `unresolved`, which is the only deliberate non-zero exit in the package. Discovery examines the
two Homebrew keg directories and the MacPorts directory first, then `$CONDA_PREFIX/lib`, then the runtimes vendored
inside the installed Python distributions, and it links the first candidate that exists. That order is deliberate,
because a package manager runtime outlives the Python distributions installed beside it while a vendored one ties the
threading layer to the lifecycle of the distribution carrying it. `-s/--source` names a runtime explicitly and
`-t/--target` names the link destination. `resolve_openmp_runtime` raises on any platform other than macOS, so run the
command on macOS alone. Whenever a `linked` summary reports that the runtime still does not load, tell the operator to
set `DYLD_LIBRARY_PATH` to include the directory holding the runtime instead.

---

## Response contract

Every other forging skill references this plugin-wide contract by pointer and MUST NOT restate it.

### Response envelope

Every one of the 28 tools returns a plain `dict[str, Any]` built by one of the two constructors in
`interfaces/responses.py`, and no tool raises for an ordinary failure. `ok_response(**payload)` returns
`{"success": True, **payload}`, with `success` written first and the payload keys at the top level.
`error_response(message)` returns `{"success": False, "error": message}` and exactly those two keys. A failure carries
no error code, no error type, and no partial data, so branch on `success` before reading any other key and then match
the human-readable `error` string, which is stable per tool.

Three tools depart from the envelope. `execute_jobs_tool` appends `invalid_jobs` to its `No valid jobs to execute.`
error. `read_server_configuration_tool` and `write_server_configuration_tool` nest their payload under a `data` key,
which no other tool does. `read_server_configuration_tool` catches only `OSError` and `ValueError`, so a malformed YAML
document or a dataclass construction failure propagates out of that tool rather than returning an envelope.

### The three widening read stages

Every read tool answers at one of three widths, and a caller pays only for the width it requests.

1. A bare call reports the totals and a `breakdown` over the axes the tool accepts as filters, and lists no items.
2. Naming any filter, or passing `include_items=True`, adds a page of items alongside the totals and the breakdown.
3. Adding `detailed=True` widens each listed item with its provenance fields and drops the default page size to 50.

The stage-three fields differ per tool, the executor and the timestamps and the error text on the job readers, `notes`
on the manifest reader, and `job_id` and `memory_modeled` and `prerequisite_ids` on the plan reader. The scheduler
reader adds the requested figures against the occupied ones on its accounting view, and the allocation's shape and its
remaining walltime on its queue view. Four tools depart from the staged read. `discover_remote_project_tool` carries no
`detailed` parameter and stops at stage two. `list_project_datasets_tool`, `list_prepared_batches_tool`, and
`read_resource_model_tool` carry no `include_items` parameter, because their listing is always present, and
`read_resource_model_tool` carries no `detailed` parameter either.

### Paging keys and limits

Every listing tool merges the same four keys through `page_fields` at the top level of its response, never nested under
the items list.

| Key              | Value                                                                                   |
|------------------|-----------------------------------------------------------------------------------------|
| `rows`           | The items this response actually carries                                                |
| `matched_rows`   | The items matching the caller's filters, before the cap                                 |
| `start_row`      | The clamped offset at which this page begins                                            |
| `next_start_row` | The `start_row` that retrieves the next page, or `null` when this page ends the matches |

Walk a long result by following `next_start_row` until it is `null`, which the source calls a stronger signal than
comparing `rows` to `matched_rows`. `resolve_detail_limit` fills an omitted `limit` with `_DEFAULT_ITEM_LIMIT`, 200,
and with `_DEFAULT_DETAILED_LIMIT`, 50, when `detailed=True`. A caller-supplied `limit` is used verbatim, and a limit
at or below zero lifts the cap entirely and returns every remaining match in one response. `resolve_page` clamps a
negative `start_row` to `0` rather than counting back from the end, and a `start_row` at or past the total returns an
empty page whose `next_start_row` is `null`.

### Breakdown axes and sparse items

`count_values` renders one axis as `{str(value): count}` sorted by key, counting a `None` under the literal key `none`.
`bounded_counts` wraps it with the `_BREAKDOWN_AXIS_LIMIT` of 50. An axis holding more than 50 distinct values reports a
`distinct_values` count and an `elided` note in place of the counts. That note tells the caller to filter on the axis to
read the items carrying one of its values. `frame_breakdown` omits an axis the stored table does not carry rather than
reporting it empty.

Every listed item passes through `project_item` with `drop_empty=True`, which keeps only the fields the tool declares,
in the order it declares them, and drops a value that is `None`, an empty string, an empty list, or an empty dict. A
listed item is therefore sparse. `error_message` is absent rather than `null` for a job that recorded no failure, and
`prerequisite_ids` is absent for a job carrying no prerequisites. `0` and `False` survive, so a `complete` of `0` and a
`memory_modeled` of `false` are both reported. Read every field of a listed item through `.get()` semantics, and never
read an absent field as a recorded empty value.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the `sollertia-forgery` MCP server is
connected. Confirm reachability with `read_resource_model_tool()`, the cheapest probe the server carries. It touches no
disk, writes no tracker, opens no connection, and takes no `host` parameter, so a success envelope proves the server
answers without changing anything. A connected server settles the transport and the Python environment alone. On macOS
it does not clear the OpenMP prerequisite, so read the OpenMP runtime prerequisite section above whenever the tools
connect and every dispatched job fails.

### Step 2: Verify command availability

```bash
which slf
```

`slf` is the single parent binary that owns `mcp`, `omp`, and every planning, processing, and management subcommand. It
should resolve inside the active Python environment's `bin/` directory. If the binary is not found, proceed to step 3.
Server-side import health is settled in step 5 by hand-launching the server.

### Step 3: Identify the environment type and resolve

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep sollertia-forgery
```

The conda, venv, and uv decision tree is identical to the other sollertia MCP environment setup skills. The only
substitution is the package name:

```bash
pip install sollertia-forgery
# or
uv pip install sollertia-forgery
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating an environment
after the assistant has started leaves `slf` unavailable to the MCP server subprocess.

### Step 4: Verify Python version compatibility

```bash
python --version
```

sollertia-forgery requires Python `>=3.14,<3.15` (the `requires-python` field of `pyproject.toml`). If the version does
not match, instruct the user to create or activate an environment carrying a compatible one.

### Step 5: Verify package integrity

```bash
slf --help
slf mcp -t streamable-http
```

`slf --help` proves the CLI package imports, and it does not prove that the MCP server starts, because `slf mcp` defers
the `run_server` import into the command body and `--help` never reaches it. `slf mcp -t streamable-http` is the real
smoke test, and it falls outside the `--help` exemption, so the user runs it and pastes the output back. That branch
echoes its startup line and then blocks, so a startup line followed by a blocking process means the server is healthy
and the fault lies in the assistant's launch environment. A traceback in place of the startup line means a broken
dependency or an aborted registry check. Tell the user to interrupt it with Ctrl+C. Use `streamable-http` rather than
the `stdio` default, because a silent `stdio` server is indistinguishable from a hung one.

If either command fails with an import error, run:

```bash
pip check 2>&1 | head -20
```

The dependency bounds a skewed environment violates:

| Dependency                         | Required bound    | Triage note                                                                  |
|------------------------------------|-------------------|------------------------------------------------------------------------------|
| `mcp`                              | `>=2,<3`          | Check first, because `mcp_instance.py` imports `MCPServer` from `mcp.server` |
| `sollertia-shared-assets`          | `>=10,<11`        | 10.0.0 first exports the shared checksum exclusion set                       |
| `ataraxis-data-structures`         | `>=7.1,<8`        | 7.1 first exports the atomic and direct write helpers                        |
| `ataraxis-video-system`            | `>=5.1,<6`        | 5.1 first exports the layout against which the video pipeline resolves       |
| `ataraxis-communication-interface` | `>=7.1,<8`        | 7.1 first answers the archive estimator with a `JobSizing` record            |
| `cindra`                           | `>=2.0.0,<3`      | 2.0.0 first states the re-measured stage worker allocations                  |

`slf --help` never imports a tool module, so a failure inside one never reaches it. Reproduce that import directly:

```bash
python -c "import sollertia_forgery.interfaces.mcp_server"
```

That command also surfaces the second start-time failure, a `RuntimeError` whose message begins
`Unable to validate donor-registry coverage for`. `registries.py` runs `_assert_registry_coverage()` at import, and
every tool module reaches that module through `orchestration/dispatch.py`, so an incomplete acquisition-system extension
aborts the whole server. Hand that case to `/library-extension`, which owns the registry contracts and the extension
procedure.

### Step 6: Restart the MCP server

After resolving the environment issue, the user must restart the Claude assistant. The forging plugin reconnects the
server on the next session.

---

## Common issues and resolutions

| Symptom                                             | Cause                                                     | Resolution                                       |
|-----------------------------------------------------|-----------------------------------------------------------|--------------------------------------------------|
| `slf: command not found`                            | Environment not activated                                 | Activate conda or venv, restart the assistant    |
| `slf: command not found`                            | `sollertia-forgery` not installed                         | `pip install sollertia-forgery`                  |
| Import error on `slf mcp`                           | A pinned sibling library is skewed                        | Upgrade the offending package (see Step 5)       |
| `Unable to validate donor-registry coverage for`    | An incomplete acquisition-system extension                | See `/library-extension`                         |
| Python version mismatch                             | Wrong environment activated                               | Activate Python `>=3.14,<3.15`                   |
| Every job fails on macOS, the server reports ok     | The OpenMP runtime is not loadable                        | `slf omp`, then `slf omp -y` in that environment |
| `Unable to locate the OpenMP runtime`               | The same fault, raised by a pipeline preflight            | `slf omp`, then `slf omp -y` in that environment |
| `no OpenMP runtime to link`                         | No examined path holds a `libomp.dylib`                   | `brew install libomp`, re-run `slf omp`          |
| `Writing the link requires permission to modify`    | `slf omp -y` cannot write the interpreter's lib directory | Re-run under `sudo`, keeping that interpreter    |
| A tool errors naming the platform working directory | The working directory was never configured                | See `assets:working-directory`                   |
| `Unable to locate the 'server_configuration.yaml'`  | The server access record was never authored               | See `/server-configuration`                      |
| Some slf tools present and others missing           | Never an environment fault                                | Check tool names and plugin registration         |

Every runtime dependency other than the sibling libraries is bounded rather than pinned, and the sibling bounds carry
release floors that a lone upgrade breaks. Upgrade `sollertia-forgery` alone and let pip resolve the sibling versions
it requires, rather than naming a sibling beside it in one `--upgrade` invocation.

---

## Related skills

This skill is a prerequisite for every skill in the forging plugin that calls an `slf mcp` tool, and for every skill in
another plugin that reaches this server. The table below covers only the relationships the blanket prerequisite does
not already capture.

| Skill                                         | Relationship                                                                          |
|-----------------------------------------------|---------------------------------------------------------------------------------------|
| `/cli-reference`                              | Owns the rest of the `slf` command surface, including every option named nowhere here |
| `/job-planning`                               | Owns `read_resource_model_tool`, which this skill calls as a reachability probe alone |
| `/library-extension`                          | Owns the registry contracts whose import-time coverage check aborts the server        |
| `/batch-processing`                           | The heaviest consumer of the staged-read contract and the paging keys                 |
| `/server-configuration`                       | Owns the server access record every `host="remote"` call reads                        |
| `/pipeline`                                   | Routes a workflow across the plugin and depends on the server throughout              |
| `assets:working-directory`                    | Owns the platform working directory under which every batch record is written         |
| `assets:assets-mcp-environment-setup`         | Peer carrying the equivalent diagnostic for the `slsa mcp` server                     |
| `experiment:experiment-mcp-environment-setup` | Peer carrying the equivalent diagnostic for the `sle mcp` server                      |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:

- A session begins and the `sollertia-forgery` MCP tools are expected but unavailable
- Any `slf` MCP tool call fails with a connection or a server error
- The user mentions issues with sollertia-forgery MCP server connectivity
- Every dispatched job fails on a macOS host while the MCP server itself reports healthy
- Another skill needs the response envelope, the staged read widths, or the paging keys stated once

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

sollertia-forgery MCP environment setup:
- [ ] sollertia-forgery MCP server is connected
- [ ] Probed the server with read_resource_model_tool and read a success envelope back
- [ ] Verified 'slf' resolves on PATH inside the active Python environment
- [ ] Confirmed the interpreter version matches >=3.14,<3.15
- [ ] Identified the environment type (conda, venv, system)
- [ ] Asked the user to run 'slf mcp -t streamable-http' and read the startup line out of their pasted output
- [ ] Asked the user to run 'slf omp' on every macOS host and confirmed the reported status is available or linked
- [ ] Routed an 'Unable to validate donor-registry coverage' abort to library-extension
- [ ] Informed the user that the assistant must be restarted after environment changes
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
