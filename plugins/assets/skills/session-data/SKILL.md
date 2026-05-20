---
name: session-data
description: >-
  Reads, writes, and validates SessionData markers and produces per-session health and
  inventory reports via the sollertia-shared-assets MCP server. Owns inspect_sessions_tool
  and the file-path based read / write / describe trio for session_data.yaml. Use when
  inspecting one or more sessions, auditing lifecycle status, or repairing a corrupted
  SessionData marker.
user-invocable: true
---

# Sollertia session data

Reads, writes, and validates the canonical `SessionData` marker file that lives inside every
Sollertia session directory, exposes the `SessionTypes` enum surface, and produces detailed
per-session health and inventory reports. Uses the `slsa mcp` MCP server. This skill is the
**exclusive** owner of `inspect_sessions_tool` and the file-path based `read_session_data_tool` /
`write_session_data_tool` / `describe_session_data_schema_tool` trio — no other skill in the
marketplace may call these.

The **primary** on-disk `session_data.yaml` copy is authored by the acquisition runtime via
`SessionData.create` at session start. `write_session_data_tool` exists for agent-driven
**repair** of a corrupted or partially missing marker; it is not part of the normal acquisition
flow. The inventory side of "which descriptors and assets exist for a session" is part of
`inspect_sessions_tool`'s report — there is no separate discover-descriptors tool.

---

## Scope

**Covers:**
- Reading the `SessionData` marker file
- Per-session health and inventory reports for one or many sessions (`inspect_sessions_tool`)
- Listing the canonical `SessionTypes` enum values
- The structural anatomy of a Sollertia session directory

**Does not cover:**
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing the per-session `MesoscopeHardwareState` snapshot (see
  `/session-hardware-state`)
