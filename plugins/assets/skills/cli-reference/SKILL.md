---
name: cli-reference
description: >-
  Documents the human-facing slsa command-line interface of the sollertia-shared-assets library. Covers the root group,
  the get and configure subgroups, every leaf command and option with its short form, long form, type, default, and
  effect, the MCP tool each command maps to, and the tool families the CLI leaves entirely unexposed. Use when a user
  asks what an slsa command or option does, or when the MCP server is unavailable and the user must be told what to run
  by hand.
user-invocable: false
---

# CLI reference

> **The `slsa` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Print the command, ask the user to run it, and
> ask them to paste the output back.

**The one exemption** is `--help`. `slsa --help` and `slsa COMMAND --help` may be run, and no other `slsa` invocation
is exempt. `/assets-mcp-environment-setup` owns that exemption.

---

## Scope

**Covers:**
- The complete `slsa` command surface: every Click node, its purpose, and the MCP tool it maps to
- Every declared option: short form, long form, type, default, required, flag, or repeatable status, and effect
- Per-command failure modes, the exception each raises, and the exit code the user observes
- Which CLI commands have no MCP equivalent, which MCP tool families have no CLI equivalent, and where paired
  surfaces differ
- What to tell a user to run when the MCP server cannot be restored in this session

**Does not cover:**
- What the working directory, data root, credentials, and task-templates directory are, and the bootstrap order that
  sets them. Owned by `/working-directory`.
- The project hierarchy, the discovery strategies, and project creation semantics. Owned by `/project-hierarchy`.
- Session discovery and filtering. Owned by `/session-discovery`. Per-session inspection. Owned by `/session-data`.
- Every record schema and its read, write, and describe tools. Owned by `/session-data`, `/session-descriptors`,
  `/session-hardware-state`, `/data-assets`, `/datasets`, `/task-templates`, and `/experiment-configuration`.
- The registry-backed extension path behind the `list_supported_*` tools. Owned by `/library-extension`.
- The Unity Editor relay tools. Owned by the `unity` plugin.
- MCP connectivity, the response envelope, and diagnosing why the server is down. Owned by
  `/assets-mcp-environment-setup`.

**Handoff rules:** If the user wants an operation performed rather than explained, use the MCP tools and invoke the
owning skill. If the MCP tools are unavailable, invoke `/assets-mcp-environment-setup` first, and fall back to the
handoff table below only after the server cannot be restored.

---

## Agent requirements

You MUST answer CLI questions from this skill or from `slsa COMMAND --help`, never from memory. When a user's report
disagrees with this reference, ask them to run `slsa COMMAND --help` and read the installed build's answer.

The `slsa` CLI is a bootstrap and inspection surface alone. It sets the four persisted host records, creates a project
directory, reports what is configured, and starts the MCP server. Every authoring, validation, schema, and Unity
operation the plugin performs lives on the MCP server, so a question about reading or writing any record is answered by
the owning skill rather than by this one.

---

## Command surface

The CLI declares fifteen Click nodes: the root group, two subgroups, and twelve leaf commands. Every node lives in
`interfaces/cli.py`, and the entry point is `slsa = "sollertia_shared_assets.interfaces.cli:slsa_cli"`. The `get`
group reports what is configured, and the `configure` group writes it.

| Click node                   | Kind    | Purpose                                                                  | MCP equivalent                              |
|------------------------------|---------|--------------------------------------------------------------------------|---------------------------------------------|
| `slsa`                       | group   | Entry point. Dispatches to the subcommands and prints the listing        | None, dispatch only                         |
| `slsa mcp`                   | command | Starts the MCP server over the named transport                           | None, this command IS the server            |
| `slsa get`                   | group   | Dispatches the reporting subcommands                                     | None, dispatch only                         |
| `slsa get directory`         | command | Reports the configured local platform working directory                  | `read_working_directory_tool`               |
| `slsa get data-root`         | command | Reports the configured local platform data root                          | `read_data_root_tool`                       |
| `slsa get credentials`       | command | Reports the path to one category's stored credentials file               | `read_credentials_tool`                     |
| `slsa get templates`         | command | Reports the configured task templates directory                          | `read_task_templates_directory_tool`        |
| `slsa get projects`          | command | Lists the project directories under the data root                        | `get_data_root_overview_tool`, nearest only |
| `slsa get experiments`       | command | Lists one project's experiment configuration filename stems              | `discover_experiments_tool`                 |
| `slsa configure`             | group   | Dispatches the configuration subcommands                                 | None, dispatch only                         |
| `slsa configure directory`   | command | Sets the local platform working directory and creates its subdirectories | `set_working_directory_tool`                |
| `slsa configure data-root`   | command | Sets the local platform data root                                        | `set_data_root_tool`                        |
| `slsa configure credentials` | command | Copies a credentials file into the platform credentials directory        | `set_credentials_tool`                      |
| `slsa configure templates`   | command | Sets the sollertia-virtual-reality task templates directory              | `set_task_templates_directory_tool`         |
| `slsa configure project`     | command | Creates a new project's directory structure under the data root          | `create_project_tool`                       |

