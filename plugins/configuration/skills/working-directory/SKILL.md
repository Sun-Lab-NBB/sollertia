---
name: working-directory
description: >-
  Initializes the local Sollertia working directory and the Google Sheets credentials and task templates
  directory paths for sollertia-shared-assets. Covers the set/read MCP tools, expected directory layout,
  and the bootstrap order required before any other configuration work. Use when setting up Sollertia on
  a new host, when the configuration MCP tools fail with "working directory not set", or when relocating
  the Sollertia data root.
user-invocable: true
---

# Sollertia working directory setup

Initializes the local Sollertia working directory and the supporting credentials / template paths used
by `sollertia-shared-assets`. This is the first skill to invoke on any host that will run Sollertia
configuration or runtime tooling.

---

## Scope

**Covers:**
- Setting and reading the local working directory
- Setting and reading the Google Sheets credentials path
- Setting and reading the task templates directory
- The bootstrap order required by other configuration MCP tools

**Does not cover:**
- Authoring system configuration (see `/system-configuration`)
- Authoring server configuration (see `/server-configuration`)
- Authoring task templates (see `/task-templates`)
- Authoring experiment configurations (see `/experiment-configuration`)
- Creating projects (see `/project-hierarchy`)
- Reading session-level data (see `/session-data`, `/session-descriptors`, `/session-snapshots`,
  `/subject-metadata`)
- Reading datasets (see `/datasets`)
- Diagnosing MCP server connectivity (see `/mcp-environment-setup`)

---

## What lives in the working directory

The Sollertia working directory is the cache root for **host-machine-local** configuration and runtime
state. It is distinct from the long-term storage tier where session data lives. A typical layout:

```text
<working-directory>/
├── system_configuration.yaml         # MesoscopeSystemConfiguration (or other acquisition system)
├── server_configuration.yaml         # ServerConfiguration for remote storage transfer
├── google_credentials.json           # Google Sheets API credentials (path is configurable)
└── ...
```

The path is persisted via `platformdirs` so it survives across CLI invocations and MCP sessions on the
same host. Other MCP tools resolve their default paths against this directory.

---

## MCP tool surface

| Tool                                  | Purpose                                                           |
|---------------------------------------|-------------------------------------------------------------------|
| `set_working_directory_tool`          | Sets the local Sollertia working directory                        |
| `read_working_directory_tool`         | Reads the currently configured working directory                  |
| `set_google_credentials_tool`         | Sets the path to the Google Sheets credentials JSON file          |
| `read_google_credentials_tool`        | Reads the currently configured Google credentials path            |
| `set_task_templates_directory_tool`   | Sets the directory holding YAML task templates                    |
| `read_task_templates_directory_tool`  | Reads the currently configured task templates directory           |

All `set_*` tools accept absolute paths and create the directory if it does not exist (working directory)
or expect the file/directory to exist (credentials, templates).

---

## Bootstrap workflow

You SHOULD run this workflow once per host. After it completes, the values are persisted and other
configuration plugin skills can be used freely.

### Step 1: Verify MCP connectivity

Confirm the `sollertia-shared-assets` MCP server is connected. If not, hand off to
`/mcp-environment-setup`.

### Step 2: Set the working directory

Ask the user where they want Sollertia to cache local state if it has not already been decided. A
typical choice on Linux is `~/sollertia/` or `~/.local/share/sollertia/`. Then call:

```text
set_working_directory_tool(directory="<absolute path>")
```

Verify with `read_working_directory_tool`.

### Step 3: Configure Google Sheets credentials

If the user's project pulls animal metadata or water restriction data from Google Sheets (typical for
the Sun Lab Mesoscope-VR pipeline), set the credentials' path:

```text
set_google_credentials_tool(credentials_path="<absolute path to credentials.json>")
```

Verify with `read_google_credentials_tool`. If the user does not use Google Sheets, this step can be
skipped — but downstream tools that read from sheets will fail until it is set.

### Step 4: Configure task templates directory

Task templates live in their own directory so they can be shared across projects on the same host. Set
the path:

```text
set_task_templates_directory_tool(directory="<absolute path>")
```

Verify with `read_task_templates_directory_tool`. If the directory is empty, the user should populate
it before invoking `/task-templates` (which owns template authoring) or `/experiment-configuration`.

### Step 5: Verify templates are discoverable

After the templates directory is set, call `discover_templates_tool` once to confirm the directory
layout is recognized. This is a read-only "natural share" of the discover tool that is also exposed by
`/task-templates` — the call here exists only to validate the templates path was set correctly. Do not
inspect or modify any template content from this skill; that is owned by `/task-templates`.

---

## When to re-run this skill

- **New host:** Always.
- **Relocating the Sollertia data root:** Run steps 2 onward to point everything at the new location.
- **Rotating Google credentials:** Step 3 only.
- **Adding a new shared templates directory:** Step 4 only.
- **MCP tools fail with "no working directory":** Steps 1–2 to reinitialize.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Working directory exists and is readable (`read_working_directory_tool` returns expected path)
- [ ] Google credentials path is set if the project requires Sheets access
- [ ] Task templates directory is set and discoverable
- [ ] discover_templates_tool returns at least the templates the user expects (or empty if none yet)
```

---

## Related skills

This skill is a prerequisite for **every** other skill in the configuration plugin. The relationships
below summarize where each downstream skill picks up after the working directory is set.

| Downstream skill              | What it needs from this skill                                         |
|-------------------------------|-----------------------------------------------------------------------|
| `/mcp-environment-setup`      | (sibling — run first if the MCP server is not connected)              |
| `/system-configuration`       | Working directory                                                     |
| `/server-configuration`       | Working directory                                                     |
| `/task-templates`             | Working directory + task templates directory                          |
| `/experiment-configuration`   | Working directory                                                     |
| `/project-hierarchy`          | Working directory                                                     |
| `/session-data`               | Working directory                                                     |
| `/session-descriptors`        | Working directory                                                     |
| `/session-snapshots`          | Working directory                                                     |
| `/subject-metadata`           | Working directory + Google credentials                                |
| `/datasets`                   | Working directory                                                     |
