---
name: session-data
description: >-
  Reads, writes, and validates SessionData markers and produces per-session health and inventory reports via the
  sollertia-shared-assets MCP server. Owns inspect_sessions_tool and the file-path based read / write / describe trio
  for session_data.yaml. Use when inspecting one or more sessions, auditing lifecycle status, or repairing a corrupted
  SessionData marker.
user-invocable: false
---

# Sollertia session data

Reads, writes, and validates the canonical `SessionData` marker file that lives inside every Sollertia session
directory, exposes the `SessionTypes` and `ProcessingTrackers` enum surfaces, and produces detailed per-session health
and inventory reports. Uses the `slsa mcp` MCP server. This skill is the **exclusive** owner of `inspect_sessions_tool`
and the file-path based `read_session_data_tool`, `write_session_data_tool`, and `describe_session_data_schema_tool`
trio, and no other skill in the marketplace may call these. It also owns the read pattern for
`list_supported_session_types_tool` and `list_processing_trackers_tool`, which other skills defer to here.

The inventory side of "which descriptors and assets exist for a session" is part of `inspect_sessions_tool`'s report,
and no separate discover-descriptors tool exists.

---

## Scope

**Covers:**
- Reading the `SessionData` marker file
- Per-session health and inventory reports for one or many sessions (`inspect_sessions_tool`)
- Listing the canonical `SessionTypes` enum values
- Listing the canonical `ProcessingTrackers` filenames (`list_processing_trackers_tool`)
- The structural anatomy of a Sollertia session directory

**Does not cover:**
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing the per-session hardware-state snapshot (see `/session-hardware-state`)
- Reading or writing the per-session Zaber and mesoscope-objective position snapshots (see
  `mesoscope:mesoscope-vr-snapshots`)
- Reading the frozen system configuration captured at session start (see `mesoscope:mesoscope-vr`, which owns
  `read_session_system_configuration_tool`)
- Reading the frozen VR task configuration captured at session start (see `/task-templates`, which owns
  `vr_configuration.yaml`)
- Reading the frozen experiment configuration captured at session start (see `/experiment-configuration` for
  `read_experiment_configuration_tool`)
- Reading subject metadata (see `/data-assets`)
- Discovering projects, animals, or sessions (see `/project-hierarchy`, which owns `get_data_root_overview_tool`)
- Datasets that aggregate sessions (see `/datasets`)
- Preprocessing, deleting, or migrating sessions (see the experiment plugin's `experiment:data-management`)

---

## What is a session

A **session** is a single, contiguous data acquisition run performed on one animal, under one project, with one
acquisition system. Sessions are the atomic unit of acquired data, so every descriptor, snapshot, dataset, and
processing pipeline output is keyed off one or more sessions. A session bundles the canonical `SessionData` marker, the
per-session-type descriptor, the frozen system and experiment configurations, hardware-state and position snapshots, and
the raw acquired data itself.

The acquisition runtime is the only authorized creator of a session. The only write path this skill owns is
`write_session_data_tool`.

### How sessions are named

A session's name is a **UTC timestamp accurate to microseconds**, of the shape `YYYY-MM-DD-HH-MM-SS-ffffff` (for example
`2026-04-19-15-04-23-789012`). Microsecond precision makes name collisions between sessions started on the same machine
impossible, and lexicographic sort order coincides with chronological order, so sorting an animal directory's children
yields sessions in the order they were acquired.

The full session path is `<root>/<project>/<animal>/<session_name>/`. Sessions are created only on the acquisition
machine, and storage and processing destinations only ever see sessions that the acquisition runtime has already
produced and pushed downstream.

### Why the structure splits `raw_data/` from `processed_data/`

Every session's contents are partitioned into two top-level subdirectories with very different lifecycles:

- **`raw_data/`** is the **immutable record** of what was acquired. It holds the original sensor data, the `SessionData`
  marker, the frozen configuration snapshots, the hardware-state snapshot, and the per-session-type descriptor. Once
  acquisition is complete the contents of `raw_data/` are not supposed to change.
