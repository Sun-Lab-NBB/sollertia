---
name: session-discovery
description: >-
  Guides use of sollertia-forgery MCP tools for discovering sessions under a project root and
  filtering by date range, animal, or session name. Produces confirmed session path lists consumed
  by downstream batch skills. Use when locating sessions ahead of checksum, transfer, manifest,
  or any processing of session data including dataset assembly, or when filtering a previously
  discovered session list.
user-invocable: true
---

# Session discovery

Discovers and filters Sollertia sessions via the shared MCP tools. This skill is domain-agnostic —
it provides the raw discover → filter surface that any downstream batch skill can chain from. For
behavior-processing eligibility rules, see `/behavior-input-format`.

---

## Scope

**Covers:**
- Recursively discovering sessions under a project root via `session_data.yaml` markers
- Optional session type filtering at discovery time
- Post-discovery filtering by date range, animal inclusion/exclusion, and session inclusion/exclusion
- Producing confirmed `session_paths` lists for downstream batch skills

**Does not cover:**
- Reading or generating project manifest files (see `/project-manifest`)
- Checksum verification or regeneration (see `/checksum-verification`)
- Session transfer or deletion (see `/session-transfer`)
- Behavior-processing eligibility rules (see `/behavior-input-format`)
- MCP server connectivity issues (see `/forging-mcp-environment-setup`)

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for session discovery. Do not scan the filesystem
manually or import `sollertia_forgery.shared_assets.session_discovery` directly.

You MUST confirm the root directory path with the user before calling `discover_sessions_tool`.

---

## Available tools

### Session discovery

| Tool                     | Purpose                                                       |
|--------------------------|---------------------------------------------------------------|
| `discover_sessions_tool` | Recursively discovers sessions by `session_data.yaml` markers |

**Parameters:**

| Parameter        | Type               | Default    | Description                                           |
|------------------|--------------------|------------|-------------------------------------------------------|
| `root_directory` | `str`              | (required) | Absolute path to root directory; searched recursively |
| `session_types`  | `list[str] / None` | `None`     | Session type strings for eligibility filtering        |

Valid `session_types` values: `"lick_training"`, `"run_training"`, `"mesoscope_experiment"`,
`"window_checking"`. When omitted, all discovered sessions are returned as eligible.

**Return structure:**

```text
sessions[]:              Per-session entries:
  session_path:          Absolute path to the session root directory
  session_name:          Session name (format: YYYY-MM-DD-HH-MM-SS-microseconds, 7 components)
  animal_id:             Animal identifier string
  session_type:          Session type string
  acquisition_system:    Acquisition system string
  raw_data_path:         Absolute path to the session's raw_data subdirectory
  processed_data_path:   Absolute path to the session's processed_data subdirectory
  eligible:              True if the session matches the type filter (or no filter applied)
  error:                 (only on load failure) human-readable error message
session_paths:           Flat list of eligible session root paths (pass to downstream batch tools)
total_sessions:          Total sessions found (including ineligible and errored)
total_eligible:          Count of sessions in session_paths
```

`session_paths` is the pre-filtered list of eligible roots — pass it directly to downstream
prepare tools (`prepare_checksum_batch_tool`, `prepare_transfer_batch_tool`, or
`prepare_behavior_processing_batch_tool`).

### Session filtering

| Tool                   | Purpose                                                               |
|------------------------|-----------------------------------------------------------------------|
| `filter_sessions_tool` | Filters a session list by date range and inclusion/exclusion criteria |

Designed for chaining with `discover_sessions_tool`: accepts the `sessions` list from its output
and returns a filtered subset with the same structure.

**Parameters:**

| Parameter          | Type               | Default    | Description                                                 |
|--------------------|--------------------|------------|-------------------------------------------------------------|
| `sessions`         | `list[dict]`       | (required) | Session entries from `discover_sessions_tool` output        |
| `start_date`       | `str / None`       | `None`     | Include sessions on or after this date (`YYYY-MM-DD`)       |
| `end_date`         | `str / None`       | `None`     | Include sessions on or before this date (includes full day) |
| `include_sessions` | `list[str] / None` | `None`     | Session names to include regardless of date range           |
| `exclude_sessions` | `list[str] / None` | `None`     | Session names to exclude (precedence over all inclusion)    |
| `include_animals`  | `list[str] / None` | `None`     | Animal IDs to include; only these animals considered        |
| `exclude_animals`  | `list[str] / None` | `None`     | Animal IDs to exclude (precedence over `include_animals`)   |
| `utc_timezone`     | `bool`             | `True`     | Interpret dates in UTC; `False` for America/New_York        |

