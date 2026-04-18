---
name: session-data
description: >-
  Reads the canonical SessionData marker file for a Sollertia session via the sl-configure MCP server,
  and exposes the SessionTypes enum surface. Use when inspecting an individual session, confirming a
  session marker file exists, or enumerating supported session type strings. SessionData is written only
  by sl-run at runtime — this skill is read-only for that file.
user-invocable: true
---

# Sollertia session data

Reads the canonical `SessionData` marker file that lives inside every Sollertia session directory and
exposes the `SessionTypes` enum surface. Uses the `sl-configure mcp` MCP server.

This skill has **no exclusive setters**. `SessionData` is written only by `sl-run` at runtime. The
discovery side of "which descriptors exist for a session" is included here as a natural-share query.

---

## Scope

**Covers:**
- Reading the `SessionData` marker file
- Discovering descriptor files inside a specific session directory
- Listing the canonical `SessionTypes` enum values
- The structural anatomy of a Sollertia session directory

**Does not cover:**
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing per-session frozen runtime snapshots — hardware state, Zaber positions, mesoscope
  positions (see `/session-snapshots`)
- Reading the frozen system configuration captured at session start (see `/system-configuration` for
  `read_session_system_configuration_tool`)
- Reading the frozen experiment configuration captured at session start (see `/experiment-configuration`
  for `read_session_experiment_configuration_tool`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering projects, animals, or sessions (see `/project-hierarchy`)
- Datasets that aggregate sessions (see `/datasets`)
- Initial working directory setup (see `/working-directory`)
- Preprocessing, deleting, or migrating sessions (experiment plugin `/data-management`)

---

## Anatomy of a session directory

Every Sollertia session is a directory containing:

```text
<session-id>/
├── session_data.yaml                    # SessionData — the canonical marker (THIS SKILL)
├── <session-type>_descriptor.yaml       # /session-descriptors
├── system_configuration.yaml            # /system-configuration (frozen)
├── experiment_configuration.yaml        # /experiment-configuration (frozen, experiment sessions only)
├── mesoscope_hardware_state.yaml        # /session-snapshots
├── zaber_positions.yaml                 # /session-snapshots
├── mesoscope_positions.yaml             # /session-snapshots
├── raw_data/                            # acquired data (no skill — written by sl-run)
└── processed_data/                      # populated by experiment plugin /data-management
```

The `SessionData` file is the discovery marker — `discover_sessions_tool` (owned by `/project-hierarchy`)
walks the working directory tree and recognizes any directory containing a `session_data.yaml`. This
skill owns reading the marker after it has been discovered.

---

## SessionTypes

| `SessionTypes` value     | Descriptor file (owned by `/session-descriptors`) |
|--------------------------|---------------------------------------------------|
| `lick training`          | `lick_training_descriptor.yaml`                   |
| `run training`           | `run_training_descriptor.yaml`                    |
| `window checking`        | `window_checking_descriptor.yaml`                 |
| `mesoscope experiment`   | `mesoscope_experiment_descriptor.yaml`            |

---

## MCP tool surface

| Tool                                  | Purpose                                                          |
|---------------------------------------|------------------------------------------------------------------|
| `read_session_data_tool`              | Reads the `SessionData` marker for a session                     |
| `discover_session_descriptors_tool`   | Lists the descriptor file(s) in a specific session directory     |
| `list_supported_session_types_tool`   | Returns the canonical `SessionTypes` enum strings                |

`read_session_data_tool` and `list_supported_session_types_tool` are owned by this skill in the sense
that this is where the read patterns are documented and where other skills should hand off when they
need them. They are read-only and may also be called as natural shares.

`discover_session_descriptors_tool` is the discovery side of the descriptor read workflow. It lives here
(rather than in `/session-descriptors`) because it answers "what does this session contain?" — a
session-level question — not "what is in this descriptor file?" — a descriptor-level question. Other
skills may call it as a natural share.

`SessionData` is **not writeable** through the slsa MCP layer. It is created by `sl-run` at session
start and never modified afterward. There is no `write_session_data_tool` and there will not be one.

---

## Workflows

### Inspecting a specific session

1. **Verify prerequisites:**
   - MCP server connected (else `/mcp-environment-setup`).
   - Working directory set (else `/working-directory`).
2. **Locate the session.** Hand off to `/project-hierarchy` to call `discover_sessions_tool` with the
   appropriate filters (project, animal, session type, date range).
3. **Read the marker:**
   ```text
   read_session_data_tool(session_path="<absolute>")
   ```
4. **Discover the descriptor file present in the session:**
   ```text
   discover_session_descriptors_tool(session_path="<absolute>")
   ```
5. **Hand off to `/session-descriptors`** to read the descriptor contents.
6. **Hand off to `/session-snapshots`** to read the frozen hardware / Zaber / mesoscope positions.
7. **Hand off to `/system-configuration`** for the frozen system configuration via
   `read_session_system_configuration_tool`.
8. **Hand off to `/experiment-configuration`** for the frozen experiment configuration via
   `read_session_experiment_configuration_tool`.

### Querying supported session types

```text
list_supported_session_types_tool()
```

Use this when you need to validate a session-type string before using it in another tool call (e.g.,
when handing off to `/session-descriptors` to read a descriptor).

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] read_session_data_tool returned an intact marker before any further inspection
- [ ] Discovered descriptor files via discover_session_descriptors_tool before reading
- [ ] Did not attempt to write SessionData (not supported)
- [ ] Handed off to /session-descriptors, /session-snapshots, /subject-metadata, /system-configuration,
      or /experiment-configuration for any read that goes deeper than the marker
```

---

## Related skills

| Skill                                | Relationship                                                              |
|--------------------------------------|---------------------------------------------------------------------------|
| `/working-directory`                 | Required prerequisite — must be run first                                 |
| `/mcp-environment-setup`             | Run first if the MCP server is not connected                              |
| `/project-hierarchy`                 | Owns `discover_sessions_tool` and the project tree walk                   |
| `/session-descriptors`               | Sibling — owns the per-session descriptor read/write/schema               |
| experiment plugin `/session-snapshots` | Owns the frozen hardware / Zaber / mesoscope position snapshots         |
| `/subject-metadata`                  | Sibling — owns animal-scoped subject records                              |
| experiment plugin `/system-configuration` | Owns `read_session_system_configuration_tool` (frozen system config)   |
| `/experiment-configuration`          | Owns `read_session_experiment_configuration_tool` (frozen exp config)     |
| forging plugin `/datasets`           | Datasets aggregate sessions                                               |
| experiment plugin `/data-management` | Preprocesses, migrates, and deletes sessions via `sl-manage` MCP          |
