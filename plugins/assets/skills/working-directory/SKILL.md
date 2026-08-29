---
name: working-directory
description: >-
  Initializes the local Sollertia working directory, data root, platform credentials, and task templates directory via
  the sollertia-shared-assets MCP server. Prerequisite for every other assets-plugin skill. Use when setting up
  Sollertia on a new host, relocating the data root, or when configuration tools fail because the working directory is
  not set.
user-invocable: false
---

# Sollertia working directory setup

Initializes the local Sollertia working directory, the data root, and the supporting credentials files and task
templates directory used by `sollertia-shared-assets`. This is the first skill to invoke on any host that will run
Sollertia configuration or runtime tooling.

---

## Scope

**Covers:**
- Setting and reading the local working directory
- Setting and reading the Sollertia data root
- Setting and reading platform credentials files by category (currently the `google` category)
- Setting and reading the task templates directory
- The bootstrap order required by other configuration MCP tools
- Explaining what each configurable asset is, why it exists, and when it is needed

**Does not cover:**
- Authoring system configuration (see `experiment:acquisition-system-design`)
- Authoring server configuration (see `forging:server-configuration`)
- Authoring task templates (see `/task-templates`)
- Authoring experiment configurations (see `/experiment-configuration`)
- Creating projects (see `/project-hierarchy`)
- Reading session-level data (see `/session-data`, `/session-descriptors`, `/session-hardware-state`, `/data-assets`,
  `/session-discovery`, and `mesoscope:mesoscope-vr-snapshots`)
- Reading datasets (see `/datasets`)
- Diagnosing MCP server connectivity (see `/assets-mcp-environment-setup`)

---

## What lives in the working directory

The Sollertia working directory is the cache root for **host-machine-local** configuration state. It is distinct from
the long-term storage tier where session data lives. Setting the working directory creates the `configuration/` and
`credentials/` subdirectories:

```text
<working-directory>/
├── configuration/
│   ├── <system>_system_configuration.yaml   # Owned by sollertia-experiment (e.g. mesoscope_system_configuration.yaml)
│   └── server_configuration.yaml            # Owned by sollertia-forgery
└── credentials/
    └── google_credentials.json              # Configured via set_credentials_tool
```

The working directory path is persisted under `platformdirs.user_data_dir(appname="sollertia_data",
appauthor="sollertia")`, so it survives across CLI invocations and MCP sessions on the same host, and other MCP tools
resolve their default paths against this directory. That application directory holds three plain-text records, each
containing exactly one path string: `working_directory_path.txt`, `data_root_path.txt`, and
`task_templates_directory_path.txt`. Every getter reads its record back with `rstrip("\r\n")`, so only line terminators
are stripped. A directory name ending in a space survives intact, and a record holding nothing but a newline is rejected
as empty. The three settings are fully independent, with no precedence, no fallback, and no inheritance between them.
The data root is not derived from the working directory or the reverse. No environment variable and no CLI argument
overrides any of the three records.

### configuration/

This skill initializes the `configuration/` subdirectory but never reads or writes its files. The YAMLs inside are owned
by downstream plugins:

- `<system>_system_configuration.yaml`, one per acquisition system, backed by a per-system configuration class in
  `sollertia-experiment`. For the current Mesoscope-VR reference system that is `mesoscope_system_configuration.yaml`
  backed by `MesoscopeSystemConfiguration`. Its authoring follows the `experiment:acquisition-system-design` pattern.
- `server_configuration.yaml`, backed by `ServerConfiguration` in `sollertia-forgery` and authored by the
  `forging:server-configuration` skill.

Defer to each owning skill for the schema, the authoring workflow, and the rotation cadence.

### credentials/

The `credentials/` subdirectory stores the credentials files the platform uses to interact with external services. Each
supported credentials category maps to a canonical filename. The `google` category, holding the Google Sheets
service-account credentials JSON, is stored as `google_credentials.json`. `set_credentials_tool` validates the source
file and **copies** it into this subdirectory under the canonical name, replacing any previously configured file for the
same category. Because the platform reads the copy, later edits to the original file do not propagate until the
credentials are re-set.

