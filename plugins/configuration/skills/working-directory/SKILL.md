---
name: working-directory
description: >-
  Initializes the local Sollertia working directory and the Google Sheets credentials and task templates
  directory paths for sollertia-shared-assets. Covers the set/read MCP tools, expected directory layout,
  the bootstrap order required before any other configuration work, and detailed what/why/when context
  for every configurable asset (working directory, Google credentials, task templates directory, and the
  configuration files that live inside the working directory). Use when setting up Sollertia on a new
  host, when the configuration MCP tools fail with "working directory not set", when relocating the
  Sollertia data root, or when the user asks what an asset is, why it exists, or when to configure it.
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
- Explaining what each configurable asset is, why it exists, and when it is needed

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
├── system_configuration.yaml         # Acquisition system configuration (schema depends on system type)
├── server_configuration.yaml         # ServerConfiguration for remote storage transfer
├── google_credentials.json           # Google Sheets API credentials (path is configurable)
└── ...
```

The path is persisted via `platformdirs` so it survives across CLI invocations and MCP sessions on the
same host. Other MCP tools resolve their default paths against this directory.

### system_configuration.yaml

The system configuration YAML file captures everything that is host-machine-specific and stable across
sessions. Its schema depends on the acquisition system type — the only currently supported type is
`MesoscopeSystemConfiguration`, but the design accommodates additional system types. For the mesoscope
system, this includes local storage paths, camera indices and encoding parameters, microcontroller USB
ports and hardware calibration data, Zaber motor ports, MQTT broker settings, and Google Sheets IDs for
animal metadata. The acquisition runtime (`sl-run`) reads this file once at session start to know how
to talk to every piece of hardware on the rig — without it, the hardware layer cannot be initialized.

It is typically authored once when a new acquisition PC is brought online and rarely changed afterward.
Modifications happen when hardware is swapped (new camera, re-calibrated valve, different USB port
assignment) or when Google Sheet IDs rotate. Authoring is owned by `/system-configuration`.

### server_configuration.yaml

The `ServerConfiguration` YAML file stores the remote server hostname or IP, the SSH credentials path
on the local machine, the remote storage root where preprocessed sessions are deposited, and optional
per-project storage path overrides. After a session is preprocessed locally, the data management
pipeline (`sl-manage`) uses this file to transfer the output to a long-term remote storage tier
(typically a cloud compute server).

It is authored when a new acquisition PC is connected to a remote server, then updated when the server
hostname changes, SSH keys are rotated, or the remote storage path moves. It rotates independently of
the system configuration — the two live side by side in the working directory but have different
lifecycles. Authoring is owned by `/server-configuration`.

### google_credentials.json

A Google Cloud service-account credentials JSON file used to authenticate with the Google Sheets API.
Subject metadata — surgery logs, water restriction records, implant details, injection histories, and
drug administration records — lives in Google Sheets as the upstream source of truth. The
`sollertia-shared-assets` library stores and retrieves the credentials path, but the actual Sheets
reading happens in `sollertia-experiment` during session preprocessing (via `SurgeryLog`, `WaterLog`,
and related classes). The credentials file authorizes that access.

This file is required whenever a project pulls animal metadata from Google Sheets, which is the typical
workflow for acquisition pipelines that track animal welfare and surgical history. If no project on the
host uses Google Sheets, it can be omitted — but preprocessing steps that fetch sheet data will fail
until the credentials path is set. The credentials path can point anywhere on the filesystem; it does
not have to live inside the working directory, though that is a common default.

---

## Task templates directory

The task templates directory is a standalone directory (separate from the working directory) that holds
reusable `TaskTemplate` YAML files. Each template describes a complete behavioral paradigm: the VR
environment, the cue catalog, the segment layout, the available trial types, and the trial structure.
This is typically the path to the local sollertia-unity-tasks repository's template directory:
`<local-repo>/Assets/InfiniteCorridorTask/Configurations/`.

Templates are **project-agnostic** — the same template can back many per-project experiment
configuration instances across different projects and even across different hosts.
Decoupling the templates directory from the working directory lets multiple hosts share the same
template set (for example, via a network mount or a synced folder). It also lets a single host maintain
one template library that serves all of its projects.

The templates directory must be set before any template authoring (`/task-templates`) or experiment
configuration (`/experiment-configuration`) work. If the directory is empty after being set, templates
must be authored before they can be referenced by experiment configurations. The directory only needs to
be re-configured if its location on disk changes.

A template defines **what is possible** (the full trial vocabulary), while an experiment configuration
picks **which template to use** and parameterizes it (state durations, trial weights, reward volumes,
project-specific overrides). Templates are authored by `/task-templates`; experiment configurations are
authored by `/experiment-configuration`.

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
acquisition pipelines that track animal welfare and surgical history), set the credentials' path:

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

### Working directory

- **New host:** Always — no other configuration skill can run until the working directory is set.
- **Relocating the Sollertia data root:** Step 2 to point at the new location. Note that relocating the
  working directory does **not** migrate the configuration files inside it. The user must either move
  those files manually or re-author them via `/system-configuration` and `/server-configuration`.
- **MCP tools fail with "working directory not set" or "no working directory":** Steps 1–2 to
  reinitialize. This typically happens after a fresh OS install or if the `platformdirs` persisted path
  was cleared.

### Google credentials

- **New host with Google Sheets integration:** Step 3 during initial bootstrap.
- **Rotating credentials:** Step 3 only. This is needed when the Google Cloud service-account key is
  regenerated or when switching to a different service account.
- **Adding Sheets integration to a host that previously skipped it:** Step 3 only.
- **Credentials file moved on disk:** Step 3 to update the stored path.

### Task templates directory

- **New host that will author or consume templates:** Step 4 during initial bootstrap.
- **Templates directory relocated:** Step 4 to update the stored path, then step 5 to verify templates
  are still discoverable.
- **Switching to a shared network templates directory:** Step 4 to point at the new mount, then step 5.

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
| experiment plugin `/system-configuration` | Working directory                                         |
| forging plugin `/server-configuration` | Working directory                                            |
| `/task-templates`             | Working directory + task templates directory                          |
| `/experiment-configuration`   | Working directory                                                     |
| `/project-hierarchy`          | Working directory                                                     |
| `/session-data`               | Working directory                                                     |
| `/session-descriptors`        | Working directory                                                     |
| experiment plugin `/session-snapshots` | Working directory                                            |
| `/subject-metadata`           | Working directory + Google credentials                                |
| forging plugin `/datasets`    | Working directory                                                     |
