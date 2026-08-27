---
name: session-discovery
description: >-
  Discovers Sollertia sessions under a project root and filters by date range, animal, or session name via the
  sollertia-shared-assets MCP server. Chains get_data_root_overview_tool through filter_sessions_tool to produce
  `session_paths` lists consumed by downstream batch skills. Use when locating sessions ahead of any batch workflow or
  filtering a previously discovered list.
user-invocable: false
---

# Sollertia session discovery

Discovers and filters Sollertia sessions via the sollertia-shared-assets MCP tools. This skill is domain-agnostic,
providing the raw discover and filter surface from which any downstream batch skill can chain. For the artifacts each
processing pipeline requires on disk, see the forging plugin's `forging:processing-input-format`.

---

## Scope

**Covers:**
- Recursively discovering sessions under a data root via `session_data.yaml` markers, using
  `get_data_root_overview_tool`
- Post-discovery filtering by date range, animal inclusion-exclusion, and session inclusion-exclusion via
  `filter_sessions_tool`
- Producing confirmed `session_paths` lists for downstream batch skills

**Does not cover:**
- Walking the full project tree (projects, animals, experiments) as a primary workflow, see `/project-hierarchy`, which
  owns `get_data_root_overview_tool`
- Reading individual `SessionData` markers or full session health reports, see `/session-data`
- Reading or generating project manifest files, see the forging plugin's `forging:project-state`
- Checksum verification or regeneration, see the forging plugin's `forging:batch-processing`
- The artifacts each processing pipeline requires on disk, see the forging plugin's `forging:processing-input-format`
- MCP server connectivity issues, see `/assets-mcp-environment-setup`

---

## Agent requirements

You MUST use the sollertia-shared-assets MCP tools for session discovery. Do not scan the filesystem manually or import
`sollertia_shared_assets.discover_sessions` directly.

You MUST confirm the root directory path with the user before calling `get_data_root_overview_tool`.

---

## Available tools

### Session discovery

| Tool                          | Purpose                                                                                                                      |
|-------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `get_data_root_overview_tool` | Recursively discovers sessions by `session_data.yaml` markers and returns them as a flat list alongside a per-project rollup |

**Parameters:**

| Parameter        | Type  | Default    | Description                                                                   |
|------------------|-------|------------|-------------------------------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to the root directory, searched recursively                     |
| `strategy`       | `str` | `markers`  | `markers` or `directories`, both owned and documented by `/project-hierarchy` |

There is no server-side `project`, `animal_id`, or `session_types` narrowing. The tool scans the entire root and returns
the full hierarchy. Callers filter client-side for project, animal, and session type, or via `filter_sessions_tool` for
date range, session-name inclusion and exclusion, and animal inclusion and exclusion.

**Return structure (excerpt):**

```text
projects[]:              Per-project rollup, owned and documented by /project-hierarchy
sessions[]:              Flat per-session entries (drop-in input for filter_sessions_tool):
  session_name:          Session name (format: YYYY-MM-DD-HH-MM-SS-microseconds, 7 components)
  project:               Project name the session belongs to
  animal:                Animal identifier string
  session_type:          Session type string
  acquisition_system:    Acquisition system string
  experiment_name:       Experiment name (None for non-experiment session types)
  session_path:          Absolute path to the session root directory
  raw_data_path:         Absolute path to the session's raw_data subdirectory
  processed_data_path:   Absolute path to the session's processed_data subdirectory
  status:                Lifecycle status: uninitialized | error | incomplete | processed | acquired
  uninitialized:         True if the session still has the nk.bin marker
  incomplete:            True / False / None (None when status="uninitialized", because the descriptor
                         is not read, or when status="error", because it could not be read)
  has_processed_data:    True when processed_data/ exists and is non-empty
  error_detail:          (only on status="error") human-readable error message
counts:                  Root-wide status tally across every discovered session (including errors)
total_projects, total_animals, total_sessions, root_directory
```