The published copy is an atomic binary write, so a rotation killed partway leaves the previous key intact rather than
truncated. That copy carries umask-derived permissions rather than the source file's mode, so a key that was `0600` at
its origin can land world-readable. You MUST verify or tighten the mode on
`<working-directory>/credentials/google_credentials.json` after Step 4.

Google credentials are required whenever a project **reads** subject metadata **from** Google Sheets or **writes**
water-restriction logs **to** them. The integration is bidirectional, so a read-only service account fails on the write
path even though it authenticates. If no project on the host uses Google Sheets, the credentials can be left unset, but
downstream tools that reach the sheets will fail until they are set.

---

## The data root

The Sollertia **data root** is the directory under which every project directory, and therefore every animal and session
directory, is stored on this host (`<data-root>/<project>/<animal>/<session>/`). It is distinct from the working
directory: the working directory caches host-local configuration, while the data root anchors the acquired-data
hierarchy. The two are commonly separate volumes, a fast local disk for configuration and a large storage tier for
sessions.

The data root path is persisted via `platformdirs`, independently of the working directory. Persisting it lets the
project-hierarchy and session-discovery workflows resolve the project tree without the caller re-supplying the root each
time. Its consumers are the `slsa get` CLI commands that resolve against it (`slsa get projects`,
`slsa get experiments`, `slsa get data-root`). `create_project_tool` also consumes it when its optional `root_directory`
argument is omitted. `slsa configure project` consumes it as well, resolving the root from `get_data_root()` and
accepting no override at all.

The MCP discovery and inventory tools do **not** default to the persisted root. `get_data_root_overview_tool`,
`discover_experiments_tool`, and `discover_datasets_tool` all take `root_directory` as a required argument. Reading the
persisted data root with `read_data_root_tool` is how the agent recalls "where projects live on this host" before
passing that path to those tools.

The data root is **optional** at the platform-status level, carrying `required=False`, so leaving it unset does not
break `overall_ok`. A host that always supplies an explicit root to the discovery and inventory tools can skip it. A
host that mints projects cannot, because `slsa configure project` has no root override and `create_project_tool` falls
back to the persisted record whenever `root_directory` is omitted.

---

## Task templates directory

The task templates directory is a standalone directory (separate from the working directory) that holds reusable
`TaskTemplate` YAML files. This is typically the path to the local sollertia-virtual-reality repository's template
directory: `<local-repo>/Assets/InfiniteCorridorTask/Configurations/`. A template outside that directory is invisible to
the MCP `create_task_tool` (see `unity_tools.py`). The Unity-side `CreateFromTemplate` generator scans the same
directory (see `unity:task-prefabs` for `create_task_tool` and `unity:task-generator` for the generator's directory
scan), so this directory is the single source of truth. Keep it pointed at the canonical repo path.

Templates are **project-agnostic**, so the same template can back many per-project experiment configuration instances
across different projects and even across different hosts. Decoupling the templates directory from the working directory
lets multiple hosts share the same template set (for example, via a network mount or a synced folder), and lets a single
host maintain one template library that serves all of its projects. Sharing is safe only on a host that never calls
`unity:task-prefabs` `create_task_tool`, which resolves the template basename inside the Unity project's own
`Configurations/` folder and ignores this record. You MUST keep the record on the `Configurations/` folder of the local
sollertia-virtual-reality clone on any host that generates Unity tasks.

The templates directory must be set before any template authoring (`/task-templates`) work, because the authoring tools
resolve their target directory from the persisted record. Experiment configuration does not read that record:
`create_experiment_from_vr_template_tool` takes an explicit `template_path` and never calls
`get_task_templates_directory()`. The configured directory is still how candidates are found. Run
`discover_templates_tool` to enumerate the templates under it, then pass the absolute `path` it returns for the chosen
template to `create_experiment_from_vr_template_tool`. If the directory is empty after being set, templates must be
authored before they can be referenced by experiment configurations. The directory only needs to be re-configured if its
location on disk changes.