- **`processed_data/`** is the **accreted** record of derived outputs. It does not exist at session creation and only
  appears when downstream processing pipelines start writing into it. Each pipeline owns its own subdirectory under
  `processed_data/`. Because the contents are always derivable from `raw_data/` plus configuration, `processed_data/`
  can be deleted and regenerated at any time without losing scientific data.

The absence of `processed_data/` on a freshly acquired session is therefore expected, not an error. For the catalog of
which pipelines write where, defer to the skills that own the output paths and tracker conventions: the forging plugin's
processing skills, `cindra:single-recording-processing`, `cindra:multi-recording-processing`,
`communication:log-processing`, and `video:log-processing`. The `cindra:`, `communication:`, and `video:` entries
resolve through the cindra and ataraxis marketplaces.

### Two independent "not-healthy" signals: `nk.bin` (uninitialized) against descriptor `incomplete`

A session can fail to be a clean acquisition in two completely different ways. `inspect_sessions_tool` surfaces both as
independent flags, so do not conflate them.

- **`nk.bin` marker (uninitialized).** A zero-byte `nk.bin` file inside `raw_data/` is written by `SessionData.create()`
  at session creation and removed by `mark_runtime_initialized()` once the acquisition runtime has finished creating its
  per-session snapshots and initializing instruments. While this marker is present the session holds **no data of
  value**, so it is a trash target and is safe to purge. Its absence means the session made it past initialization,
  rather than that acquisition went cleanly.
- **Descriptor `incomplete` field.** Every descriptor dataclass declares `incomplete: bool = True`, which the
  acquisition runtime flips to `False` on a clean session end (see `/session-descriptors` for the per-session-type
  descriptor schemas). A session with `incomplete=True` **has real data**, because it ran past initialization, yet
  something went wrong during the run and the data may have gaps. Static processing pipelines should skip it or handle
  it manually rather than delete it.

Both signals and the derived lifecycle `status` are returned by `inspect_sessions_tool` and by
`get_data_root_overview_tool` (owned by `/project-hierarchy`, which uses the same two-signal model for its per-session
and per-project rollups).

---

## Anatomy of a session directory

Every Sollertia session is a directory whose YAML files live **inside `raw_data/`** rather than at the session root:

```text
<session>/
├── raw_data/                                  # acquired data and frozen metadata (written by the acquisition runtime)
│   ├── session_data.yaml                      # SessionData marker (THIS SKILL)
│   ├── session_descriptor.yaml                # /session-descriptors (per-session-type dataclass, flat filename)
│   ├── surgery_metadata.yaml                  # /data-assets (written by preprocessing, not acquisition)
│   ├── system_configuration.yaml              # frozen system config (owned by sollertia-experiment)
│   ├── experiment_configuration.yaml          # /experiment-configuration (frozen, experiment sessions only)
│   ├── vr_configuration.yaml                  # /task-templates frozen snapshot (corridor-task sessions only)
│   ├── hardware_state.yaml                    # /session-hardware-state
│   ├── <system-specific raw assets>           # dispatched via SYSTEM_RAW_DATA_REGISTRY, Mesoscope-VR layout -> mesoscope:mesoscope-vr-snapshots
│   ├── ax_checksum.txt                        # raw_data integrity checksum (experiment:data-management)
│   ├── checksum_processing_tracker.yaml       # checksum resolution tracker (forging:processing-results, written by sollertia-forgery)
│   ├── nk.bin                                 # uninitialized-session marker (see note below)
│   ├── behavior_data/                         # DataLogger NPZ archives (raw microcontroller and runtime messages)
│   └── camera_data/                           # raw camera recordings
└── processed_data/                            # populated by downstream processing pipelines
```

`surgery_metadata.yaml` is the one exception to the `raw_data/` comment above. Acquisition-system preprocessing writes
it after the session ends, so it is absent until preprocessing runs (see `/data-assets`).

`nk.bin` and the descriptor's `incomplete` field are the two independent signals described above.

### The marker's persisted keys

`session_data.yaml` persists eight keys, in declaration order: `project_name`, `animal_id`, `session_name`,
`session_type`, `acquisition_system`, `experiment_name`, `python_version`, and `sollertia_experiment_version`. The
`raw_data_path` and `processed_data_path` fields carry `YAML_EXCLUDE_METADATA`, so they are absent from the file and
from the `data` payload `read_session_data_tool` returns, and `load()` re-derives both from the marker's own on-disk
location. That makes `session_data.yaml` host-portable, so copying or migrating a session tree to another machine or
mount point needs no marker rewrite. The marker write is atomic, so an interrupted save leaves the previously written
marker intact.

