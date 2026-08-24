---
name: working-directory
description: >-
  Initializes the local Sollertia working directory, data root, platform credentials, and task
  templates directory via the sollertia-shared-assets MCP server. Prerequisite for every other
  assets-plugin skill. Use when setting up Sollertia on a new host, relocating the data root,
  or when configuration tools fail because the working directory is not set.
user-invocable: false
---

# Sollertia working directory setup

Initializes the local Sollertia working directory, the data root, and the supporting credentials
files and task templates directory used by `sollertia-shared-assets`. This is the first skill to
invoke on any host that will run Sollertia configuration or runtime tooling.

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
- Reading session-level data (see `/session-data`, `/session-descriptors`, `/session-hardware-state`,
  `/data-assets`, and `mesoscope:mesoscope-vr-snapshots`)
- Reading datasets (see `forging:datasets`)
- Diagnosing MCP server connectivity (see `/assets-mcp-environment-setup`)

---

## What lives in the working directory

The Sollertia working directory is the cache root for **host-machine-local** configuration state. It
is distinct from the long-term storage tier where session data lives. Setting the working directory
creates the `configuration/` and `credentials/` subdirectories:

```text
<working-directory>/
├── configuration/
│   ├── <system>_system_configuration.yaml   # Owned by sollertia-experiment (e.g. mesoscope_system_configuration.yaml)
│   └── server_configuration.yaml            # Owned by sollertia-forgery
└── credentials/
    └── google_credentials.json              # Configured via set_credentials_tool
```

The working directory path is persisted via `platformdirs` so it survives across CLI invocations and
MCP sessions on the same host. Other MCP tools resolve their default paths against this directory.

### configuration/

This skill initializes the `configuration/` subdirectory but never reads or writes its files. The
YAMLs inside are owned by downstream plugins:

- `<system>_system_configuration.yaml` — one per acquisition system, backed by a per-system
  configuration class in `sollertia-experiment` (for the current Mesoscope-VR reference system,
  `mesoscope_system_configuration.yaml` backed by `MesoscopeSystemConfiguration`); its authoring
  follows `experiment:acquisition-system-design` pattern.
- `server_configuration.yaml` — backed by `ServerConfiguration` in `sollertia-forgery`. Authored by
  `forging:server-configuration` skill.

Defer to each owning skill for the schema, the authoring workflow, and the rotation cadence.

### credentials/

The `credentials/` subdirectory stores the credentials files the platform uses to interact with
external services. Each supported credentials category maps to a canonical filename; the `google`
category — the Google Sheets service-account credentials JSON — is stored as `google_credentials.json`.
`set_credentials_tool` validates the source file and **copies** it into this subdirectory under the
canonical name, replacing any previously configured file for the same category. Because the platform
reads the copy, later edits to the original file do not propagate until the credentials are re-set.

Google credentials are required whenever a project reads animal metadata, surgery logs, or water
restriction records from Google Sheets. If no project on the host uses Google Sheets, the credentials
can be left unset — but downstream tools that fetch sheet data will fail until they are set.

---

## The data root

The Sollertia **data root** is the directory under which every project directory — and therefore every
animal and session directory — is stored on this host (`<data-root>/<project>/<animal>/<session>/`). It is
distinct from the working directory: the working directory caches host-local configuration, while the data
root anchors the acquired-data hierarchy. The two are commonly separate volumes (a fast local disk for
configuration, a large storage tier for sessions).

The data root path is persisted via `platformdirs`, independently of the working directory. Persisting it
lets the project-hierarchy and session-discovery workflows resolve the project tree without the caller
re-supplying the root each time, and it is the path the `slsa get` CLI commands (`slsa get projects`,
`slsa get experiments`, `slsa get data-root`) resolve against. The MCP discovery and inventory tools still
accept an explicit `root_directory` argument; reading the persisted data root with `read_data_root_tool` is
how the agent recalls "where projects live on this host" before passing that path to those tools.

The data root is **optional** at the platform-status level — it carries `required=False`, so leaving it
unset does not break `overall_ok`. Hosts that always supply an explicit root to the discovery tools can
skip it.

---

## Task templates directory