`status="error"` entries come in two shapes, both owned and documented by `/project-hierarchy`, which also states the
rule excluding them from the `projects[*]` aggregates. Because `filter_sessions_tool` drops them from `session_paths`,
a marker or descriptor problem silently shrinks the batch a downstream skill receives. You MUST surface every error
entry to the user before handing off.

### Session filtering

| Tool                   | Purpose                                                               |
|------------------------|-----------------------------------------------------------------------|
| `filter_sessions_tool` | Filters a session list by date range and inclusion-exclusion criteria |

Designed for chaining with `get_data_root_overview_tool`, this tool accepts the flat `sessions` list from that output
and returns a filtered subset with the same structure.

**Parameters:**

| Parameter          | Type               | Default    | Description                                                                                                          |
|--------------------|--------------------|------------|----------------------------------------------------------------------------------------------------------------------|
| `sessions`         | `list[dict]`       | (required) | Session entries from `get_data_root_overview_tool` output                                                            |
| `start_date`       | `str / None`       | `None`     | Include sessions on or after this bound (`YYYY-MM-DD` or `YYYY-MM-DD HH:MM:SS`). A date-only value binds at midnight |
| `end_date`         | `str / None`       | `None`     | Include sessions on or before this bound (same two formats). A date-only value is rolled to the end of that day      |
| `include_sessions` | `list[str] / None` | `None`     | Session names to include regardless of date range                                                                    |
| `exclude_sessions` | `list[str] / None` | `None`     | Session names to exclude (precedence over all inclusion)                                                             |
| `include_animals`  | `list[str] / None` | `None`     | Animal IDs to include, only these animals considered                                                                 |
| `exclude_animals`  | `list[str] / None` | `None`     | Animal IDs to exclude (precedence over `include_animals`)                                                            |
| `utc_timezone`     | `bool`             | `True`     | Keyword-only. Interpret dates in UTC, `False` for host local time                                                    |

**Filtering precedence:** Animal filtering is applied before session filtering. Exclusion always takes precedence over
inclusion. The `exclude_sessions` list overrides both `include_sessions` and date range criteria. Each input entry must
carry `session_name` and `animal` keys, matching the shape produced by `get_data_root_overview_tool`.

The date-range pass runs only when `start_date` or `end_date` is supplied. Inside that pass, a session whose name does
not parse as the 7-component `YYYY-MM-DD-HH-MM-SS-microseconds` grammar is dropped without an error. With no date bound
the pass is skipped and such names survive.

**Return structure:** Structurally identical to the input shape, carrying `sessions`, `session_paths`, `total_sessions`,
and `total_eligible`. Entries with `status="error"`, and entries carrying no `session_path` key, are excluded from
`session_paths` but remain in `sessions` so the agent can surface them to the user. `sessions` is sorted by
`(session_name, animal, session_path)` and `session_paths` by path. An `invalid_entries` key appears when input entries
lack the required `session_name` or `animal` fields.

---

## Discovery workflow

### Step 1: Confirm root directory

Ask the user for the absolute path to the directory to search. A data-root or project-level root is the typical input.
When the host has a data root persisted via `/working-directory`, `read_data_root_tool` supplies a default to confirm
with the user rather than asking them to type it. For project hierarchy conventions, see `/project-hierarchy`.

### Step 2: Run discovery

Call `get_data_root_overview_tool` with the confirmed `root_directory`. Present the result as a readable summary:

```text
**Session Discovery** for /data/projects/my_project
Found 12 session(s): 9 acquired, 2 processed, 1 error (descriptor load failure)

| Session                          | Animal | Type                 | Status       |
|----------------------------------|--------|----------------------|--------------|
| 2026-03-04-14-30-00-000000       | 1      | lick training        | acquired     |
| 2026-03-05-10-15-00-000000       | 1      | run training         | processed    |
| 2026-03-06-09-00-00-000000       | 2      | mesoscope experiment | acquired     |
| 2026-03-07-11-00-00-000000       | 3      | window checking      | error        |
```

Surface any sessions with `status="error"` and their `error_detail`, because broken session metadata may require repair
via `/session-data` or `/session-descriptors`.