**Filtering precedence:** Animal filtering is applied before session filtering. Exclusion always
takes precedence over inclusion. The `exclude_sessions` list overrides both `include_sessions` and
date range criteria.

**Return structure:** Structurally identical to `discover_sessions_tool` output — contains
`sessions`, `session_paths`, `total_sessions`, and `total_eligible`. An `invalid_entries` key
appears when input entries lack required `session_name` or `animal_id` fields.

---

## Discovery workflow

### Step 1: Confirm root directory

Ask the user for the absolute path to the directory to search. A project-level root is the
typical input. For project hierarchy conventions, see `/configuration:project-hierarchy`.

### Step 2: Run discovery

Call `discover_sessions_tool` with the confirmed `root_directory` and an optional `session_types`
filter if the user wants only certain session types. Present the result as a readable summary:

```text
**Session Discovery** — /data/projects/my_project
Found 12 session(s): 9 eligible, 3 ineligible

| Session                          | Animal | Type                 | Eligible |
|----------------------------------|--------|----------------------|----------|
| 2026-03-04-14-30-00-000000       | 1      | lick_training        | yes      |
| 2026-03-05-10-15-00-000000       | 1      | run_training         | yes      |
| 2026-03-06-09-00-00-000000       | 2      | mesoscope_experiment | yes      |
| 2026-03-07-11-00-00-000000       | 3      | window_checking      | no       |
```

Surface any sessions with an `error` field — broken session metadata may require repair via
`/configuration:session-data` or `/configuration:session-descriptors`.

### Step 3: Optionally filter

If the user wants to narrow the list by date range, animal, or specific session names, call
`filter_sessions_tool` with the `sessions` list from step 2 and the requested criteria.

### Step 4: Confirm and hand off

Present the final `session_paths` list to the user. Once confirmed, hand off to the appropriate
downstream skill:
- `/checksum-verification` for data integrity operations
- `/session-transfer` for transfer or deletion
- `/project-manifest` for manifest generation
- `/behavior-processing` for behavior extraction (filter by eligible types first)

---

## Error routing

| Error                                | Resolution                                                     |
|--------------------------------------|----------------------------------------------------------------|
| `Directory does not exist`           | Verify the root directory path with the user                   |
| `Path is not a directory`            | The path points to a file; ask for the correct directory       |
| `Permission denied during search`    | Filesystem ACLs block recursion; resolve or try a subdirectory |
| `Invalid session type in filter`     | Check the session type string against valid enum values        |
| `Unable to load session: <reason>`   | Session metadata is corrupt or missing                         |
| `sessions=[]` or `total_eligible=0`  | No markers found or no eligible types; verify the search root  |
| MCP tool unavailable                 | Invoke `/forging-mcp-environment-setup`                        |

---

## Related skills

| Skill                              | Relationship                                               |
|------------------------------------|------------------------------------------------------------|
| `/forging-mcp-environment-setup`   | Prerequisite: MCP server connectivity                      |
| `/project-manifest`                | Downstream: manifest reading and generation                |
| `/checksum-verification`           | Downstream: consumes confirmed session_paths               |
| `/session-transfer`                | Downstream: consumes confirmed session_paths               |
| `/behavior-processing`             | Downstream: consumes confirmed session_paths               |
| `/behavior-input-format`           | Reference: behavior-processing eligibility rules           |
| `/configuration:session-data`      | Reference: SessionData marker and SessionTypes enum        |
| `/configuration:project-hierarchy` | Reference: project / animal / session layout               |

---

## Verification checklist

```text
Session Discovery:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Confirmed root directory with user
- [ ] Ran discover_sessions_tool and reviewed results
- [ ] Surfaced ineligible and errored sessions to the user
- [ ] Applied filter_sessions_tool if date/animal/session filtering was requested
- [ ] Confirmed final session_paths list with user
- [ ] Handed off to the appropriate downstream skill
```
