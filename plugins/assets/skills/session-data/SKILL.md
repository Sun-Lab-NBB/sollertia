---
name: session-data
description: >-
  Reads the canonical SessionData marker file for a Sollertia session via the slsa MCP server,
  exposes the SessionTypes enum surface, and reports per-session and batch-wide lifecycle status.
  Owns validate_session_tool, get_session_status_tool, and get_batch_session_status_overview_tool.
  Use when inspecting an individual session, confirming a session marker file exists, enumerating
  supported session type strings, validating a session's file inventory against its session_type,
  or auditing lifecycle progress across every session under the data root. SessionData is written
  only by the acquisition runtime — this skill is read-only for that file.
user-invocable: true
---

# Sollertia session data

Reads the canonical `SessionData` marker file that lives inside every Sollertia session directory,
exposes the `SessionTypes` enum surface, and reports session lifecycle status. Uses the `slsa mcp`
MCP server. This skill is the **exclusive** owner of `validate_session_tool`,
`get_session_status_tool`, and `get_batch_session_status_overview_tool` — no other skill in the
marketplace may call these.

`SessionData` itself has **no setter**. It is written only by the acquisition runtime at session
start. The discovery side of "which descriptors exist for a session" is included here as a natural-share query.

---

## Scope

**Covers:**
- Reading the `SessionData` marker file
- Discovering descriptor files inside a specific session directory
- Listing the canonical `SessionTypes` enum values
- The structural anatomy of a Sollertia session directory
- Validating that a session has every file required by its `session_type` (`validate_session_tool`)
- Reading per-session lifecycle status (`get_session_status_tool`)
- Aggregating lifecycle status across every session under the data root
  (`get_batch_session_status_overview_tool`)

**Does not cover:**
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing the per-session `MesoscopeHardwareState` snapshot (see
  `/session-hardware-state`)