`/task-templates` owns what lives on a `TaskTemplate`, and `/experiment-configuration` owns what an experiment
configuration parameterizes on top of one.

---

## MCP tool surface

| Tool                                   | Purpose                                                                                    |
|----------------------------------------|--------------------------------------------------------------------------------------------|
| `set_working_directory_tool`           | Sets the local Sollertia working directory                                                 |
| `read_working_directory_tool`          | Reads the currently configured working directory                                           |
| `set_data_root_tool`                   | Sets the local Sollertia data root                                                         |
| `read_data_root_tool`                  | Reads the currently configured data root                                                   |
| `set_credentials_tool`                 | Copies a credentials file into the platform credentials directory under its canonical name |
| `read_credentials_tool`                | Reads the path to a category's configured credentials file                                 |
| `list_supported_credentials_tool`      | Enumerates the supported credentials categories and their canonical filenames              |
| `set_task_templates_directory_tool`    | Sets the directory holding YAML task templates                                             |
| `read_task_templates_directory_tool`   | Reads the currently configured task templates directory                                    |
| `get_platform_environment_status_tool` | Reports `required`, `configured`, and `ok` status for every component in one health report |

All `set_*` tools accept absolute paths and create the directory if it does not exist (working directory and data root)
or expect the file or directory to exist (credentials, templates). `set_credentials_tool` takes a `credentials` category
and a `file_path`, and requires the source file's extension to match the category's canonical filename, which is `.json`
for `google`. It copies the file rather than recording its path, so edits to the original file do not propagate until
the credentials are re-set. `set_task_templates_directory_tool` returns `{"success": false, "error": ...}` carrying the
underlying message for both of its rejections: a path that does not exist, and a path that exists but is not a
directory. Use `get_platform_environment_status_tool` as a one-call health check before handing off to any downstream
configuration skill. For the shape of every response this server returns, see the `Response contract` section of
`/assets-mcp-environment-setup`.

The three setters differ in how they persist what they are given. `set_task_templates_directory_tool` persists
`str(path.resolve())` while echoing the raw input, so `read_task_templates_directory_tool` legitimately returns a
different string than the one that was set whenever the input carried a symlink, a `..` segment, or a relative segment.
The working-directory and data-root setters persist the path exactly as given, which is why supplying an absolute path
matters more for those two.

### Required vs. optional components

`get_platform_environment_status_tool` distinguishes required from optional components in its per-component report and
computes `overall_ok` from the **required components only**. The report carries one `<category>_credentials` component
per supported credentials category:

| Component                  | `required` | When the host needs it                                                              |
|----------------------------|------------|-------------------------------------------------------------------------------------|
| `working_directory`        | `True`     | Always. Every other slsa workflow assumes the working directory is set              |
| `data_root`                | `False`    | Only when defaulting project creation, discovery, or inventory to a persisted root  |
| `task_templates_directory` | `False`    | Authoring task templates, and creating any experiment session of a VR-task type     |
| `google_credentials`       | `False`    | Only when reading subject metadata from or writing water-restriction logs to Sheets |

Each per-component dict carries `required`, `configured`, `ok`, and either `path` (when configured) or `error` (when
not). An optional unset component reports `configured=False` and `ok=False` but does **not** gate `overall_ok`.
`overall_ok=True` whenever every component with `required=True` is configured and accessible. A freshly bootstrapped
host with only the working directory configured is therefore a fully healthy slsa install with `overall_ok=True`. The
data root, credentials, and templates directory are configured as each workflow that depends on them comes online.

The `task_templates_directory` row's `required=False` reports what the **server** needs in order to start, not what
**session creation** needs. `SessionData.create()` resolves the persisted templates directory in order to cache the
`vr_configuration.yaml` snapshot into the new session's raw data directory. That cache is gated on two conditions
holding at once. The session must name an experiment (`experiment_name is not None`), and its session type must be a
member of `SESSION_TYPES_USING_VR_TASK`. When both hold, the getter runs, and it raises `FileNotFoundError` if the
record is unset. So `overall_ok=True` on a host with no templates directory is not evidence that session creation will
succeed there. See Step 5 for the full rule.