### How `SessionData.load()` finds the marker

`load()` first tries the canonical fast path `<session>/raw_data/session_data.yaml` and uses it whenever that file
exists, which costs one metadata query instead of a recursive walk. For any other directory it falls back to a recursive
`discover_marker_files` scan of the whole tree. The scan raises `FileNotFoundError` when the tree holds zero markers or
more than one:

```text
Expected a single session_data.yaml file to be located under the directory tree specified by the input path: <path>. Instead, encountered <n> candidate files.
```

The fallback scan also raises `OSError` when it reaches a directory it cannot read.

Both session-level tools surface either exception verbatim as `error_detail: "Failed to load SessionData: ..."`, which
is what tells the two remediations apart. A nested or duplicated marker is fixed by pointing the call at the true
session root, while a genuinely corrupt marker is fixed by a `write_session_data_tool` repair.

### Path-resolution sub-dataclasses on `SessionData`

Python code that needs a per-session file path should read it from the `SessionData` instance's sub-dataclass attributes
rather than concatenating filenames by hand. The shared-assets library packages every system-agnostic session filename
and directory into three enums (`RawDataFiles`, `Directories`, `ProcessingTrackers`) and dispatches them onto two of the
three runtime-only sub-dataclasses populated by `SessionData._build_sub_dataclasses()` (called from both `create` and
`load`). The third sub-dataclass draws on the acquisition system's own filename and directory enums instead, dispatched
through `SYSTEM_RAW_DATA_REGISTRY`. Five of the seven `ProcessingTrackers` members are dispatched onto a session
sub-dataclass field. The two that are not are `ProcessingTrackers.FORGING`, which lives at the forged dataset root, and
`ProcessingTrackers.MANIFEST`, which lives at the project root:

- **`instance.raw_data` (`RawData`)** holds the system-agnostic raw assets: `session_data_path`,
  `session_descriptor_path`, `surgery_metadata_path`, `hardware_state_path`, `system_configuration_path`,
  `experiment_configuration_path`, `vr_configuration_path`, `checksum_path`, `checksum_tracker_path`, `nk_path`,
  `behavior_data_path`, `camera_data_path`. `vr_configuration_path` is populated only when the session runs the corridor
  task (check `.exists()` before reading). Microcontroller raw data is bundled into the DataLogger archives under
  `behavior_data_path`, so there is no separate raw microcontroller field.
- **`instance.processed_data` (`ProcessedData`)** declares eight system-agnostic processed assets: `runtime_data_path`,
  `runtime_tracker_path`, `video_data_path`, `video_tracker_path`, `microcontroller_data_path`,
  `microcontroller_tracker_path`, `cindra_data_path`, and `two_photon_tracker_path`. `video_data_path` covers both the
  per-frame camera timestamps extracted from the camera log archives and the re-packaged pose-estimation output, so a
  single field serves the whole video stage. The raw-side `behavior_data_path` on `RawData` and the processed-side
  `runtime_data_path` here are separate fields on separate sub-dataclasses. Cindra fields live here (not under a
  system-specific sub-dataclass) because cindra is reusable by any neural-imaging-data-generating acquisition system.
  Only cindra's single-recording stage is addressed here. Its multi-recording outputs are written once per dataset
  inside a dataset-named directory that no fixed per-session path reaches, so the library declares no field for them and
  sollertia-forgery resolves them through cindra's own `resolve_dataset_path`.
- **`instance.system_raw_data`** holds the acquisition-system-specific raw assets, dispatched from
  `SYSTEM_RAW_DATA_REGISTRY` keyed by `acquisition_system`. Each system registers its `<System>RawData` builder in
  `SYSTEM_RAW_DATA_REGISTRY`, so the fields exposed here vary by system. For the Mesoscope-VR raw-data layout, defer to
  `mesoscope:mesoscope-vr-snapshots`.

