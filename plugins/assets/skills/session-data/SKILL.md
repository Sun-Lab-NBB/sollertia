---
name: session-data
description: >-
  Reads SessionData markers and reports per-session and batch-wide lifecycle status via the
  sollertia-shared-assets MCP server. Owns validate_session_tool, get_session_status_tool, and
  get_batch_session_status_overview_tool. Use when inspecting a session, validating its file
  inventory against session_type, or auditing lifecycle progress across every session under
  the data root.
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

### Two independent "not-healthy" signals: `nk.bin` (uninitialized) vs. descriptor `incomplete`

A session can fail to be a clean acquisition in two completely different ways. The slsa status
tools surface both as independent flags; do not conflate them.

- **`nk.bin` marker (uninitialized).** A zero-byte `nk.bin` file inside `raw_data/` is written
  by `SessionData.create()` at session creation and removed by `mark_runtime_initialized()`
  once the acquisition runtime has finished creating its per-session snapshots and
  initializing instruments. While this marker is present the session holds **no data of
  value** — it is a trash target and is safe to purge. Its absence means the session made it
  past initialization, not that acquisition went cleanly.
- **Descriptor `incomplete` field.** Every descriptor dataclass (`LickTrainingDescriptor`,
  `RunTrainingDescriptor`, `MesoscopeExperimentDescriptor`, `WindowCheckingDescriptor`)
  carries a boolean `incomplete` field, default `True`, that the acquisition runtime flips to
  `False` on a clean session end. A session with `incomplete=True` **has real data** — it ran
  past initialization — but something went wrong during the run and the data may have gaps.
  Static processing pipelines should skip it or handle it manually rather than delete it.

