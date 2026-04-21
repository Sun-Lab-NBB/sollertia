---
name: session-discovery
description: >-
  Guides use of sollertia-shared-assets MCP tools for discovering Sollertia sessions under a project
  root and filtering them by date range, animal, or session name. Produces confirmed session path
  lists consumed by every downstream batch skill across the sollertia-forgery plugin. Use when
  locating sessions ahead of checksum verification, transfer, manifest generation, behavior or dataset
  processing, or any workflow that starts from a set of discovered sessions; also use when filtering
  a previously discovered session list.
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
- Recursively discovering sessions under a project root via `session_data.yaml` markers
- Optional session-type eligibility filtering at discovery time
- Post-discovery filtering by date range, animal inclusion-exclusion, and session inclusion-exclusion
- Producing confirmed `session_paths` lists for downstream batch skills

**Does not cover:**
- Walking the full project tree (projects, animals, experiments) — see `/project-hierarchy`
- Reading individual `SessionData` markers — see `/session-data`
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

You MUST confirm the root directory path with the user before calling `discover_sessions_tool`.

---

## Available tools

### Session discovery

| Tool                     | Purpose                                                       |
|--------------------------|---------------------------------------------------------------|
| `discover_sessions_tool` | Recursively discovers sessions by `session_data.yaml` markers |

**Parameters:**

| Parameter        | Type               | Default    | Description                                                |
|------------------|--------------------|------------|------------------------------------------------------------|
| `root_directory` | `str / None`       | `None`     | Absolute path to root directory; searched recursively      |
| `project`        | `str / None`       | `None`     | Narrows the search to the given project subtree            |
| `animal_id`      | `str / None`       | `None`     | Narrows the search to the given animal under `project`     |
| `session_types`  | `list[str] / None` | `None`     | Session type strings for eligibility classification        |

Valid `session_types` values: `"lick training"`, `"run training"`, `"mesoscope experiment"`,
`"window checking"`. When omitted, every discovered session is marked eligible. `project` and
`animal_id` hard-narrow the search tree; `session_types` classifies each discovered entry without
removing it from the response, so agents can still see ineligible or broken sessions for diagnosis.

**Return structure:**

```text
sessions[]:              Per-session entries:
  session_name:          Session name (format: YYYY-MM-DD-HH-MM-SS-microseconds, 7 components)
  project:               Project name the session belongs to
  animal:                Animal identifier string
  session_type:          Session type string
  acquisition_system:    Acquisition system string
  experiment_name:       Experiment name (None for non-experiment session types)
  session_path:          Absolute path to the session root directory
  raw_data_path:         Absolute path to the session's raw_data subdirectory
  processed_data_path:   Absolute path to the session's processed_data subdirectory
  incomplete:            True if the session still has an nk.bin acquisition marker
  eligible:              True if the session matches the session_types filter (or no filter applied)
  error:                 (only on load failure) human-readable error message
session_paths:           Flat list of eligible session root paths (pass to downstream batch tools)
total_sessions:          Total sessions found (including ineligible and errored)
total_eligible:          Count of sessions in session_paths
root_directory:          Resolved root directory the walk started from
```

`session_paths` is the pre-filtered list of eligible roots — pass it directly to downstream
prepare tools such as the sollertia-forgery plugin's `prepare_checksum_batch_tool`,
`prepare_transfer_batch_tool`, `prepare_behavior_processing_batch_tool`, or `prepare_forging_batch_tool`.

### Session filtering

| Tool                   | Purpose                                                               |
|------------------------|-----------------------------------------------------------------------|
| `filter_sessions_tool` | Filters a session list by date range and inclusion-exclusion criteria |

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
date range criteria. Each input entry must carry `session_name` and `animal` keys (matching the
shape produced by `discover_sessions_tool`).

**Return structure:** Structurally identical to `discover_sessions_tool` output — contains
`sessions`, `session_paths`, `total_sessions`, and `total_eligible`. An `invalid_entries` key
appears when input entries lack the required `session_name` or `animal` fields.

---

## Discovery workflow

### Step 1: Confirm root directory

Ask the user for the absolute path to the directory to search. A project-level root is the
typical input. For project hierarchy conventions, see `/project-hierarchy`.

### Step 2: Run discovery

Call `discover_sessions_tool` with the confirmed `root_directory` and an optional `session_types`
filter when the user wants to classify only certain session types as eligible. Present the result
as a readable summary:

```text
**Session Discovery** — /data/projects/my_project
Found 12 session(s): 9 eligible, 3 ineligible

| Session                          | Animal | Type                 | Eligible |
|----------------------------------|--------|----------------------|----------|
| 2026-03-04-14-30-00-000000       | 1      | lick training        | yes      |
| 2026-03-05-10-15-00-000000       | 1      | run training         | yes      |
| 2026-03-06-09-00-00-000000       | 2      | mesoscope experiment | yes      |
| 2026-03-07-11-00-00-000000       | 3      | window checking      | no       |
```

Surface any sessions with an `error` field — broken session metadata may require repair via
`/session-data` or `/session-descriptors`.

### Step 3: Optionally filter

If the user wants to narrow the list by date range, animal, or specific session names, call
`filter_sessions_tool` with the `sessions` list from step 2 and the requested criteria.

### Step 4: Confirm and hand off

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

| Error                                | Resolution                                                     |
|--------------------------------------|----------------------------------------------------------------|
| `Directory does not exist`           | Verify the root directory path with the user                   |
| `Path is not a directory`            | The path points to a file; ask for the correct directory       |
| `Permission denied during search`    | Filesystem ACLs block recursion; resolve or try a subdirectory |
| `Invalid session type in filter`     | Check the session type string against valid enum values        |
| `Failed to load session: <reason>`   | Session metadata is corrupt or missing                         |
| `sessions=[]` or `total_eligible=0`  | No markers found or no eligible types; verify the search root  |
| MCP tool unavailable                 | Invoke `/assets-mcp-environment-setup`                         |

---

## Related skills

| Skill                                        | Relationship                                         |
|----------------------------------------------|------------------------------------------------------|
| `/assets-mcp-environment-setup`              | Prerequisite: MCP server connectivity                |
| `/project-hierarchy`                         | Sibling: project / animal / session layout reference |
| `/session-data`                              | Reference: SessionData marker and SessionTypes enum  |
| `/session-descriptors`                       | Reference: per-session descriptor repair             |
| forging plugin `/project-manifest`           | Downstream: manifest reading and generation          |
| forging plugin `/checksum-verification`      | Downstream: consumes confirmed session_paths         |
| forging plugin `/session-transfer`           | Downstream: consumes confirmed session_paths         |
| forging plugin `/behavior-processing`        | Downstream: consumes confirmed session_paths         |
| forging plugin `/dataset-forging`            | Downstream: consumes confirmed session names         |
| forging plugin `/behavior-input-format`      | Reference: behavior-processing eligibility rules     |

---

## Verification checklist

```text
Session Discovery:
- [ ] Verified sollertia-shared-assets MCP connectivity (invoked /assets-mcp-environment-setup if unavailable)
- [ ] Confirmed root directory with user
- [ ] Ran discover_sessions_tool and reviewed results
- [ ] Surfaced ineligible and errored sessions to the user
- [ ] Applied filter_sessions_tool if date, animal, or session-name filtering was requested
- [ ] Confirmed final session_paths list with user
- [ ] Handed off to the appropriate downstream skill
```