All fields return a `Path` unconditionally, and callers check existence with `.exists()` when the path is conditional,
covering experiment-only files, not-yet-produced outputs, and forward-looking pipelines. Instances constructed via
`SessionData.from_yaml` directly, without going through `load`, do not have the sub-dataclass attributes populated, and
access raises `AttributeError`. `inspect_sessions_tool` iterates `dataclasses.fields()` over each sub-dataclass
internally to produce the `raw_data_files` and `processed_data_subdirs` entries in its per-session report, so MCP
callers do not need to reconstruct the list by hand.

Each entry in the report's `raw_data_files` / `processed_data_subdirs` lists carries:

- `field` is the sub-dataclass field name, such as `session_descriptor_path`.
- `path` is the absolute resolved path.
- `scope` is `"generic"` for fields read off `raw_data` or `processed_data`, and `"system"` for fields read off
  `system_raw_data`, such as Mesoscope-VR snapshots.
- `kind` is `"file"` when the path has a non-empty suffix, and `"directory"` otherwise.
- `exists` reports whether the path is currently present on disk.

`get_data_root_overview_tool` (owned by `/project-hierarchy`) walks the data root looking for `session_data.yaml`
markers and returns seven top-level keys: `projects`, `sessions`, `counts`, `total_projects`, `total_animals`,
`total_sessions`, and `root_directory`. Each entry of the flat `sessions` list carries a singular `session_path`. The
plural `session_paths` exists only nested inside `projects[].animals[]`. All session-path arguments to this skill's
tools accept **either the session root directory or its `raw_data/` subdirectory**, and the resolver normalizes both
forms before any read.

---

## Session types

The canonical `SessionTypes` enum values are `lick training`, `run training`, `window checking`, and
`mesoscope experiment`. Use `list_supported_session_types_tool` for the authoritative list. It returns `value`, `name`,
and `descriptor_class` for each type. The descriptor filename is always `session_descriptor.yaml` for every session
type, so the tool omits it. Descriptor schemas are owned by `/session-descriptors`.

Session types are paired with acquisition systems by `SYSTEM_SESSION_TYPES`. Each acquisition system declares the
session types it can run, and `SessionData.create()` rejects a session-type and acquisition-system pairing that is not
declared. Pass the system as `list_supported_session_types_tool(acquisition_system=...)` whenever you already know the
current acquisition system, such as on a configured host or after reading a session's `acquisition_system`. The result
then reflects what that system can actually run. Omit the argument only when you genuinely need the platform-wide list.
`list_session_type_support_tool` returns the full system-to-session-type map in one call.

---

## MCP tool surface

| Tool                                | Purpose                                                                                                      |
|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| `inspect_sessions_tool`             | Produces a detailed health and inventory report for one or more sessions (exclusive)                         |
| `read_session_data_tool`            | Reads a `session_data.yaml` file via the `SessionData` schema (file-path based, exclusive)                   |
| `write_session_data_tool`           | Creates or replaces a `session_data.yaml` file, validated against `SessionData` (file-path based, exclusive) |
| `describe_session_data_schema_tool` | Returns the `SessionData` dataclass schema (exclusive)                                                       |
| `list_supported_session_types_tool` | Returns the supported `SessionTypes`, optionally scoped to one acquisition system                            |
| `list_session_type_support_tool`    | Returns the full map of each acquisition system to the session types it can run                              |
| `list_processing_trackers_tool`     | Enumerates every `ProcessingTrackers` filename used across the platform (`name`, `filename`, `description`)  |

`inspect_sessions_tool` accepts `session_paths: list[str]`, so pass a single-element list for one session, or many paths
to inspect a batch. There is no separate single or batch signature. The tool returns a flat `sessions` list of
per-session reports, a top-level `total_sessions` count that includes error reports, and a top-level `counts` tally of
lifecycle statuses across the batch. `counts` always carries all five status keys (`uninitialized`, `incomplete`,
`acquired`, `processed`, `error`) zero-filled, so a missing key is never meaningful.