**Note on `-h`:** the CLI leaves Click's `help_option_names` at its `["--help"]` default, and no command binds `-h` to
an option, so `slsa get -h` fails with `Error: No such option '-h'.` at exit code 2.

---

## Option reference

The surface declares nine options across eight leaf commands. The other four take no arguments at all, those being
`slsa get directory`, `slsa get data-root`, `slsa get templates`, and `slsa get projects`. Neither group declares an
option of its own, so no option is ever parsed on a parent and no option order rule applies. No option on this surface
is a flag, none is repeatable, and none prompts. "Required" means Click rejects the invocation without it at exit
code 2.

Every node passes `context_settings={"max_content_width": 120}`, so every `--help` rendering wraps at that width or at
the terminal width, whichever is narrower.

### `slsa mcp`

| Short | Long          | Type     | Default   | Form     | Effect                                             |
|-------|---------------|----------|-----------|----------|----------------------------------------------------|
| `-t`  | `--transport` | `Choice` | `"stdio"` | optional | Transport. Choice of `stdio` and `streamable-http` |

The choice is case-insensitive and declares `show_default=True`. `stdio` calls `console.disable()`, because one echoed
status line inside the JSON-RPC stream the two share makes a message unparsable. `/assets-mcp-environment-setup` owns
the transport behavior and the smoke-test procedure that follows from it.

### `slsa get`

| Command       | Short | Long         | Type     | Default    | Form     | Effect                                                            |
|---------------|-------|--------------|----------|------------|----------|-------------------------------------------------------------------|
| `credentials` | `-c`  | `--category` | `Choice` | (required) | required | The credentials category to report. Members of `CredentialsTypes` |
| `experiments` | `-p`  | `--project`  | `str`    | (required) | required | The project whose experiment configurations are listed            |

The `--category` choice is built at import from `[member.value for member in CredentialsTypes]` and is
case-insensitive. That enumeration currently holds the single member `google`, so Click rejects every other value with
`'<value>' is not 'google'`. `/library-extension` owns the path that adds a member, and no CLI edit is needed when one
lands.

### `slsa configure`

| Command       | Short | Long          | Type     | Default    | Form     | Effect                                                            |
|---------------|-------|---------------|----------|------------|----------|-------------------------------------------------------------------|
| `directory`   | `-d`  | `--directory` | `Path`   | (required) | required | The directory that caches platform configuration and runtime data |
| `data-root`   | `-d`  | `--directory` | `Path`   | (required) | required | The directory under which all project directories are stored      |
| `credentials` | `-c`  | `--category`  | `Choice` | (required) | required | The credentials category to configure                             |
| `credentials` | `-f`  | `--file`      | `Path`   | (required) | required | The credentials file to copy into the credentials directory       |
| `templates`   | `-d`  | `--directory` | `Path`   | (required) | required | The sollertia-virtual-reality Configurations (Template) directory |
| `project`     | `-p`  | `--project`   | `str`    | (required) | required | The name of the project to create                                 |

The three `--directory` options carry different `click.Path` constraints, and the difference decides whether Click or
the library reports a missing target.