---

## Bootstrap workflow

You SHOULD run this workflow once per host. After it completes, the values are persisted and other assets plugin skills
can be used freely.

### Step 1: Verify MCP connectivity

Confirm the `sollertia-shared-assets` MCP server is connected. If not, hand off to `/assets-mcp-environment-setup`.

### Step 2: Set the working directory

Ask the user where they want Sollertia to cache local state if it has not already been decided. Common choices are
`~/sollertia/` or `~/.local/share/sollertia/` on Linux, `~/Library/Application Support/sollertia/` on macOS, and
`%LOCALAPPDATA%\sollertia\` on Windows. Then call:

```text
set_working_directory_tool(directory="<absolute path>")
```

Verify with `read_working_directory_tool`.

### Step 3: Set the data root (optional)

The data root is an **optional** platform component, carrying `required=False`, so leaving it unset does not break
`overall_ok`. Skip this step on hosts that always pass an explicit root to the project-hierarchy and session-discovery
tools. Set it when the host should remember where its project hierarchy lives so the `slsa get` CLI commands and the
discovery workflows can default to it:

```text
set_data_root_tool(directory="<absolute path to the project-hierarchy root>")
```

The tool creates the directory if it does not exist. Verify with `read_data_root_tool`.

### Step 4: Configure credentials (optional)

Credentials are **optional** platform components. Skip this step entirely on a host that neither reads subject metadata
from Google Sheets nor writes water-restriction logs to them. `get_platform_environment_status_tool` will still report
`overall_ok=True` there, because credentials carry `required=False`, and only the downstream tools that actually reach
the sheets will fail, with a clear error stating the credentials file has not been set.

If the user's project reads subject metadata from Google Sheets or writes water-restriction logs to them, configure the
`google` credentials category:

```text
set_credentials_tool(credentials="google", file_path="<absolute path to the source credentials JSON>")
```

The tool copies the source file to `<working-directory>/credentials/google_credentials.json`. Verify with
`read_credentials_tool(credentials="google")`. Use `list_supported_credentials_tool` to enumerate the supported
categories and their canonical filenames.

### Step 5: Configure task templates directory

The task templates directory carries `required=False` at the platform-status level because the slsa MCP server starts
without it, so leaving it unset does not break `overall_ok`. That flag reports what the server needs in order to start,
not what session creation needs. Two independent conditions require the directory:

1. **Authoring task templates.** `/task-templates` resolves its target directory from the persisted record and refuses
   to run until it is set.
2. **Creating an experiment session of a VR-task type.** `SessionData.create()` resolves the record when the session
   names an experiment (`experiment_name is not None`) **and** its session type is a member of
   `SESSION_TYPES_USING_VR_TASK`, in order to cache the `vr_configuration.yaml` snapshot beside the session's experiment
   configuration. The getter raises `FileNotFoundError` when the record is unset, so session creation fails outright.

You MUST set the templates directory on any host that creates experiment sessions of a VR-task type, even when that host
authors nothing. The "skip it if authoring runs elsewhere" carve-out therefore applies only to a host that neither
authors templates nor creates such sessions. An experiment session whose type is outside `SESSION_TYPES_USING_VR_TASK`,
and a VR-typed session that names no experiment, never reach the getter.

Task templates live in their own directory so they can be shared across projects on the same host. Set the path:

```text
set_task_templates_directory_tool(directory="<absolute path>")
```

Verify with `read_task_templates_directory_tool`. The setter persists the resolved path while the tool echoes the raw
input, so the two strings can legitimately differ. If the directory is empty, the user should populate it before
invoking `/task-templates`, which owns template authoring.

### Step 6: Verify templates are discoverable (only if Step 5 was run)

Skip this step if Step 5 was skipped, because there is no templates directory to verify. Otherwise call
`discover_templates_tool` once to confirm the directory layout is recognized. This is a read-only "natural share" of the
discover tool that `/task-templates` also exposes, and the call here exists only to validate the templates path was set
correctly. Do not inspect or modify any template content from this skill, because `/task-templates` owns that.

A `discover_templates_tool` success proves only that this record points at readable templates. It does not imply that
`unity:task-prefabs` `create_task_tool` will find them, because that tool resolves the template basename inside the
Unity Editor's own project folder rather than through this record.

### CLI equivalents

Every bootstrap step that writes a record has a `slsa` CLI equivalent, for a host operating without the MCP server.
These are the exact commands the library's own `FileNotFoundError` messages name when a record is unset.

| Step | Setter                                               | Reader                               |
|------|------------------------------------------------------|--------------------------------------|
| 2    | `slsa configure directory -d <path>`                 | `slsa get directory`                 |
| 3    | `slsa configure data-root -d <path>`                 | `slsa get data-root`                 |
| 4    | `slsa configure credentials -c <category> -f <file>` | `slsa get credentials -c <category>` |
| 5    | `slsa configure templates -d <path>`                 | `slsa get templates`                 |

`configure directory` and `configure data-root` accept non-existent paths (`exists=False`) and create them.
`configure templates` and `configure credentials --file` require the target to exist already (`exists=True`), and Click
rejects a missing target before the library is reached. A fifth setter, `slsa configure project -p <name>`, creates a
project directory under the data root. It resolves that root from `get_data_root()` and accepts no override, so it fails
on a host that skipped Step 3.

Once the bootstrap steps above are done, return to `experiment:pipeline`, which owns the canonical phase order for the
whole Sollertia lifecycle and names the phase that follows working-directory setup.

---

## When to re-run this skill

### Working directory

- **New host:** Always, because no other configuration skill can run until the working directory is set.
- **Relocating the working directory:** Step 2 to point at the new location. Note that relocating the working directory
  does **not** migrate the configuration files inside it. The user must either move those files manually or re-author
  them via `experiment:acquisition-system-design` and `forging:server-configuration`.
- **A configuration tool fails because the working directory cannot be resolved:** diagnose the exact condition with the
  table under "Diagnosing a path-record failure" below, then re-run Steps 1 and 2. A missing record typically follows a
  fresh OS install or a cleared `platformdirs` application directory.

### Data root

- **New host that should remember its project-hierarchy root:** Step 3 during initial bootstrap.
- **Relocating the Sollertia data root:** Step 3 to point at the new location. Setting the data root does **not** move
  any session data already on disk. Move the project directories first, then update the stored root.
- **Adding a persisted root to a host that previously passed explicit roots:** Step 3 only.

### Credentials

- **New host with Google Sheets integration:** Step 4 during initial bootstrap.
- **Rotating credentials:** Step 4 only. This is needed when the Google Cloud service-account key is regenerated or when
  switching to a different service account.
- **Adding Sheets integration to a host that previously skipped it:** Step 4 only.
- **Source credentials file edited or regenerated:** Step 4 to re-copy it. The platform reads the copy stored in its
  credentials directory, so changes to the original file do not propagate until the credentials are re-set.

### Task templates directory

- **New host that will author or consume templates:** Step 5 during initial bootstrap.
- **Templates directory relocated:** Step 5 to update the stored path, then step 6 to verify templates are still
  discoverable.
- **Switching to a shared network templates directory:** Step 5 to point at the new mount, then step 6. Do this only on
  a host that never calls `unity:task-prefabs` `create_task_tool`, which resolves a template basename under the Unity
  project's own `Assets/InfiniteCorridorTask/Configurations/` and ignores this record.

### Diagnosing a path-record failure

All three getters behind these settings (`get_working_directory`, `get_data_root`, and `get_task_templates_directory`)
raise `FileNotFoundError` under the same three conditions, evaluated in this order. The message states the condition and
names the `slsa configure` command that fixes it.

| Condition                    | Message fragment                                                                | Cause                                                           | Fix                                  |
|------------------------------|---------------------------------------------------------------------------------|-----------------------------------------------------------------|--------------------------------------|
| Record file missing          | `...as it has not been set.`                                                    | Never written on this host, or the cache directory was cleared  | Re-run the setter (Step 2, 3, or 5)  |
| Record empty after stripping | `...as the cached path record is empty.`                                        | A truncated or hand-edited record holding only line terminators | Re-run the setter                    |
| Directory absent from disk   | `...as the currently configured directory does not exist at the expected path.` | The record is intact, but its directory moved or is unmounted   | Re-set the path, or restore the tree |

The empty-record branch exists because `Path("")` resolves to the process working directory and would pass every later
check, silently redirecting every consumer. Note that a record holding only spaces is not empty by this rule, so it
falls through to the third condition instead.

Only the third condition is not fixed by re-running the setter with the same argument, because the stored path is stale
rather than missing. Either put the directory back where the record points, or re-run the setter against the directory's
new location. Note also that the existence check is `.exists()` rather than `.is_dir()`, so a regular file left at the
recorded path passes it, and the failure surfaces later in whichever consumer tries to open a subdirectory. The
templates getter is the one variation on the third message: it says "previously configured" rather than "currently
configured", and it is the only one of the three that interpolates the offending path.

---

## Related skills

This skill is a prerequisite for **every** other skill in the `assets` plugin. The relationships below summarize where
each downstream skill picks up after the working directory is set.

| Skill                                  | Relationship                                                                      |
|----------------------------------------|-----------------------------------------------------------------------------------|
| `/cli-reference`                       | Owns the `slsa get` and `slsa configure` command surface these records sit behind |
| `/assets-mcp-environment-setup`        | (sibling, run first if the MCP server is not connected)                           |
| `experiment:pipeline`                  | (owns the canonical lifecycle phase order; return there once bootstrap completes) |
| `experiment:acquisition-system-design` | Working directory                                                                 |
| `forging:server-configuration`         | Working directory                                                                 |
| `/library-extension`                   | Working directory                                                                 |
| `/task-templates`                      | Working directory + task templates directory                                      |
| `unity:task-prefabs`                   | Templates directory, pointed at Unity `Configurations/`                           |
| `/experiment-configuration`            | Working directory                                                                 |
| `/project-hierarchy`                   | Working directory, optionally the persisted data root                             |
| `/session-discovery`                   | Working directory. The persisted data root supplies the `root_directory` argument |
| `/session-data`                        | Working directory                                                                 |
| `/session-descriptors`                 | Working directory                                                                 |
| `/session-hardware-state`              | Working directory                                                                 |
| `mesoscope:mesoscope-vr-snapshots`     | Working directory                                                                 |
| `/data-assets`                         | None. The tools take absolute paths                                               |
| `/datasets`                            | Working directory                                                                 |

The Google credentials are consumed by the preprocessing-side capture that reads a read asset out of its Google Sheet,
which `experiment:google-sheets-processing` owns. The `/data-assets` tools themselves take absolute paths and need
neither the working directory nor the credentials.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Working directory exists and is readable (`read_working_directory_tool` returns expected path)
- [ ] Data root is set ONLY IF the host should default discovery and inventory to a persisted root, and is
      otherwise intentionally left unset, which is a healthy state rather than an error
- [ ] Google credentials file is set ONLY IF the project reads subject metadata from Google Sheets or writes
      water-restriction logs to them, and is otherwise intentionally left unset, which is a healthy state
- [ ] Task templates directory is set IF the host authors task templates, and MUST be set if the host creates
      experiment sessions whose session type is in SESSION_TYPES_USING_VR_TASK, because SessionData.create resolves
      it to cache the vr_configuration.yaml snapshot. Otherwise intentionally left unset (healthy, not an error)
- [ ] If the templates directory is set, `discover_templates_tool` returns at least the templates the
      user expects (or empty if none have been authored yet)
- [ ] `get_platform_environment_status_tool` returns `overall_ok=True`, counting required components only, since
      optional components carry `required=False` and do not gate the aggregate
```