- Reading or writing the per-session Zaber and mesoscope-objective position snapshots (see the
  experiment plugin's `/session-snapshots`)
- Reading the frozen system configuration captured at session start (see the experiment plugin's
  `/system-configuration`)
- Reading the frozen experiment configuration captured at session start (see
  `/experiment-configuration` for `read_session_experiment_configuration_tool`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering projects, animals, or sessions (see `/project-hierarchy`)
- Datasets that aggregate sessions (see forging plugin's `/datasets`)
- Preprocessing, deleting, or migrating sessions (see the experiment plugin's
  `/managing-session-data`)

---

## What is a session

A **session** is a single, contiguous data acquisition run performed on one animal, under one
project, with one acquisition system. Sessions are the atomic unit of acquired data — every
descriptor, snapshot, dataset, and processing pipeline output is keyed off one or more
sessions. A session bundles the canonical `SessionData` marker, the per-session-type
descriptor, the frozen system and experiment configurations, hardware-state and position
snapshots, and the raw acquired data itself.

The acquisition runtime is the only authorized creator of a session. This skill is **read-only**
with respect to `SessionData`; the read patterns and lifecycle status checks documented here
operate on sessions that already exist on disk.

### How sessions are named

A session's name is a **UTC timestamp accurate to microseconds**, of the shape
`YYYY-MM-DD-HH-MM-SS-ffffff` (for example `2026-04-19-15-04-23-789012`). Microsecond precision
makes name collisions between sessions started on the same machine impossible, and lexicographic
sort order coincides with chronological order — sorting an animal directory's children yields
sessions in the order they were acquired.

The full session path is `<root>/<project>/<animal>/<session_name>/`. Sessions are created only
on the acquisition machine; storage and processing destinations only ever see sessions that the
acquisition runtime has already produced and pushed downstream.

### Why the structure splits `raw_data/` from `processed_data/`

Every session's contents are partitioned into two top-level subdirectories with very different
lifecycles:

- **`raw_data/`** is the **immutable record** of what was acquired. It holds the original sensor
  data, the `SessionData` marker, the frozen configuration snapshots, the hardware-state
  snapshot, and the per-session-type descriptor. Once acquisition is complete the contents of
  `raw_data/` are not supposed to change.
- **`processed_data/`** is the **accreted** record of derived outputs. It does not exist at
  session creation and only appears when downstream processing pipelines start writing into it.
  Each pipeline owns its own subdirectory under `processed_data/`. Because the contents are
  always derivable from `raw_data/` plus configuration, `processed_data/` can be deleted and
  regenerated at any time without losing scientific data.

The absence of `processed_data/` on a freshly acquired session is therefore expected, not an
error. For the catalog of which pipelines write where, defer to the owning skills (forging
plugin's processing skills, cindra plugin's processing skills, the upstream ataraxis log-processing
skills) — they own their respective output paths and tracker conventions.

### The `nk.bin` initialization marker

The acquisition runtime writes a zero-byte `nk.bin` file inside `raw_data/` when a session is
first created and removes it once acquisition has progressed past the point where the session
should be preserved. The presence of `nk.bin` therefore identifies an incomplete session that is
safe to purge; its absence identifies a real acquired session that must be preserved.
`get_session_status_tool` and `get_batch_session_status_overview_tool` use this marker as their
sole "incomplete" signal.

---

## Anatomy of a session directory

Every Sollertia session is a directory whose YAML files live **inside `raw_data/`** (not at the
session root):

```text
<session>/
├── raw_data/                                  # acquired data and frozen metadata (written by the acquisition runtime)
│   ├── session_data.yaml                      # SessionData marker (THIS SKILL)
│   ├── <descriptor>.yaml                      # /session-descriptors (filename per session_type)
│   ├── system_configuration.yaml              # frozen system config (owned by sollertia-experiment)
│   ├── experiment_configuration.yaml          # /experiment-configuration (frozen, experiment sessions only)
│   ├── hardware_state.yaml                    # /session-hardware-state
│   ├── zaber_positions.yaml                   # experiment plugin /session-snapshots
│   ├── mesoscope_positions.yaml               # experiment plugin /session-snapshots
│   ├── nk.bin                                 # incomplete-session marker (removed when runtime initializes)
│   └── ... acquired data files ...
└── processed_data/                            # populated by experiment plugin /managing-session-data
```

`discover_sessions_tool` (owned by `/project-hierarchy`) walks the data root looking for
`session_data.yaml` markers. The session root is always two directory levels above the marker
(`<session>/raw_data/session_data.yaml`). This skill owns reading the marker after it has been
discovered.

All session-path arguments below accept **either the session root directory or its `raw_data/`
subdirectory** — `_resolve_session_root` normalizes both forms to the
canonical session root before any tool runs.

---

## SessionTypes

The canonical `SessionTypes` enum values are `lick training`, `run training`, `window checking`,
and `mesoscope experiment`. Use `list_supported_session_types_tool` for the authoritative list
(it also returns each type's descriptor filename and dataclass). Per-type descriptor file
mapping and schemas are owned by `/session-descriptors`.

---

## MCP tool surface

| Tool                                     | Purpose                                                                      |
|------------------------------------------|------------------------------------------------------------------------------|
| `read_session_data_tool`                 | Reads the `SessionData` marker for a session                                 |
| `discover_session_descriptors_tool`      | Lists the descriptor file(s) in a specific session directory                 |
| `list_supported_session_types_tool`      | Returns the canonical `SessionTypes` enum strings                            |
| `validate_session_tool`                  | Verifies a session has every file required by its `session_type` (exclusive) |
| `get_session_status_tool`                | Returns lifecycle status for one session (exclusive)                         |
| `get_batch_session_status_overview_tool` | Aggregates lifecycle status across every session under the root (exclusive)  |

`read_session_data_tool` and `list_supported_session_types_tool` are owned by this skill in the sense
that this is where the read patterns are documented and where other skills should hand off when they
need them. They are read-only and may also be called as natural shares.

`discover_session_descriptors_tool` is the discovery side of the descriptor read workflow. It lives here
(rather than in `/session-descriptors`) because it answers "what does this session contain?" — a
session-level question — not "what is in this descriptor file?" — a descriptor-level question. Other
skills may call it as a natural share.

`SessionData` is **not writeable** through the slsa MCP layer. It is created by the acquisition
runtime at session start and never modified afterward. There is no `write_session_data_tool` and there will not be one.

---

## Workflows

### Inspecting a specific session

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`).
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
6. **Hand off to `/session-hardware-state`** to read the frozen `MesoscopeHardwareState`.
7. **Hand off to the experiment plugin's `/session-snapshots`** to read the Zaber and
   mesoscope-objective position snapshots.
8. **Hand off to `/experiment-configuration`** for the frozen experiment configuration via
   `read_session_experiment_configuration_tool`.
9. **Hand off to the experiment plugin's `/system-configuration`** for the frozen
   `system_configuration.yaml` snapshot.

### Querying supported session types

```text
list_supported_session_types_tool()
```

Use this when you need to validate a session-type string before using it in another tool call (e.g.,
when handing off to `/session-descriptors` to read a descriptor).

---

## Session lifecycle status

A Sollertia session moves through a small set of lifecycle states. This skill exposes three tools
that report where each session sits in that pipeline.

### Validating a single session's file inventory

```text
validate_session_tool(session_path="<absolute>")
```

Reads the session's `SessionData` marker, determines the expected file inventory from its
`session_type`, and reports any missing files. The current checks are:

- The descriptor file (filename derived from `session_type`).
- The frozen `experiment_configuration.yaml` (only required when `session_type` is
  `mesoscope experiment`).
- The frozen `system_configuration.yaml`.

Returns `valid` (True / False), an `issues` list, a `summary` (session_name, project, animal,
session_type, incomplete flag), and the resolved `session_path`. The check does **not** verify
hardware-state, position snapshots, or the contents of `raw_data/` beyond these files. Use this
before handing off to the experiment plugin's `/managing-session-data` for preprocessing.

### Per-session lifecycle status

```text
get_session_status_tool(session_path="<absolute>")
```

Returns the session's coarse-grained status by inspecting two signals only:

- The presence of the `nk.bin` incomplete-session marker in `raw_data/`.
- Whether `processed_data/` exists and contains any files.

The tool returns `status` (one of `incomplete`, `acquired`, `processed`), the boolean `incomplete`
and `has_processed_data` flags, and the resolved `session_path`. It does **not** report transferred
or dataset-member states, does **not** include transition timestamps, and does **not** enumerate
the files that triggered the inference.

### Batch status overview

```text
get_batch_session_status_overview_tool(root_directory="<absolute path to data root>")
```

Walks every session under the supplied data root and applies the same coarse `incomplete` /
`acquired` / `processed` classification per session. The `root_directory` argument is required
(slsa no longer auto-resolves a data root after the system configuration moved to
`sollertia-experiment`). Returns `counts` (per-status totals plus an `error` bucket for sessions
that failed to load), `sessions` (the per-session entries), `total_sessions`, and the resolved
`root_directory`. It is a read-only aggregation.

Typical workflow:

1. Call `get_batch_session_status_overview_tool(root_directory=…)` to get the summary.
2. For any session in an unexpected state, drill in with `get_session_status_tool(session_path=…)`.
3. For sessions reported as incomplete, call `validate_session_tool(session_path=…)` to identify
   missing files.
4. Hand off to `/session-descriptors`, `/session-hardware-state`, the experiment plugin's
   `/session-snapshots`, or the experiment plugin's `/managing-session-data` to remediate.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] read_session_data_tool returned an intact marker before any further inspection
- [ ] Discovered descriptor files via discover_session_descriptors_tool before reading
- [ ] Did not attempt to write SessionData (not supported)
- [ ] validate_session_tool was called before handing off to the experiment plugin's
      /managing-session-data for preprocessing
- [ ] Handed off to /session-descriptors, /session-hardware-state, /subject-metadata,
      /experiment-configuration, or the experiment plugin's /session-snapshots for any read that
      goes deeper than the marker
```

---

## Related skills

| Skill                                      | Relationship                                                          |
|--------------------------------------------|-----------------------------------------------------------------------|
| `/assets-mcp-environment-setup`            | Run first if the MCP server is not connected                          |
| `/project-hierarchy`                       | Owns `discover_sessions_tool` and the project tree walk               |
| `/session-descriptors`                     | Sibling — owns the per-session descriptor read/write/schema           |
| `/session-hardware-state`                  | Sibling — owns the per-session `MesoscopeHardwareState` snapshot      |
| experiment plugin `/session-snapshots`     | Owns the frozen Zaber and mesoscope-objective position snapshots      |
| `/subject-metadata`                        | Sibling — owns animal-scoped subject records                          |
| experiment plugin `/system-configuration`  | Authors the system configuration consumed at session start            |
| `/experiment-configuration`                | Owns `read_session_experiment_configuration_tool` (frozen exp config) |
| forging plugin `/datasets`                 | Datasets aggregate sessions                                           |
| experiment plugin `/managing-session-data` | Preprocesses, migrates, and deletes sessions                          |