| Command                      | Option        | `exists` | `file_okay` | `dir_okay` | Consequence                                      |
|------------------------------|---------------|----------|-------------|------------|--------------------------------------------------|
| `slsa configure directory`   | `--directory` | `False`  | `False`     | `True`     | A missing directory is created by the command    |
| `slsa configure data-root`   | `--directory` | `False`  | `False`     | `True`     | A missing directory is created by the command    |
| `slsa configure templates`   | `--directory` | `True`   | `False`     | `True`     | Click rejects a missing directory at exit code 2 |
| `slsa configure credentials` | `--file`      | `True`   | `True`      | `False`    | Click rejects a missing file at exit code 2      |

Every one of them declares `path_type=Path`, so the command body receives a `pathlib.Path` rather than a string.

### Short forms

No short form on this surface means two different things. `-t` is `--transport`, `-c` is `--category`, `-p` is
`--project`, `-d` is `--directory`, and `-f` is `--file`, in every command that declares them. Quote the long form
anyway when handing a command to a user, because the same short form spans two groups.

---

## Command behavior and failure modes

### How a failure reaches the user

No command body is wrapped in a catching decorator, and every library guard raises through
`ataraxis_base_utilities.console.error`, which raises whether or not the console is enabled. A guard failure therefore
leaves a Python traceback and exit code 1, so ask the user to paste the traceback rather than the status.

| Path                                                                      | Mechanism                        | Observable outcome    |
|---------------------------------------------------------------------------|----------------------------------|-----------------------|
| A missing required option, a rejected choice, or a path failing its check | Click's own parameter validation | Usage message, exit 2 |
| `-h`, or any other unrecognized token                                     | Click's own parser               | Usage message, exit 2 |
| A configured record that is unset, empty, or points at a missing path     | `console.error`, uncaught        | Traceback, exit 1     |
| A credentials file that is missing or carries the wrong extension         | `console.error`, uncaught        | Traceback, exit 1     |

There is no version option, no verbosity option, no dry-run option, and no host option anywhere on the surface. `slsa`
addresses this machine only.

### What every command prints

`sollertia_shared_assets/__init__.py` enables the shared console at import, so every `console.echo` in the CLI reaches
the terminal. Each command prints one human-readable sentence that ends in a full stop, and no command emits JSON or
any other structured form. `slsa get directory` prints `Working directory: /path/to/directory.`, so a caller parsing
that output strips the trailing full stop.

### The four `get` readers

`slsa get directory`, `slsa get data-root`, and `slsa get templates` each read one `platformdirs`-managed record and
raise `FileNotFoundError` in three cases: the record was never written, the cached path record is empty, and the
configured directory no longer exists. Each message names the `slsa configure` command that fixes it, and
`/working-directory` owns what those records mean.

`slsa get credentials` resolves the category to a canonical filename, then joins it under the working directory's
`credentials` subdirectory. It raises `FileNotFoundError` when the working directory is unset or when that category's
file was never copied in, and the message names `slsa configure credentials`. Its `ValueError` branch for an unknown
category is unreachable through the CLI, because Click's `Choice` rejects the value first.

### `slsa get projects` and `slsa get experiments`

`slsa get projects` calls `discover_projects` with `strategy="directories"`, which lists the non-hidden directories
directly under the data root in natural sort order. `/project-hierarchy` owns the two strategies and the
false-positive cost of the directory one. Finding nothing is not an error, and the command prints a line naming
`slsa configure project` as the remedy.

`slsa get experiments` builds a `ProjectData` view and globs `configuration/*.yaml` under it. The view is a path
grammar that does not require the project to exist, and `experiment_configs()` returns an empty tuple when the
configuration directory is absent, so a misspelled project name reports the empty-result message rather than failing.
That message names the acquisition system's own CLI as the way to create a configuration, because this library
authors none.

Both commands resolve the data root through `get_data_root()`, so both raise `FileNotFoundError` on a host that never
ran `slsa configure data-root`.

### The five `configure` writers

| Command                      | What it writes                                                                                        |
|------------------------------|-------------------------------------------------------------------------------------------------------|
| `slsa configure directory`   | Creates the directory plus its `configuration` and `credentials` subdirectories, then caches the path |
| `slsa configure data-root`   | Creates the directory, then caches the path. No subdirectories are created                            |
| `slsa configure credentials` | Copies the file under its canonical registered name into the working directory's `credentials`        |
| `slsa configure templates`   | Caches the resolved path after re-checking that it exists and is a directory                          |
| `slsa configure project`     | Creates `<data root>/<project>/configuration`, which creates the project directory as its parent      |