The task templates directory is a standalone directory (separate from the working directory) that holds
reusable `TaskTemplate` YAML files. Each template describes a complete behavioral paradigm: the VR
environment, the cue catalog, and the trial structures (each of which carries its own cue sequence, zone
geometry, and trigger type — there is no separate segment catalog at the template level). This is
typically the path to the local sollertia-virtual-reality repository's template directory:
`<local-repo>/Assets/InfiniteCorridorTask/Configurations/`. The MCP `create_task_tool` refuses
templates outside that directory (see `unity_tools.py`), and the Unity-side `CreateFromTemplate`
generator scans the same directory (see `unity:task-prefabs` for `create_task_tool` and
`unity:task-generator` for the generator's directory scan), so this directory is the single source of
truth. Keep it pointed at the canonical repo path.

Templates are **project-agnostic** — the same template can back many per-project experiment
configuration instances across different projects and even across different hosts.
Decoupling the templates directory from the working directory lets multiple hosts share the same
template set (for example, via a network mount or a synced folder). It also lets a single host maintain
one template library that serves all of its projects.

The templates directory must be set before any template authoring (`/task-templates`) or experiment
configuration (`/experiment-configuration`) work. If the directory is empty after being set, templates
must be authored before they can be referenced by experiment configurations. The directory only needs to
be re-configured if its location on disk changes.

A template defines **what is possible** (the full trial vocabulary within the linear infinite corridor), while an
experiment configuration picks **which template to use** and parameterizes it (state durations, trial weights, reward
volumes, project-specific overrides). Templates are authored by `/task-templates`; experiment configurations are
authored by `/experiment-configuration`.

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

All `set_*` tools accept absolute paths and create the directory if it does not exist (working directory
and data root) or expect the file/directory to exist (credentials, templates).
`set_credentials_tool` takes a `credentials` category and a `file_path`; it requires the source file's
extension to match the category's canonical filename (`.json` for `google`) and copies the file rather
than recording its path, so edits to the original file do not propagate until the credentials are re-set.
`set_task_templates_directory_tool` rejects paths that exist but are not directories with a `ValueError`
(in addition to the existence check). Use `get_platform_environment_status_tool` as a one-call health
check before handing off to any downstream configuration skill.

### Required vs. optional components

`get_platform_environment_status_tool` distinguishes required from optional components in its
per-component report and computes `overall_ok` from the **required components only**. The report
carries one `<category>_credentials` component per supported credentials category:

| Component                  | `required` | When the host needs it                                                           |
|----------------------------|------------|----------------------------------------------------------------------------------|
| `working_directory`        | `True`     | Always — every other slsa workflow assumes the working directory is set          |
| `data_root`                | `False`    | Only when defaulting discovery / inventory to a persisted project-hierarchy root |
| `task_templates_directory` | `False`    | The slsa server starts without it; needed to author task templates and configs   |
| `google_credentials`       | `False`    | Only when fetching subject metadata or water-restriction logs from Google Sheets |

Each per-component dict carries `required`, `configured`, `ok`, and either `path` (when configured) or
`error` (when not). An optional unset component reports `configured=False` and `ok=False` but does **not**
gate `overall_ok` — `overall_ok=True` whenever every component with `required=True` is configured and
accessible. A freshly bootstrapped host with only the working directory configured is therefore a fully healthy
slsa install with `overall_ok=True`; the data root, credentials, and templates directory are configured as each
workflow that depends on them comes online.

---

## Bootstrap workflow

You SHOULD run this workflow once per host. After it completes, the values are persisted and other
assets plugin skills can be used freely.

### Step 1: Verify MCP connectivity

Confirm the `sollertia-shared-assets` MCP server is connected. If not, hand off to
`/assets-mcp-environment-setup`.

### Step 2: Set the working directory

Ask the user where they want Sollertia to cache local state if it has not already been decided. A
choices are `~/sollertia/` or `~/.local/share/sollertia/` on Linux, `~/Library/Application Support/sollertia/`
on macOS, and `%LOCALAPPDATA%\sollertia\` on Windows. Then call:

```text
set_working_directory_tool(directory="<absolute path>")
```

Verify with `read_working_directory_tool`.

### Step 3: Set the data root (optional)

The data root is an **optional** platform component — it carries `required=False`, so leaving it unset
does not break `overall_ok`. Skip this step on hosts that always pass an explicit root to the
project-hierarchy and session-discovery tools. Set it when the host should remember where its project
hierarchy lives so the `slsa get` CLI commands and the discovery workflows can default to it:

```text
set_data_root_tool(directory="<absolute path to the project-hierarchy root>")
```

The tool creates the directory if it does not exist. Verify with `read_data_root_tool`.

### Step 4: Configure credentials (optional)

Credentials are **optional** platform components. Skip this step entirely on hosts that do not fetch
subject metadata or water-restriction logs from Google Sheets — `get_platform_environment_status_tool`
will still report `overall_ok=True` because credentials carry `required=False`. Only the downstream
tools that actually read sheets will fail (with a clear error stating the credentials file has not been
set) on a host that skipped this step.

If the user's project pulls animal metadata or water restriction data from Google Sheets, configure the
`google` credentials category:

```text
set_credentials_tool(credentials="google", file_path="<absolute path to the source credentials JSON>")
```

The tool copies the source file to `<working-directory>/credentials/google_credentials.json`. Verify
with `read_credentials_tool(credentials="google")`. Use `list_supported_credentials_tool` to enumerate
the supported categories and their canonical filenames.

### Step 5: Configure task templates directory

The task templates directory carries `required=False` at the platform-status level because the slsa MCP server
starts without it, so leaving it unset does not break `overall_ok`. It is needed to author the task templates and
experiment configurations that every acquisition system uses — the `/task-templates` and `/experiment-configuration`
skills refuse to run until it is set. Configure it during bootstrap unless those workflows will only run on a
different host.

Task templates live in their own directory so they can be shared across projects on the same host. Set the path:

```text
set_task_templates_directory_tool(directory="<absolute path>")
```

Verify with `read_task_templates_directory_tool`. If the directory is empty, the user should populate
it before invoking `/task-templates` (which owns template authoring) or `/experiment-configuration`.

### Step 6: Verify templates are discoverable (only if Step 5 was run)

Skip this step if Step 5 was skipped — there is no templates directory to verify. Otherwise call
`discover_templates_tool` once to confirm the directory layout is recognized. This is a read-only
"natural share" of the discover tool that is also exposed by `/task-templates` — the call here
exists only to validate the templates path was set correctly. Do not inspect or modify any template
content from this skill; that is owned by `/task-templates`.

---

## When to re-run this skill

### Working directory

- **New host:** Always — no other configuration skill can run until the working directory is set.
- **Relocating the working directory:** Step 2 to point at the new location. Note that relocating
  the working directory does **not** migrate the configuration files inside it. The user must either
  move those files manually or re-author them via `experiment:acquisition-system-design`
  and `forging:server-configuration`.
- **MCP tools fail with "working directory not set" or "no working directory":** Steps 1–2 to
  reinitialize. This typically happens after a fresh OS install or if the `platformdirs` persisted path
  was cleared.

### Data root

- **New host that should remember its project-hierarchy root:** Step 3 during initial bootstrap.
- **Relocating the Sollertia data root:** Step 3 to point at the new location. Setting the data root does
  **not** move any session data already on disk — move the project directories first, then update the
  stored root.
- **Adding a persisted root to a host that previously passed explicit roots:** Step 3 only.

### Credentials

- **New host with Google Sheets integration:** Step 4 during initial bootstrap.
- **Rotating credentials:** Step 4 only. This is needed when the Google Cloud service-account key is
  regenerated or when switching to a different service account.
- **Adding Sheets integration to a host that previously skipped it:** Step 4 only.
- **Source credentials file edited or regenerated:** Step 4 to re-copy it. The platform reads the copy
  stored in its credentials directory, so changes to the original file do not propagate until the
  credentials are re-set.

### Task templates directory

- **New host that will author or consume templates:** Step 5 during initial bootstrap.
- **Templates directory relocated:** Step 5 to update the stored path, then step 6 to verify templates
  are still discoverable.
- **Switching to a shared network templates directory:** Step 5 to point at the new mount, then step 6.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Working directory exists and is readable (`read_working_directory_tool` returns expected path)
- [ ] Data root is set ONLY IF the host should default discovery / inventory to a persisted root —
      otherwise intentionally left unset (this is a healthy state, not an error)
- [ ] Google credentials file is set ONLY IF the project fetches data from Google Sheets — otherwise
      intentionally left unset (this is a healthy state, not an error)
- [ ] Task templates directory is set ONLY IF the host authors templates or experiment configurations
      — otherwise intentionally left unset (this is a healthy state, not an error)
- [ ] If the templates directory is set, `discover_templates_tool` returns at least the templates the
      user expects (or empty if none have been authored yet)
- [ ] `get_platform_environment_status_tool` returns `overall_ok=True` (required components only —
      optional components carry `required=False` and do not gate the aggregate)
```

---

## Related skills

This skill is a prerequisite for **every** other skill in the 'assets' plugin. The relationships
below summarize where each downstream skill picks up after the working directory is set.

| Downstream skill                       | What it needs from this skill                            |
|----------------------------------------|----------------------------------------------------------|
| `/assets-mcp-environment-setup`        | (sibling — run first if the MCP server is not connected) |
| `experiment:acquisition-system-design` | Working directory                                        |
| `forging:server-configuration`         | Working directory                                        |
| `/task-templates`                      | Working directory + task templates directory             |
| `unity:task-prefabs`                   | Templates directory, pointed at Unity `Configurations/`  |
| `/experiment-configuration`            | Working directory                                        |
| `/project-hierarchy`                   | Working directory; optionally the persisted data root    |
| `/session-data`                        | Working directory                                        |
| `/session-descriptors`                 | Working directory                                        |
| `/session-hardware-state`              | Working directory                                        |
| `mesoscope:mesoscope-vr-snapshots`     | Working directory                                        |
| `/data-assets`                         | Working directory + Google credentials                   |
| `forging:datasets`                     | Working directory                                        |