The two signals are reported as separate keys everywhere they surface: `uninitialized` (from
`nk.bin`) and `incomplete` (from the descriptor's field). `get_session_status_tool` and
`get_batch_session_status_overview_tool` check both; `validate_session_tool` reports both in
its summary; `get_project_overview_tool` keeps independent counts for each.

---

## Anatomy of a session directory

Every Sollertia session is a directory whose YAML files live **inside `raw_data/`** (not at the
session root):

```text
<session>/
├── raw_data/                                  # acquired data and frozen metadata (written by the acquisition runtime)
│   ├── session_data.yaml                      # SessionData marker (THIS SKILL)
│   ├── session_descriptor.yaml                # /session-descriptors (per-session-type dataclass, flat filename)
│   ├── surgery_metadata.yaml                  # /subject-metadata
│   ├── system_configuration.yaml              # frozen system config (owned by sollertia-experiment)
│   ├── experiment_configuration.yaml          # /experiment-configuration (frozen, experiment sessions only)
│   ├── hardware_state.yaml                    # /session-hardware-state
│   ├── zaber_positions.yaml                   # experiment plugin /session-snapshots
│   ├── mesoscope_positions.yaml               # experiment plugin /session-snapshots
│   ├── ax_checksum.txt                        # raw_data integrity checksum (/managing-session-data)
│   ├── checksum_processing_tracker.yaml       # checksum resolution tracker (/managing-session-data)
│   ├── nk.bin                                 # uninitialized-session marker (present until the runtime finishes session init; absence ≠ clean acquisition — see the descriptor's `incomplete` field for that)
│   └── ... acquired data files ...
└── processed_data/                            # populated by experiment plugin /managing-session-data
```

### Path-resolution properties on `SessionData`

Python code that needs a per-session file path should read it directly from the `SessionData`
instance rather than concatenating filenames by hand. The shared-assets library packages every
canonical session filename and directory into three enums (`RawDataFiles`, `Directories`,
`ProcessingTrackers`) and exposes each as a property on `SessionData`:

- Raw-data files: `session_data_path`, `session_descriptor_path`, `surgery_metadata_path`,
  `hardware_state_path`, `experiment_configuration_path`, `system_configuration_path`,
  `checksum_path`, `checksum_tracker_path`.
- Raw-data Mesoscope-VR snapshots (authoring owned by the experiment plugin's
  `/session-snapshots`, but the `Path` properties exist on `SessionData` for read access):
  `zaber_positions_path`, `mesoscope_positions_path`, `window_screenshot_path`.
- Raw-data subdirectories: `raw_camera_data_path`, `raw_behavior_data_path`,
  `raw_microcontroller_data_path`, `raw_mesoscope_data_path`.
- Processed-data subdirectories: `behavior_data_path`, `cindra_data_path`, `camera_timestamps_path`,
  `camera_data_path`, `microcontroller_data_path`.
- Processing trackers: `behavior_tracker_path`, `camera_tracker_path`, `video_tracker_path`,
  `microcontroller_tracker_path`, `cindra_single_recording_tracker_path`.
- Cindra layout: `cindra_multi_recording_path`.

All properties return a `Path` unconditionally; callers check existence with `.exists()` when the
path is conditional (experiment-only files, not-yet-produced outputs, forward-looking pipelines).

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

A Sollertia session moves through a small set of lifecycle states. This skill exposes three
tools that report where each session sits. All three distinguish the two orthogonal
"not-healthy" signals described above (`nk.bin` ≠ descriptor `incomplete`) — treat them as
independent flags, not two names for the same thing.

### Status values

`get_session_status_tool` and `get_batch_session_status_overview_tool` collapse the flag
combination into a single `status` enum with the following precedence (highest wins):

| `status`        | Meaning                                                                                                       |
|-----------------|---------------------------------------------------------------------------------------------------------------|
| `uninitialized` | `nk.bin` present. Session never finished runtime init; no data of value. Safe to purge.                       |
| `error`         | `nk.bin` absent but the descriptor YAML cannot be loaded (missing, malformed, wrong schema). State unknown.   |
| `incomplete`    | Descriptor loaded and its `incomplete` field is True. Session ran but had runtime issues; data may have gaps. |
| `processed`     | Clean session (descriptor `incomplete=False`) with at least one file under `processed_data/`.                 |
| `acquired`      | Clean session (descriptor `incomplete=False`) with no `processed_data/` contents yet.                         |

In addition to `status`, both tools return the independent boolean flags `uninitialized`,
`incomplete` (nullable — `None` when the descriptor cannot be loaded), and `has_processed_data`
so callers can compose their own logic.

### Validating a single session's file inventory

```text
validate_session_tool(session_path="<absolute>")
```

Reads the session's `SessionData` marker, determines the expected file inventory from its
`session_type`, and reports any missing files. The current checks are:

- The descriptor file (canonical `session_descriptor.yaml` under `raw_data/`).
- The frozen `experiment_configuration.yaml` (only required when `session_type` is
  `mesoscope experiment`).
- The frozen `system_configuration.yaml`.

Returns `valid` (True / False), an `issues` list, a `summary` (`session_name`, `project`,
`animal`, `session_type`, `uninitialized` flag from `nk.bin`, `incomplete` flag from the
descriptor — `None` when the descriptor cannot be loaded), and the resolved `session_path`.
The check does **not** verify hardware-state, position snapshots, or the contents of
`raw_data/` beyond these files. Use this before handing off to the experiment plugin's
`/managing-session-data` for preprocessing.

### Per-session lifecycle status

```text
get_session_status_tool(session_path="<absolute>")
```

Inspects three signals to derive the session's status: the `nk.bin` marker (uninitialized),
the descriptor's `incomplete` field, and the `processed_data/` population. Returns:

- `status`: one of `uninitialized`, `error`, `incomplete`, `processed`, `acquired` (see the
  precedence table above).
- `uninitialized`: boolean — `nk.bin` present.
- `incomplete`: boolean or `None` — the descriptor's field, or `None` when the descriptor
  could not be loaded (in which case `status` is `"error"`).
- `has_processed_data`: boolean.
- `session_path`: resolved session root.
- `error_detail`: present only when `status` is `"error"`; describes the descriptor load
  failure.

The tool does **not** report transferred or dataset-member states, does **not** include
transition timestamps, and does **not** enumerate the files that triggered the inference.

### Batch status overview

```text
get_batch_session_status_overview_tool(root_directory="<absolute path to data root>")
```

Walks every session under the supplied data root and applies the same classification per
session. The `root_directory` argument is required (slsa no longer auto-resolves a data root
after the system configuration moved to `sollertia-experiment`). Returns:

- `counts`: a dict with keys `uninitialized`, `incomplete`, `acquired`, `processed`, and
  `error` — per-status totals.
- `sessions`: per-session entries each carrying `session_name`, `project`, `animal`,
  `session_type`, `session_path`, `status`, `uninitialized`, `incomplete`,
  `has_processed_data`.
- `total_sessions`, `root_directory`.

It is a read-only aggregation.

Typical workflow:

1. Call `get_batch_session_status_overview_tool(root_directory=…)` for the summary.
2. For any session in an unexpected state, drill in with
   `get_session_status_tool(session_path=…)`.
3. For sessions reported as `uninitialized`, coordinate purging via the experiment plugin's
   `/managing-session-data` — these have no data of value.
4. For sessions reported as `incomplete` or `error`, call `validate_session_tool(session_path=…)`
   to identify missing files, then hand off to `/session-descriptors`, `/session-hardware-state`,
   the experiment plugin's `/session-snapshots`, or `/managing-session-data` to remediate.

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
