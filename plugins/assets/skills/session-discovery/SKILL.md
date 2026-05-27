---
name: session-discovery
description: >-
  Discovers Sollertia sessions under a project root and filters by date range, animal, or
  session name via the sollertia-shared-assets MCP server. Chains
  get_data_root_overview_tool through filter_sessions_tool to produce `session_paths` lists
  consumed by downstream batch skills. Use when locating sessions ahead of any batch workflow
  or filtering a previously discovered list.
user-invocable: true
---

# Session discovery

Discovers and filters Sollertia sessions via the sollertia-shared-assets MCP tools. This skill is
domain-agnostic — it provides the raw discover → filter surface that any downstream batch skill
can chain from. For behavior-processing eligibility rules, see the sollertia-forgery plugin's
`/behavior-input-format`.

---

## Scope

**Covers:**
- Recursively discovering sessions under a data root via `session_data.yaml` markers using
  `get_data_root_overview_tool`
- Post-discovery filtering by date range, animal inclusion-exclusion, and session
  inclusion-exclusion via `filter_sessions_tool`
- Producing confirmed `session_paths` lists for downstream batch skills

**Does not cover:**
- Walking the full project tree (projects, animals, experiments) as a primary workflow — see
  `/project-hierarchy`, which owns `get_data_root_overview_tool`
- Reading individual `SessionData` markers or full session health reports — see `/session-data`
- Reading or generating project manifest files — see the sollertia-forgery plugin's
  `/project-manifest`
- Checksum verification or regeneration — see the sollertia-forgery plugin's `/checksum-verification`
- Session transfer or deletion — see the sollertia-forgery plugin's `/session-transfer`
- Behavior-processing eligibility rules — see the sollertia-forgery plugin's `/behavior-input-format`
- MCP server connectivity issues — see `/assets-mcp-environment-setup`

---

## Agent requirements

You MUST use the sollertia-shared-assets MCP tools for session discovery. Do not scan the
filesystem manually or import `sollertia_shared_assets.discover_sessions` directly.

You MUST confirm the root directory path with the user before calling
`get_data_root_overview_tool`.

---

## Available tools

### Session discovery

| Tool                          | Purpose                                                                                                                                         |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| `get_data_root_overview_tool` | Recursively discovers sessions by `session_data.yaml` markers and builds the project / animal / session tree from `SessionData` identity fields |

**Parameters:**

| Parameter        | Type    | Default    | Description                                                |
|------------------|---------|------------|------------------------------------------------------------|
| `root_directory` | `str`   | (required) | Absolute path to root directory; searched recursively      |

There is no server-side `project`, `animal_id`, or `session_types` narrowing. The tool scans the
entire root and returns the full hierarchy; callers filter client-side (for project / animal /
session-type) or via `filter_sessions_tool` (for date range, include / exclude session names, and
include / exclude animals).

**Return structure (excerpt):**

```text
projects[]:              Per-project rollup:
  name:                  Project name (from SessionData.project_name)
  path:                  Absolute path to <root>/<project>
  animals[]:             Per-animal summary: id, session_paths, session_count, counts
  session_count:         Sessions under this project
  counts:                {uninitialized, incomplete, acquired, processed, error} status tally
  sessions_by_type:      Counts keyed by SessionTypes value
  experiment_count:      *.yaml files under <project>/configuration/
  dataset_count:         dataset.yaml markers under the project
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
  incomplete:            True / False / None (None when descriptor cannot be read; status="error")
  has_processed_data:    True when processed_data/ exists and is non-empty
  error_detail:          (only on status="error") human-readable error message
counts:                  Root-wide status tally across every discovered session (including errors)
total_projects, total_animals, total_sessions, root_directory
```

Sessions whose `SessionData` cannot be loaded surface with `status="error"` and an
`error_detail` field but are **not** assigned to any project or animal (their identity is
untrusted). They appear in the flat `sessions` list only; the project aggregation remains clean.

### Session filtering

| Tool                   | Purpose                                                               |
|------------------------|-----------------------------------------------------------------------|
| `filter_sessions_tool` | Filters a session list by date range and inclusion-exclusion criteria |

Designed for chaining with `get_data_root_overview_tool`: accepts the flat `sessions` list from
its output and returns a filtered subset with the same structure.

**Parameters:**

| Parameter          | Type               | Default    | Description                                                 |
|--------------------|--------------------|------------|-------------------------------------------------------------|
| `sessions`         | `list[dict]`       | (required) | Session entries from `get_data_root_overview_tool` output   |
| `start_date`       | `str / None`       | `None`     | Include sessions on or after this date (`YYYY-MM-DD`)       |
| `end_date`         | `str / None`       | `None`     | Include sessions on or before this date (includes full day) |
| `include_sessions` | `list[str] / None` | `None`     | Session names to include regardless of date range           |
| `exclude_sessions` | `list[str] / None` | `None`     | Session names to exclude (precedence over all inclusion)    |
| `include_animals`  | `list[str] / None` | `None`     | Animal IDs to include; only these animals considered        |
| `exclude_animals`  | `list[str] / None` | `None`     | Animal IDs to exclude (precedence over `include_animals`)   |
| `utc_timezone`     | `bool`             | `True`     | Interpret dates in UTC; `False` for host local time         |