Each cached path is published through `atomic_write`, so a process killed mid-write leaves the previously configured
record readable. `slsa configure credentials` copies the source bytes atomically for the same reason, and it replaces
any previously configured file for that category outright with no confirmation. It also raises `ValueError` when the
source file's extension does not match the canonical filename's extension, which is `.json` for the `google` category.

`slsa configure project` is idempotent, because `ProjectData.create` leaves existing directories untouched. It
resolves the data root from `get_data_root()` and accepts no root override, so it fails on a host that skipped
`slsa configure data-root`.

---

## How the CLI diverges from the MCP path

The response envelope every tool on this server returns, and the write-validation contract its write tools follow, are
documented in the `## Response contract` section of `/assets-mcp-environment-setup`.

The `slsa mcp` server registers 66 tools across four modules, `configuration_tools.py`, `data_tools.py`,
`dataset_tools.py`, and `unity_tools.py`. Ten of them have a CLI counterpart. The other 56 have no CLI surface at all,
so this section carries more weight here than in the sibling CLI references.

### CLI commands with no exact MCP equivalent

- `slsa mcp` starts the server, so it is CLI-only by definition.
- `slsa get projects` has no tool that answers the project-name listing alone. `get_data_root_overview_tool` with
  `strategy="directories"` is the nearest neighbour, and it returns the whole project, animal, and session tree.

### MCP tool families with no CLI surface

Route each family to its owning skill rather than improvising a CLI substitute, because none exists.

| Tool family                            | Tools | Owning skill                |
|----------------------------------------|-------|-----------------------------|
| Platform environment status            | 1     | `/working-directory`        |
| Task template authoring and validation | 5     | `/task-templates`           |
| Experiment configuration authoring     | 5     | `/experiment-configuration` |
| Registry listings                      | 7     | `/library-extension`        |
| Session discovery and filtering        | 2     | `/session-discovery`        |
| Session data, trackers, and inspection | 5     | `/session-data`             |
| Session descriptors                    | 3     | `/session-descriptors`      |
| Session hardware state                 | 3     | `/session-hardware-state`   |
| Read assets                            | 3     | `/data-assets`              |
| Datasets                               | 7     | `/datasets`                 |
| Unity Editor relay                     | 18    | the `unity` plugin          |

The session discovery row counts `get_data_root_overview_tool`, whose project-name slice `slsa get projects` does
reach. Every other tool in these eleven families is unreachable from the CLI.

The Unity family spans task creation and deletion, prefab inspection and zone cloning, asset listing and deletion,
scene listing, opening, and inspection, play mode, task parameters, and monitor refresh. `unity:task-generator`,
`unity:task-prefabs`, `unity:zone-prefabs`, `unity:task-scenes`, `unity:play-mode`, and `unity:task-parameters` own
those operations, and `unity:unity-mcp-environment-setup` owns the relay diagnostic.

### Where a paired command and tool differ

| CLI command                  | Nearest MCP tool                     | Divergence                                                                                                                                                                                                               |
|------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `slsa get directory`         | `read_working_directory_tool`        | Same read. The CLI raises a traceback where the tool returns `{"success": false, "error": ...}`                                                                                                                          |
| `slsa get data-root`         | `read_data_root_tool`                | As above                                                                                                                                                                                                                 |
| `slsa get templates`         | `read_task_templates_directory_tool` | As above                                                                                                                                                                                                                 |
| `slsa get credentials`       | `read_credentials_tool`              | The CLI validates the category through Click's `Choice`, and the tool accepts any string and reports the `ValueError` as a failure                                                                                       |
| `slsa get projects`          | `get_data_root_overview_tool`        | The CLI prints project names only. The tool returns the project, animal, and session tree with per-status counts                                                                                                         |
| `slsa get experiments`       | `discover_experiments_tool`          | The CLI prints natural-sorted stems for one project and the tool returns lexicographic project, stem, and path triples for any number. A project directory that does not exist is an empty CLI result and a tool failure |
| `slsa configure directory`   | `set_working_directory_tool`         | Same operation. The CLI receives a `Path` validated by Click, and the tool receives a string it wraps                                                                                                                    |
| `slsa configure data-root`   | `set_data_root_tool`                 | As above                                                                                                                                                                                                                 |
| `slsa configure credentials` | `set_credentials_tool`               | Same copy. The tool re-reads the destination and echoes the resolved path in its payload                                                                                                                                 |
| `slsa configure templates`   | `set_task_templates_directory_tool`  | Click rejects a missing directory before the library runs, and the tool reports the library's own error instead                                                                                                          |
| `slsa configure project`     | `create_project_tool`                | The CLI always resolves the data root from the persisted record. The tool takes an optional `root_directory` override                                                                                                    |