Each per-session report opens with an `identity` block, holding `project`, `animal`, `session_name`, `session_type`,
`acquisition_system`, and `experiment_name` read off the loaded `SessionData`, so callers can key off the session's
identity without a separate `read_session_data_tool` call. A `status="error"` report comes in two shapes, because the
status covers two different failures. A path-resolution or `SessionData.load()` failure aborts before the report is
built, so that entry carries only `session_path`, `status`, and `error_detail`. A descriptor-read failure happens after
the marker already loaded, so that entry is a complete report, `identity` and both inventory lists included, with
`error_detail` added alongside them. Callers MUST therefore gate on the presence of the `identity` key rather than on
`status != "error"`. Gating on the status discards the identity of exactly the sessions whose descriptors need
repairing, and the sibling skills that read `identity.session_type` and `identity.acquisition_system` gate the same way
(see `/session-descriptors`, `/session-hardware-state`).

The read, write, and describe trio for `session_data.yaml` is **file-path based**, symmetric with the equivalent trios
for descriptors, hardware state, and surgery metadata. The caller supplies the absolute `file_path`, and these tools do
not resolve session roots or compute lifecycle status. **Reading the marker file directly is expected to follow a
discovery or health check.** Call `/project-hierarchy`'s `get_data_root_overview_tool` (whole-root walk) or this skill's
`inspect_sessions_tool` (per-session report) first to determine whether the session holds valid data and its lifecycle
status. Then reach for `read_session_data_tool` only when the raw YAML payload (including `python_version` /
`sollertia_experiment_version` compatibility fields) is what you actually need.

`list_supported_session_types_tool` and `list_processing_trackers_tool` are owned by this skill in the sense that this
is where the read pattern is documented and where other skills should hand off when they need them. Both are read-only
and may also be called as natural shares. `list_processing_trackers_tool` is the canonical reference for the seven
`ProcessingTrackers` members, covering the checksum, runtime, microcontroller, video, two-photon, forging, and manifest
pipelines. Five of the seven resolve to a field on a session sub-dataclass, so neither `ProcessingTrackers.FORGING` nor
`ProcessingTrackers.MANIFEST` appears in a session's `raw_data_files` or `processed_data_subdirs` inventory (see
"Path-resolution sub-dataclasses on `SessionData`"). Cindra's multi-recording tracker has no member here.

### Repairing a marker with `write_session_data_tool`

`write_session_data_tool(file_path, session_data_payload, *, overwrite=True)` exists for agent-driven repair of a
corrupted or partially missing marker. The **primary** on-disk copy is always authored by the acquisition runtime via
`SessionData.create` at session start, so this tool sits outside the normal acquisition flow.

`overwrite` is keyword-only and defaults to `True`, deliberately inverting the `False` default of the underlying write
helper, so a repair call replaces the existing marker by default. Passing `overwrite=False` makes the call fail rather
than clobber, returning:

```text
Unable to write SessionData to <path>: a file already exists at this path. Pass overwrite=True to replace it.
```

What the write actually validates, and why every amendment MUST be a read-mutate-write of the complete record, is
documented in the `## Response contract` section of `/assets-mcp-environment-setup`. On top of that generic contract,
`SessionData.__post_init__` converts `session_type` and `acquisition_system` unconditionally and raises when either
value falls outside the platform vocabulary, surfacing as `Unable to validate the payload as SessionData: ...`. Source
the valid vocabulary from `list_supported_session_types_tool` and `list_supported_acquisition_systems_tool` rather than
from memory.

A repair call carries the complete eight-field payload:

```text
write_session_data_tool(
    file_path="<absolute session root>/raw_data/session_data.yaml",
    session_data_payload={
        "project_name": "example_project",
        "animal_id": "666",
        "session_name": "2026-04-19-15-04-23-789012",
        "session_type": "mesoscope experiment",
        "acquisition_system": "mesoscope",
        "experiment_name": "example_experiment",
        "python_version": "3.14.6",
        "sollertia_experiment_version": "5.0.0",
    },
)
```

---

## Session lifecycle status

### Status values

`inspect_sessions_tool` (and `get_data_root_overview_tool`) collapse the flag combination into a single `status` enum
with the following precedence (highest wins):