**Filtering precedence:** Animal filtering is applied before session filtering. Exclusion always
takes precedence over inclusion. The `exclude_sessions` list overrides both `include_sessions` and
date range criteria. Each input entry must carry `session_name` and `animal` keys (matching the
shape produced by `get_data_root_overview_tool`).

**Return structure:** Structurally identical to the input shape — contains `sessions`,
`session_paths`, `total_sessions`, and `total_eligible`. Entries with `status="error"` are
excluded from `session_paths` but remain in `sessions` so the agent can surface them to the
user. An `invalid_entries` key appears when input entries lack the required `session_name` or
`animal` fields.

---

## Discovery workflow

### Step 1: Confirm root directory

Ask the user for the absolute path to the directory to search. A data-root or project-level
root is the typical input. When the host has a data root persisted via `/working-directory`,
`read_data_root_tool` supplies a default to confirm with the user rather than asking them to type it.
For project hierarchy conventions, see `/project-hierarchy`.

### Step 2: Run discovery

Call `get_data_root_overview_tool` with the confirmed `root_directory`. Present the result as a
readable summary:

```text
**Session Discovery** — /data/projects/my_project
Found 12 session(s): 9 acquired, 2 processed, 1 error (descriptor load failure)

| Session                          | Animal | Type                 | Status       |
|----------------------------------|--------|----------------------|--------------|
| 2026-03-04-14-30-00-000000       | 1      | lick training        | acquired     |
| 2026-03-05-10-15-00-000000       | 1      | run training         | processed    |
| 2026-03-06-09-00-00-000000       | 2      | mesoscope experiment | acquired     |
| 2026-03-07-11-00-00-000000       | 3      | window checking      | error        |
```

Surface any sessions with `status="error"` and their `error_detail` — broken session metadata
may require repair via `/session-data` or `/session-descriptors`.

### Step 3: Optionally narrow by session type or project (client-side)

`get_data_root_overview_tool` does not accept server-side session-type, project, or animal
filters. When the user wants to operate only on certain session types or on a specific project
or animal, filter the flat `sessions` list client-side before step 4:

```text
filtered = [entry for entry in response["sessions"]
            if entry["project"] == "<project>"
            and entry["session_type"] in {"mesoscope experiment", "run training"}]
```

### Step 4: Optionally filter by date / name / animal

If the user wants to narrow the list by date range, animal, or specific session names, call
`filter_sessions_tool` with the `sessions` list from step 2 (or the client-side-narrowed list
from step 3) and the requested criteria.

### Step 5: Confirm and hand off

Present the final `session_paths` list to the user. Once confirmed, hand off to the appropriate
downstream skill:
- Sollertia-forgery plugin's `/checksum-verification` for data integrity operations
- Sollertia-forgery plugin's `/session-transfer` for transfer or deletion
- Sollertia-forgery plugin's `/project-manifest` for manifest generation
- Sollertia-forgery plugin's `/behavior-processing` for behavior extraction (filter by eligible
  session types first)
- Sollertia-forgery plugin's `/dataset-forging` for dataset assembly

---

## Error routing

The strings below are the literal messages the library emits via `resolve_root_directory` and
the discovery / filter tools; match against the `error` or per-session `error_detail` field of
the response.

| Error message                                                      | Resolution                                                                                               |
|--------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| `Root directory does not exist: <path>`                            | Verify the root directory path with the user                                                             |
| `Root directory is not a directory: <path>`                        | Path points at a file or symlink; ask for the directory                                                  |
| `Failed to load SessionData: <reason>` (per-entry, status="error") | Session marker is corrupt or missing required keys; repair via `/session-data` / `/session-descriptors`  |
| Missing `session_name` or `animal` in `filter_sessions_tool` input | Entries from sources other than `get_data_root_overview_tool` may lack these keys; see `invalid_entries` |
| `sessions=[]` or `total_eligible=0` (no error, empty result)       | No markers matched; verify the search root or filter criteria                                            |
| MCP tool call raises at the transport layer                        | Invoke `/assets-mcp-environment-setup`                                                                   |

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

---

## Related skills

| Skill                                   | Relationship                                                                     |
|-----------------------------------------|----------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`         | Prerequisite: MCP server connectivity                                            |
| `/project-hierarchy`                    | Owns `get_data_root_overview_tool` as the tree walk                              |
| `/session-data`                         | Reference: SessionData marker and `inspect_sessions_tool` for per-session health |
| `/session-descriptors`                  | Reference: per-session descriptor repair                                         |
| forging plugin `/project-manifest`      | Downstream: manifest reading and generation                                      |
| forging plugin `/checksum-verification` | Downstream: consumes confirmed session_paths                                     |
| forging plugin `/session-transfer`      | Downstream: consumes confirmed session_paths                                     |
| forging plugin `/behavior-processing`   | Downstream: consumes confirmed session_paths                                     |
| forging plugin `/dataset-forging`       | Downstream: consumes confirmed session names                                     |
| forging plugin `/behavior-input-format` | Reference: behavior-processing eligibility rules                                 |