- Reading or writing the per-session Zaber and mesoscope-objective position snapshots (see the
  experiment plugin's `/session-snapshots`)
- Reading the frozen system configuration captured at session start (see the experiment plugin's
  `/system-configuration`)
- Reading the frozen experiment configuration captured at session start (see
  `/experiment-configuration` for `read_experiment_configuration_tool`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering projects, animals, or sessions (see `/project-hierarchy`, which owns
  `get_data_root_overview_tool`)
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

A session can fail to be a clean acquisition in two completely different ways. `inspect_sessions_tool`
surfaces both as independent flags; do not conflate them.

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

Both signals and the derived lifecycle `status` are returned by `inspect_sessions_tool` and by
`get_data_root_overview_tool` (owned by `/project-hierarchy`, which uses the same two-signal
model for its per-session and per-project rollups).

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
│   ├── vr_configuration.yaml                  # /task-templates frozen snapshot (when the session runs a Unity VR task)
│   ├── hardware_state.yaml                    # /session-hardware-state
│   ├── zaber_positions.yaml                   # experiment plugin /session-snapshots
│   ├── mesoscope_positions.yaml               # experiment plugin /session-snapshots
│   ├── window_screenshot.png                  # experiment plugin /session-snapshots (Mesoscope-VR)
│   ├── ax_checksum.txt                        # raw_data integrity checksum (/managing-session-data)
│   ├── checksum_processing_tracker.yaml       # checksum resolution tracker (/managing-session-data)
│   ├── nk.bin                                 # uninitialized-session marker (see note below)
│   └── ... acquired data files ...
└── processed_data/                            # populated by downstream processing pipelines
```

`nk.bin` is present until the runtime finishes session init. Its absence is **not** a "clean
acquisition" signal — see the descriptor's `incomplete` field for that.

### Path-resolution sub-dataclasses on `SessionData`

Python code that needs a per-session file path should read it from the `SessionData` instance's
sub-dataclass attributes rather than concatenating filenames by hand. The shared-assets library
packages every canonical session filename and directory into three enums (`RawDataFiles`,
`Directories`, `ProcessingTrackers`) and dispatches them onto three runtime-only sub-dataclasses
populated by `SessionData._build_sub_dataclasses()` (called from both `create` and `load`):

- **`instance.raw_data` (`RawData`)** — system-agnostic raw assets:
  `session_data_path`, `session_descriptor_path`, `surgery_metadata_path`, `hardware_state_path`,
  `system_configuration_path`, `experiment_configuration_path`, `vr_configuration_path`,
  `checksum_path`, `checksum_tracker_path`, `nk_path`, `behavior_data_path`, `camera_data_path`.
  `vr_configuration_path` is populated only when the session runs a Unity VR task (check
  `.exists()` before reading). Microcontroller raw data is bundled into the DataLogger archives
  under `behavior_data_path`, so there is no separate raw microcontroller field.
- **`instance.processed_data` (`ProcessedData`)** — system-agnostic processed assets:
  `behavior_data_path`, `behavior_tracker_path`, `camera_timestamps_path`, `camera_tracker_path`,
  `video_data_path`, `video_tracker_path`, `microcontroller_data_path`,
  `microcontroller_tracker_path`, `cindra_data_path`, `cindra_single_recording_tracker_path`,
  `cindra_multi_recording_path`. Cindra fields live here (not under a system-specific
  sub-dataclass) because cindra is reusable by any photometry-data-generating acquisition system.
- **`instance.system_raw_data`** — acquisition-system-specific raw assets, dispatched from
  `SYSTEM_RAW_DATA_REGISTRY` keyed by `acquisition_system`. For Mesoscope-VR, this is
  `MesoscopeRawData` with `zaber_positions_path`, `mesoscope_positions_path`,
  `window_screenshot_path`, and `mesoscope_data_path`. Future acquisition systems register their
  own `<System>RawData` builder in the same registry.

All fields return a `Path` unconditionally; callers check existence with `.exists()` when the
path is conditional (experiment-only files, not-yet-produced outputs, forward-looking pipelines).
Instances constructed via `SessionData.from_yaml` directly (without going through `load`) do not
have the sub-dataclass attributes populated; access raises `AttributeError`. `inspect_sessions_tool`
iterates `dataclasses.fields()` over each sub-dataclass internally to produce the
`raw_data_files` and `processed_data_subdirs` entries in its per-session report, so MCP callers
do not need to reconstruct the list by hand.

Each entry in the report's `raw_data_files` / `processed_data_subdirs` lists carries:

- `field` — the sub-dataclass field name (e.g. `session_descriptor_path`).
- `path` — the absolute resolved path.
- `scope` — `"generic"` for fields read off `raw_data` / `processed_data`; `"system"` for fields
  read off `system_raw_data` (e.g. Mesoscope-VR snapshots).
- `kind` — `"file"` when the path has a non-empty suffix; `"directory"` otherwise.
- `exists` — whether the path is currently present on disk.

`get_data_root_overview_tool` (owned by `/project-hierarchy`) walks the data root looking for
`session_data.yaml` markers and returns `session_paths` alongside per-session lifecycle status.
All session-path arguments to this skill's tool accept **either the session root directory or
its `raw_data/` subdirectory** — the resolver normalizes both forms before any read.

---

## Session types

The canonical `SessionTypes` enum values are `lick training`, `run training`, `window checking`,
and `mesoscope experiment`. Use `list_supported_session_types_tool` for the authoritative list
(it also returns each type's descriptor filename and dataclass). Per-type descriptor file
mapping and schemas are owned by `/session-descriptors`.

---

## MCP tool surface

| Tool                                 | Purpose                                                                                                          |
|--------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `inspect_sessions_tool`              | Produces a detailed health and inventory report for one or more sessions (exclusive)                             |
| `read_session_data_tool`             | Reads a `session_data.yaml` file via the `SessionData` schema (file-path based, exclusive)                       |
| `write_session_data_tool`            | Creates or replaces a `session_data.yaml` file, validated against `SessionData` (file-path based, exclusive)     |
| `describe_session_data_schema_tool`  | Returns the `SessionData` dataclass schema (exclusive)                                                           |
| `list_supported_session_types_tool`  | Returns the canonical `SessionTypes` enum strings                                                                |
| `list_processing_trackers_tool`      | Enumerates every `ProcessingTracker` filename used across the platform (`name`, `filename`, `description`)       |

`inspect_sessions_tool` accepts `session_paths: list[str]` — pass a single-element list for one
session, or many paths to inspect a batch. There is no separate single / batch signature. The
tool returns a flat `sessions` list of per-session reports plus a top-level `counts` tally of
lifecycle statuses across the batch.

The read / write / describe trio for `session_data.yaml` is **file-path based**, symmetric with
the equivalent trios for descriptors, hardware state, and surgery metadata. The caller supplies
the absolute `file_path`; these tools do not resolve session roots or compute lifecycle status.
**Reading the marker file directly is expected to follow a discovery or health check.** Call
`/project-hierarchy`'s `get_data_root_overview_tool` (whole-root walk) or this skill's
`inspect_sessions_tool` (per-session report) first to determine whether the session holds valid
data and its lifecycle status. Then reach for `read_session_data_tool` only when the raw
YAML payload (including `python_version` / `sollertia_experiment_version` compatibility fields)
is what you actually need.

`write_session_data_tool` exists for agent-driven repair of a corrupted or partially missing
marker. The **primary** on-disk copy is always authored by the acquisition runtime via
`SessionData.create` at session start; this tool should not be called during normal acquisition
flows.

`list_supported_session_types_tool` and `list_processing_trackers_tool` are owned by this skill
in the sense that this is where the read pattern is documented and where other skills should
hand off when they need them. Both are read-only and may also be called as natural shares.
`list_processing_trackers_tool` is the canonical reference for the filenames written by the
checksum, behavior, camera, video, microcontroller, cindra single- and multi-recording, forging,
analysis, manifest, and transfer pipelines.

---

## Session lifecycle status

A Sollertia session moves through a small set of lifecycle states. `inspect_sessions_tool`
distinguishes the two orthogonal "not-healthy" signals described above (`nk.bin` ≠ descriptor
`incomplete`) — treat them as independent flags, not two names for the same thing.

### Status values

`inspect_sessions_tool` (and `get_data_root_overview_tool`) collapse the flag combination into a
single `status` enum with the following precedence (highest wins):

| `status`        | Meaning                                                                                                       |
|-----------------|---------------------------------------------------------------------------------------------------------------|
| `uninitialized` | `nk.bin` present. Session never finished runtime init; no data of value. Safe to purge.                       |
| `error`         | `nk.bin` absent but the descriptor YAML cannot be loaded (missing, malformed, wrong schema). State unknown.   |
| `incomplete`    | Descriptor loaded and its `incomplete` field is True. Session ran but had runtime issues; data may have gaps. |
| `processed`     | Clean session (descriptor `incomplete=False`) with at least one file under `processed_data/`.                 |
| `acquired`      | Clean session (descriptor `incomplete=False`) with no `processed_data/` contents yet.                         |

In addition to `status`, each per-session report returns the independent boolean flags
`uninitialized`, `incomplete` (nullable — `None` when the descriptor cannot be loaded), and
`has_processed_data` so callers can compose their own logic.

---

## Workflows

### Inspecting a specific session

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`).
2. **Locate the session.** Hand off to `/project-hierarchy` to call `get_data_root_overview_tool`,
   then pick the session path from its output, or to `/session-discovery` when date-range /
   include-exclude filtering is needed.
3. **Determine health and lifecycle state first:**
   ```text
   inspect_sessions_tool(session_paths=["<absolute session root>"])
   ```
   The report's `status` plus `raw_data_files` inventory tells you whether the session holds
   valid data and which canonical assets are present. This step is the **prerequisite** for any
   read that touches the marker file directly — if `status` is `uninitialized` or `error`, the
   marker content is not meaningful, and you should surface that to the user instead of reading.
4. **Read the raw marker YAML** when a caller needs fields that `inspect_sessions_tool` does
   not project (notably `python_version` and `sollertia_experiment_version`):
   ```text
   read_session_data_tool(file_path="<absolute>/raw_data/session_data.yaml")
   ```
5. **Hand off to `/session-descriptors`** to read the descriptor contents.
6. **Hand off to `/session-hardware-state`** to read the frozen `MesoscopeHardwareState`.
7. **Hand off to the experiment plugin's `/session-snapshots`** to read the Zaber and
   mesoscope-objective position snapshots.
8. **Hand off to `/experiment-configuration`** for the frozen experiment configuration via
   `read_experiment_configuration_tool` (pass the session snapshot path).
9. **Hand off to the experiment plugin's `/system-configuration`** for the frozen
   `system_configuration.yaml` snapshot.

### Validating a session's file inventory

```text
inspect_sessions_tool(session_paths=["<absolute>"])
```

The per-session report's `required_assets` list enumerates every file the session's
`session_type` requires (the descriptor, the system configuration snapshot, and — for
`mesoscope experiment` only — the experiment configuration snapshot and the VR configuration
snapshot) with a `present` flag.
The `issues` list restates missing required files as human-readable strings. Use this before
handing off to the experiment plugin's `/managing-session-data` for preprocessing.

### Batch lifecycle audit across a data root

1. **Fetch the tree and top-level counts:** call `get_data_root_overview_tool(root_directory=…)`
   via `/project-hierarchy`. The response's `counts` is a root-wide status tally and each
   `projects[*].counts` is a per-project tally.
2. **Drill into a specific set of sessions** with `inspect_sessions_tool(session_paths=[…])`
   using the `session_paths` list from the overview (or any filtered subset produced by
   `/session-discovery`). The batch report gives you the same status plus full per-session
   inventory in one call.
3. For sessions reported as `uninitialized`, coordinate purging via the experiment plugin's
   `/managing-session-data` — these have no data of value.
4. For sessions reported as `incomplete` or `error`, read the per-session `issues` list and
   hand off to `/session-descriptors`, `/session-hardware-state`, the experiment plugin's
   `/session-snapshots`, or `/managing-session-data` to remediate.

### Querying supported session types

```text
list_supported_session_types_tool()
```

Use this when you need to validate a session-type string before using it in another tool call
(e.g., when handing off to `/session-descriptors` to read a descriptor).

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] inspect_sessions_tool (or get_data_root_overview_tool) was called before reading the marker
      file directly, so the session's lifecycle status is known first