### The three rules behind the table

1. **Error surface.** Every CLI failure is a traceback at exit code 1 or a Click usage message at exit code 2, while
   every tool returns a `success` envelope instead of raising.
2. **Payload richness.** A CLI command prints one sentence, and its paired tool returns the resolved paths and counts
   the caller would otherwise re-derive.
3. **Reach.** No `slsa` command takes a host, so the CLI addresses this machine alone.

---

## Fallback: what to tell a user when MCP is unavailable

Confirm the server is genuinely unrecoverable through `/assets-mcp-environment-setup` before handing one of these over.

| Blocked MCP tool                     | Tell the user to run                                 |
|--------------------------------------|------------------------------------------------------|
| `read_working_directory_tool`        | `slsa get directory`                                 |
| `read_data_root_tool`                | `slsa get data-root`                                 |
| `read_credentials_tool`              | `slsa get credentials -c <category>`                 |
| `read_task_templates_directory_tool` | `slsa get templates`                                 |
| `get_data_root_overview_tool`        | `slsa get projects`, project names only              |
| `discover_experiments_tool`          | `slsa get experiments -p <project>`                  |
| `set_working_directory_tool`         | `slsa configure directory -d <path>`                 |
| `set_data_root_tool`                 | `slsa configure data-root -d <path>`                 |
| `set_credentials_tool`               | `slsa configure credentials -c <category> -f <file>` |
| `set_task_templates_directory_tool`  | `slsa configure templates -d <path>`                 |
| `create_project_tool`                | `slsa configure project -p <name>`                   |

Two caveats. `slsa get projects` answers the project listing alone, so an animal or session question has no CLI
substitute. `slsa configure project` takes no root override, so a host whose data root is unset must run
`slsa configure data-root` first.

Everything else genuinely blocks until the server is back. That covers the platform environment status report, every
task template and experiment configuration operation, every registry listing, session discovery, inspection, and
filtering, every session data, descriptor, hardware state, read asset, and dataset operation, and every Unity Editor
relay operation. Say so plainly rather than improvising a substitute.

---

## Related skills

| Skill                               | Relationship                                                                                    |
|-------------------------------------|-------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`     | Owns the `--help` exemption, the `slsa mcp` transports, the response contract, and MCP recovery |
| `/working-directory`                | Owns the four configured records the `get` and `configure` commands read and write              |
| `/project-hierarchy`                | Owns the discovery strategies behind `slsa get projects` and project creation                   |
| `/session-discovery`                | Owns the session discovery and filtering tools with no CLI surface                              |
| `/task-templates`                   | Owns the template tools that sit behind the configured templates directory                      |
| `/experiment-configuration`         | Owns the configuration tools behind `slsa get experiments`                                      |
| `/library-extension`                | Owns the registries behind the `--category` choice and the `list_supported_*` tools             |
| `/datasets`                         | Owns the dataset tool family with no CLI surface                                                |
| `unity:unity-mcp-environment-setup` | Owns the relay diagnostic for the eighteen Unity tools with no CLI surface                      |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier

Answering a CLI question, reader-judged:
- [ ] Answered from this skill or from slsa COMMAND --help, never from memory
- [ ] Quoted the long option form, and never presented -h as a help alias
- [ ] Named the MCP tool the command maps to, or said plainly that none exists
- [ ] Invoked no slsa command other than --help

Handing a user a CLI command, reader-judged:
- [ ] Confirmed the MCP server is genuinely unrecoverable through assets-mcp-environment-setup first
- [ ] Printed the command for the user instead of running it
- [ ] Named the required option on every command that declares one, since none carries a default
- [ ] Routed a record read, write, validate, or describe request to its owning skill rather than to a CLI command
- [ ] Confirmed the data root is configured before recommending slsa configure project or slsa get experiments
```