### Step 3: Optionally narrow by session type or project (client-side)

`get_data_root_overview_tool` does not accept server-side session-type, project, or animal filters. When the user wants
to operate only on certain session types or on a specific project or animal, filter the flat `sessions` list client-side
before step 4:

```text
filtered = [entry for entry in response["sessions"]
            if entry["project"] == "<project>"
            and entry["session_type"] in {"mesoscope experiment", "run training"}]
```

### Step 4: Optionally filter by date / name / animal

If the user wants to narrow the list by date range, animal, or specific session names, call `filter_sessions_tool` with
the `sessions` list from step 2, or the client-side-narrowed list from step 3, and the requested criteria.

### Step 5: Confirm and hand off

Present the final `session_paths` list to the user. Once confirmed, hand off to the appropriate downstream skill:
- The forging plugin's `forging:batch-processing` for data integrity operations
- The forging plugin's `forging:project-state` for manifest generation
- The forging plugin's `forging:batch-processing` for every processing pipeline, filtering by session type first
- The forging plugin's `forging:dataset-definition` for dataset composition, taking session names rather than paths
- The forging plugin's `forging:dataset-forging` for dataset assembly
- The experiment plugin's `experiment:data-management` for preprocessing, migration, and deletion. That skill owns no
  discovery tools, and it rejects any session path that does not lie inside the data root

---

## Error routing

The table below covers the literal messages the library emits via `resolve_root_directory` and the discovery / filter
tools, plus the conditions that surface outside the response's `error` key. Every message rides in the envelope
documented in the `## Response contract` section of `/assets-mcp-environment-setup`. A per-entry failure arrives instead
in the `error_detail` field of a session entry or the `filter_error` field of an `invalid_entries` entry.

| Error message                                                                       | Resolution                                                                                                       |
|-------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `Unable to resolve the root data directory. The path <path> does not exist.`        | Verify the root directory path with the user                                                                     |
| `Unable to resolve the root data directory. The path <path> is not a directory.`    | Path points at a file or symlink. Ask for the directory                                                          |
| `Unable to resolve the data root from <root_directory>.`                            | Defensive resolver branch. Re-confirm the root directory path with the user                                      |
| `Unable to scan the data root <root> for session and dataset markers: <reason>`     | The scan hit a directory it cannot read. Fix the permissions or scan a readable root                             |
| `Failed to load SessionData: <reason>` (per-entry, status="error")                  | Session marker is corrupt or missing required keys. Repair via `/session-data` or `/session-descriptors`         |
| `Descriptor file not found at <path>` (per-entry, status="error")                   | The marker loaded, the descriptor did not. Repair via `/session-descriptors`                                     |
| `Missing required 'session_name' or 'animal' field.` (per-entry, `invalid_entries`) | The key is absent or carries a `null` value. Entries not produced by `get_data_root_overview_tool` often lack it |
| `filter_sessions_tool` raises instead of returning a response                       | `start_date` or `end_date` is unparsable. Pass `YYYY-MM-DD` or `YYYY-MM-DD HH:MM:SS`                             |
| `sessions=[]` or `total_eligible=0` (no error, empty result)                        | No markers matched. Verify the search root or filter criteria                                                    |
| MCP tool call raises at the transport layer                                         | Invoke `/assets-mcp-environment-setup`                                                                           |

---

## Library API

The functions below are the public session-discovery surface `sollertia_shared_assets` exports at the package root. They
serve library and pipeline code that runs outside an MCP session. You MUST NOT import them to drive discovery from an
agent, because the MCP tools above already wrap them and return a structured response.