- [ ] read_session_data_tool was only called when the raw payload fields
      (python_version / sollertia_experiment_version) were actually needed
- [ ] inspect_sessions_tool was called before handing off to the experiment plugin's
      /managing-session-data for preprocessing (issues list is empty for required_assets)
- [ ] write_session_data_tool was only invoked for explicit repair workflows — not during
      normal acquisition, which is the acquisition runtime's responsibility
- [ ] Handed off to /session-descriptors, /session-hardware-state, /subject-metadata,
      /experiment-configuration, or the experiment plugin's /session-snapshots for any read that
      goes deeper than the marker
```

---

## Related skills

| Skill                                      | Relationship                                                                                                                                 |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`            | Run first if the MCP server is not connected                                                                                                 |
| `/working-directory`                       | Required prerequisite — bootstraps the local working directory the agent uses to resolve project roots                                       |
| `/project-hierarchy`                       | Owns `get_data_root_overview_tool` for root-wide discovery                                                                                   |
| `/session-discovery`                       | Filters the flat `sessions` list from `get_data_root_overview_tool`                                                                          |
| `/session-descriptors`                     | Sibling — owns the per-session descriptor read/write/schema                                                                                  |
| `/session-hardware-state`                  | Sibling — owns the per-session `MesoscopeHardwareState` snapshot                                                                             |
| experiment plugin `/session-snapshots`     | Owns the frozen Zaber and mesoscope-objective position snapshots                                                                             |
| `/subject-metadata`                        | Sibling — owns animal-scoped subject records                                                                                                 |
| experiment plugin `/system-configuration`  | Authors the system configuration consumed at session start                                                                                   |
| `/experiment-configuration`                | Owns `read_experiment_configuration_tool` (reads both project source and frozen session snapshot)                                            |
| `/library-extension`                       | Cross-cutting recipe to add new `SessionTypes` or `AcquisitionSystems` members; lists the skill content here that needs updating in lockstep |
| forging plugin `/datasets`                 | Datasets aggregate sessions                                                                                                                  |
| experiment plugin `/managing-session-data` | Preprocesses, migrates, and deletes sessions. Project directories must already exist (created via `slsa configure project`) before sessions can be created |