| `status`        | Meaning                                                                                                                                                                                                                   |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `uninitialized` | `nk.bin` present. Session never finished runtime init, so it holds no data of value. Safe to purge.                                                                                                                       |
| `error`         | `nk.bin` absent, and either the descriptor YAML cannot be loaded (missing, malformed, wrong schema) or `SessionData.load()` failed on the marker itself (see "How `SessionData.load()` finds the marker"). State unknown. |
| `incomplete`    | Descriptor loaded and its `incomplete` field is True. Session ran but had runtime issues, so data may have gaps.                                                                                                          |
| `processed`     | Clean session (descriptor `incomplete=False`) with a `processed_data/` directory that exists and holds at least one entry.                                                                                                |
| `acquired`      | Clean session (descriptor `incomplete=False`) with no `processed_data/` directory, or an empty one.                                                                                                                       |

In addition to `status`, each per-session report returns the independent boolean flags `uninitialized`, `incomplete`,
and `has_processed_data`, so callers can compose their own logic. `incomplete` is nullable, holding `None` both when the
session is uninitialized and when the descriptor cannot be loaded. An uninitialized session never has its descriptor
read at all, so a `None` there reports the absence of a read rather than a stale flag.

---

## Workflows

### Inspecting a specific session

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`).
2. **Locate the session.** Hand off to `/project-hierarchy` to call `get_data_root_overview_tool`, then pick the session
   path from its output, or to `/session-discovery` when date-range / include-exclude filtering is needed.
3. **Determine health and lifecycle state first:**
   ```text
   inspect_sessions_tool(session_paths=["<absolute session root>"])
   ```
   The report's `status` plus `raw_data_files` inventory tells you whether the session holds valid data and which
   canonical assets are present. If `status` is `uninitialized`, or `error` with no `identity` block, the marker content
   is not meaningful, so surface that to the user instead of reading. An `error` report that does carry `identity` means
   the marker loaded and the descriptor did not, which is a `/session-descriptors` repair rather than a marker one.
4. **Read the raw marker YAML** when a caller needs fields that `inspect_sessions_tool` does not project (notably
   `python_version` and `sollertia_experiment_version`):
   ```text
   read_session_data_tool(file_path="<absolute>/raw_data/session_data.yaml")
   ```
5. **Hand off to `/session-descriptors`** to read the descriptor contents.
6. **Hand off to `/session-hardware-state`** to read the per-session hardware-state snapshot.
7. **Hand off to `mesoscope:mesoscope-vr-snapshots`** to read the Zaber and mesoscope-objective position snapshots.
8. **Hand off to `/experiment-configuration`** for the frozen experiment configuration via
   `read_experiment_configuration_tool` (pass the session snapshot path).
9. **Hand off to `mesoscope:mesoscope-vr`** for the frozen `system_configuration.yaml` snapshot, read with
   `read_session_system_configuration_tool`. That tool is per-system and lives on the acquisition system's own `sle mcp`
   surface, not on `slsa mcp`.

### Validating a session's file inventory

1. **Inventory the session:**
   ```text
   inspect_sessions_tool(session_paths=["<absolute>"])
   ```
   The per-session report's `required_assets` list enumerates every file the session requires, each with a `present`
   flag. The descriptor and the system configuration snapshot are always required. The experiment configuration snapshot
   is required when the session carries an `experiment_name`, and the VR configuration snapshot is required when the
   session type runs the corridor task. The `issues` list restates missing required files as human-readable strings. Use
   this before handing off to `experiment:data-management` for preprocessing.
2. **Map the processed side onto the pipelines that produced it:**
   ```text
   list_processing_trackers_tool()
   ```
   Match each `processed_data_subdirs` entry from step 1 against the tracker filename its owning pipeline writes there,
   which is how a `processed_data/` subdirectory resolves back to the stage responsible for it. Neither
   `ProcessingTrackers.FORGING` nor `ProcessingTrackers.MANIFEST` appears in the report (see "Path-resolution
   sub-dataclasses on `SessionData`").

### Batch lifecycle audit across a data root

1. **Fetch the tree and top-level counts:** call `get_data_root_overview_tool(root_directory=…)` via
   `/project-hierarchy`. The response's `counts` is a root-wide status tally and each `projects[*].counts` is a
   per-project tally.
2. **Drill into a specific set of sessions** with `inspect_sessions_tool(session_paths=[…])`, built by collecting the
   singular `session_path` from each entry of the overview's flat `sessions` list, or by reading
   `projects[*].animals[*].session_paths`, or from any filtered subset produced by `/session-discovery`. The batch
   report gives you the same status plus full per-session inventory in one call.
3. For sessions reported as `uninitialized`, coordinate purging via the experiment plugin's
   `experiment:data-management`, because these have no data of value.
4. For sessions reported as `incomplete` or `error`, read the per-session `issues` list when the report carries one,
   and hand off to `/session-descriptors`, `/session-hardware-state`, the mesoscope plugin's
   `mesoscope:mesoscope-vr-snapshots`, or `experiment:data-management` to remediate.

### Querying supported session types

```text
# Scope to the system you are operating within (preferred on a configured host):
list_supported_session_types_tool(acquisition_system="mesoscope")
# Platform-wide list:
list_supported_session_types_tool()
# Full system-to-session-type map:
list_session_type_support_tool()
```

Use the system-scoped form to validate a session-type string against your current acquisition system before handing off
to another tool (e.g., to `/session-descriptors` to read a descriptor). Default to the scoped form whenever you know the
acquisition system, because the unscoped form returns every platform session type regardless of which system can run it.

---

## Related skills

| Skill                              | Relationship                                                                                                                                                                                                                           |
|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/cli-reference`                   | Reference: the `slsa` commands available while the MCP server is down                                                                                                                                                                  |
| `/assets-mcp-environment-setup`    | Run first if the MCP server is not connected                                                                                                                                                                                           |
| `/working-directory`               | Bootstraps the host path records. This skill's tools take absolute paths and read none of them                                                                                                                                         |
| `/project-hierarchy`               | Owns `get_data_root_overview_tool` for root-wide discovery                                                                                                                                                                             |
| `/session-discovery`               | Filters the flat `sessions` list from `get_data_root_overview_tool`                                                                                                                                                                    |
| `/session-descriptors`             | Sibling that owns the per-session descriptor read, write, and schema tools                                                                                                                                                             |
| `/session-hardware-state`          | Sibling that owns the per-session hardware-state snapshot                                                                                                                                                                              |
| `mesoscope:mesoscope-vr-snapshots` | Owns the frozen Zaber and mesoscope-objective position snapshots                                                                                                                                                                       |
| `/data-assets`                     | Sibling that owns read assets, such as animal-scoped surgery records                                                                                                                                                                   |
| `mesoscope:mesoscope-vr`           | Owns `read_session_system_configuration_tool` for the frozen `system_configuration.yaml` snapshot                                                                                                                                      |
| `/experiment-configuration`        | Owns `read_experiment_configuration_tool` (reads both project source and frozen session snapshot)                                                                                                                                      |
| `/task-templates`                  | Owns `vr_configuration.yaml`, the frozen VR task snapshot captured at session start                                                                                                                                                    |
| `/library-extension`               | Cross-cutting recipes to add a `SessionTypes` or `AcquisitionSystems` member, a read asset, a raw-tree directory, or the session-record surfaces of a new processing pipeline. Lists the skill content that needs updating in lockstep |
| `/datasets`                        | Datasets aggregate sessions                                                                                                                                                                                                            |
| `experiment:data-management`       | Preprocesses, migrates, and deletes sessions. Project directories must already exist (created via `create_project_tool`) before sessions can be created                                                                                |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] inspect_sessions_tool (or get_data_root_overview_tool) was called before reading the marker
      file directly, so the session's lifecycle status is known first
- [ ] read_session_data_tool was only called when the raw payload fields
      (python_version / sollertia_experiment_version) were actually needed
- [ ] inspect_sessions_tool was called before handing off to the experiment plugin's
      experiment:data-management for preprocessing (issues list is empty for required_assets)
- [ ] write_session_data_tool was only invoked for explicit repair workflows, rather than during
      normal acquisition, which is the acquisition runtime's responsibility
- [ ] list_processing_trackers_tool was used to map processed_data/ subdirectories onto the
      pipelines that own them
- [ ] Handed off to /session-descriptors, /session-hardware-state, /data-assets,
      /experiment-configuration, or mesoscope:mesoscope-vr-snapshots for any read that
      goes deeper than the marker
```