```python
validate_directory(directory: str) -> str | None
discover_sessions(root_path: Path) -> list[Path]
iterate_sessions(root_path: Path) -> Iterator[SessionData]
get_session_root_from_marker(marker: Path) -> Path
discover_projects(root_path: Path, strategy: Literal["markers", "directories"] = "markers") -> list[ProjectData]
iter_project_animals(project: ProjectData) -> Iterator[AnimalData]
iter_animal_sessions(animal: AnimalData) -> Iterator[Path]
get_projects_for_animal(root_path: Path, animal_id: str) -> tuple[str, ...]
filter_sessions(sessions: Iterable[tuple[str, str]], *, start_date: str | None = None, end_date: str | None = None,
                include_sessions: set[str] | None = None, exclude_sessions: set[str] | None = None,
                include_animals: set[str] | None = None, exclude_animals: set[str] | None = None,
                utc_timezone: bool = True) -> set[tuple[str, str]]
parse_session_timestamp(session_name: str, *, utc_timezone: bool = True) -> datetime | None
```

| Function                       | Semantics a caller cannot infer from the signature                                                            |
|--------------------------------|---------------------------------------------------------------------------------------------------------------|
| `validate_directory`           | Takes a `str`, returns `None` on success and a message string on failure, and never raises                    |
| `discover_sessions`            | Returns sorted session root paths, and raises `OSError` on an unreadable directory rather than skipping it    |
| `iterate_sessions`             | Lazy over `SessionData`, the scan runs on the first `next()`, and a failed marker load propagates             |
| `get_session_root_from_marker` | Path arithmetic alone (the marker's grandparent), with no I/O and no check that the marker exists             |
| `discover_projects`            | Defaults to the authoritative `markers` strategy, which loads every session marker under the root             |
| `iter_project_animals`         | Naturally sorted and directory-based, so animals holding no sessions are included                             |
| `iter_animal_sessions`         | Yields session roots in lexicographic order, reading no markers and skipping unparsable names                 |
| `get_projects_for_animal`      | Returns naturally sorted project names taken from markers, so a project surfaces only when it holds a session |
| `filter_sessions`              | Takes `(session_name, animal)` tuples, returns a set, and propagates a `ValueError` on an unparsable bound    |
| `parse_session_timestamp`      | Parses the 7-component `YYYY-MM-DD-HH-MM-SS-microseconds` grammar, returning `None` for any other name        |

`validate_directory` is the contract behind the validation error `forging:dataset-forging` reports for a `project_root`
that does not exist or is not a directory. Its message string is surfaced verbatim.

---

## Related skills

| Skill                             | Relationship                                                                                     |
|-----------------------------------|--------------------------------------------------------------------------------------------------|
| `/cli-reference`                  | Reference: the `slsa` bootstrap commands, which carry no discovery counterpart                   |
| `/assets-mcp-environment-setup`   | Prerequisite: MCP server connectivity                                                            |
| `/working-directory`              | Required prerequisite. Persists the data root that `read_data_root_tool` supplies as the default |
| `/project-hierarchy`              | Owns `get_data_root_overview_tool` as the tree walk                                              |
| `/session-data`                   | Reference: SessionData marker and `inspect_sessions_tool` for per-session health                 |
| `/session-descriptors`            | Reference: per-session descriptor repair                                                         |
| `/datasets`                       | Downstream: reads and audits the dataset container once `forging:dataset-definition` composes it |
| `forging:project-state`           | Downstream: manifest reading and generation                                                      |
| `forging:batch-processing`        | Downstream: consumes confirmed session_paths                                                     |
| `forging:batch-processing`        | Downstream: consumes confirmed session_paths                                                     |
| `forging:dataset-definition`      | Downstream: composes a dataset from the confirmed session names                                  |
| `forging:dataset-forging`         | Downstream: consumes confirmed session names                                                     |
| `forging:processing-input-format` | Reference: the artifacts each processing pipeline requires on disk                               |
| `experiment:data-management`      | Downstream: consumes confirmed session paths inside the data root                                |

---

## Verification checklist

```text
Session discovery:
- [ ] Verified sollertia-shared-assets MCP connectivity (invoked /assets-mcp-environment-setup if unavailable)
- [ ] Confirmed root directory with user
- [ ] Ran get_data_root_overview_tool and reviewed results
- [ ] Surfaced error-status sessions to the user
- [ ] Applied filter_sessions_tool if date, animal, or session-name filtering was requested
- [ ] Confirmed final session_paths list with user
- [ ] Handed off to the appropriate downstream skill
```
