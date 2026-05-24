---
name: project-manifest
description: >-
  Documents the Sollertia project manifest file (contents, columns, status flag dependencies)
  and the sollertia-forgery MCP tools for reading, generating, and cleaning manifests. Use
  when answering questions about project state, session processing status, or when generating
  a manifest.
user-invocable: true
---

# Project manifest

Documents the structure and meaning of the project manifest `.feather` file and guides manifest
reading, generation, and cleanup via the sollertia-forgery MCP tools. This skill is the
authoritative reference for interpreting manifest contents so the agent can answer user queries
about project state.

---

## Scope

**Covers:**
- Manifest column definitions, storage types, and semantic meaning
- Status flag dependency chain and interpretation rules
- Summarize output structure and how to derive answers from it
- Reading the manifest in targeted (single session) and bulk (all sessions) modes
- Generating, checking, and cleaning manifest files
- Common user query patterns and how to resolve them

**Does not cover:**
- Session discovery and filtering (see the assets plugin's `/session-discovery`)
- Checksum verification or regeneration (see `/checksum-verification`)
- Session transfer or deletion (see `/session-transfer`)
- Behavior processing (see `/behavior-processing`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)

---

## Manifest contents

The manifest is an uncompressed Apache Arrow IPC (`.feather`) file that captures a snapshot of
a project's state. Each row represents one data acquisition session. The manifest is the primary
entry point for querying which sessions exist, what processing has been applied, and what remains.

### Column reference

| Column                     | Storage        | Meaning                                                       |
|----------------------------|----------------|---------------------------------------------------------------|
| `animal`                   | `UInt64`       | Unique animal identifier, monotonically increasing            |
| `session`                  | `String`       | Session name: `YYYY-MM-DD-HH-MM-SS-microseconds` format, UTC  |
| `date`                     | `Datetime`     | Session timestamp in the host machine's local time            |
| `type`                     | `String`       | Session type (see session types below)                        |
| `system`                   | `String`       | Acquisition system used (e.g., `mesoscope-vr`)                |
| `notes`                    | `String`       | Free-text experimenter notes from session descriptor          |
| `complete`                 | `UInt8`        | Session data is complete (inverse of `incomplete` flag)       |
| `integrity`                | `UInt8`        | Checksum verification passed for this session                 |
| `cindra`                   | `UInt8`        | Cindra single-recording pipeline completed                    |
| `behavior`                 | `UInt8`        | Behavior extraction pipeline completed                        |
| `video`                    | `UInt8`        | DeepLabCut video tracking pipeline completed                  |
| `multi_recording_datasets` | `List(String)` | Cindra multi-recording dataset names for this session         |
| `multi_recording_complete` | `List(UInt8)`  | Per-dataset completion, index-aligned with datasets           |

`UInt8` status columns (`complete`, `integrity`, `cindra`, `behavior`, `video`) are returned as
native Python `bool` values by `get_project_manifest_tool`. `List(UInt8)` entries in
`multi_recording_complete` are similarly returned as `list[bool]`.

### Session types

| Value                  | Description                                              |
|------------------------|----------------------------------------------------------|
| `lick_training`        | Lick training session                                    |
| `run_training`         | Run training session                                     |
| `mesoscope_experiment` | Full mesoscope experiment session                        |
| `window_checking`      | Window checking session (may lack descriptor YAML)       |

### Status flag dependency chain

```text
complete ──→ integrity ──→ cindra
                       ──→ behavior
                       ──→ video
```

If `complete` is `False`, the session's data is not ready for processing. If `integrity` is
`False`, the checksum has not been verified. In either case, `cindra`, `behavior`, and `video`
are automatically `False` — manifest generation skips processing status checks for incomplete or
unverified sessions.

This means:
- A session with `integrity=False` cannot have `cindra=True`, regardless of whether the cindra
  pipeline actually ran. The manifest captures logical readiness, not raw tracker presence.
- To advance a session from `integrity=False` to processing-eligible, run checksum verification
  via `/checksum-verification`, then regenerate the manifest.

### Multi-recording datasets

The `multi_recording_datasets` column lists cindra multi-recording dataset names that the session
participates in. Completion status is resolved from a project-wide registry — the tracker lives
on the main recording of each dataset, so `multi_recording_complete` is populated by scanning all
tracker files across the project during generation. The two list columns are index-aligned: entry
`i` of `multi_recording_complete` corresponds to dataset `i` in `multi_recording_datasets`.

### Experimenter notes

The `notes` column contains free-text content from the session's `session_descriptor.yaml` file.
Notes are always included in targeted retrieval (single session) and excluded by default in bulk
retrieval to keep responses concise. `window_checking` sessions acquired before
sollertia-experiment 3.0.0 may lack descriptors; these show `"N/A"` for notes.

---

## Summarize output

When the manifest is read via `get_project_manifest_tool`, the response includes a `summary`
dictionary with aggregate statistics computed from the full manifest (unaffected by animal or
session filters):

```text
total_sessions:              Total rows in the manifest
total_animals:               Count of unique animal IDs
animals:                     List of animal IDs (sorted ascending)
session_types:               Dict mapping type string to session count
acquisition_systems:         Dict mapping system string to session count
complete_count:              Sessions where complete = True
integrity_verified_count:    Sessions where integrity = True
cindra_processed_count:      Sessions where cindra = True
behavior_processed_count:    Sessions where behavior = True
video_processed_count:       Sessions where video = True
multi_recording_datasets:
  total_datasets:            Number of distinct multi-recording datasets
  datasets[]:                Per-dataset entries:
    name:                    Dataset name string
    session_count:           Number of sessions in this dataset
    complete:                Whether the dataset processing is complete
columns:                     List of manifest column names
total_rows:                  Same as total_sessions
```

---

## Common queries

This section maps typical user questions to the manifest fields and retrieval modes needed to
answer them.

| User question                                      | How to answer                                                  |
|----------------------------------------------------|----------------------------------------------------------------|
| "Show me a project overview"                       | Bulk mode; present `summary`                                   |
| "Which sessions need checksum verification?"       | Bulk mode; filter where `integrity` is `False`                 |
| "How many sessions have behavior processing done?" | `behavior_processed_count` from `summary`                      |
| "What are the notes for session X?"                | Targeted mode with `session` parameter                         |
| "Which animals have incomplete sessions?"          | Bulk mode; group by `animal`, filter `complete=False`          |
| "Which multi-recording datasets are incomplete?"   | Check `multi_recording_datasets` in `summary`                  |
| "What processing is left for animal 2?"            | Bulk mode with `animal=2`; check status flag columns           |
| "Has cindra run on all verified sessions?"         | Compare `integrity_verified_count` vs `cindra_processed_count` |
| "List all mesoscope experiment sessions"           | Bulk mode; filter `type` = `mesoscope_experiment`              |

---

## Available tools

### Reading the manifest

| Tool                        | Purpose                                                     |
|-----------------------------|-------------------------------------------------------------|
| `get_project_manifest_tool` | Reads a `.feather` manifest and returns structured metadata |

**Parameters:**

| Parameter       | Type         | Default    | Description                                                   |
|-----------------|--------------|------------|---------------------------------------------------------------|
| `manifest_file` | `str`        | (required) | Absolute path to the `.feather` manifest file                 |
| `animal`        | `int / None` | `None`     | Filter sessions to this animal (bulk mode only)               |
| `session`       | `str / None` | `None`     | Retrieve a single session with full notes (targeted mode)     |
| `include_notes` | `bool`       | `False`    | Include notes column in bulk mode                             |

**Retrieval modes:**

- **Targeted** (when `session` is provided): Returns full data for that single session including
  experimenter notes. The `animal` and `include_notes` parameters are ignored.
- **Bulk** (when `session` is omitted): Returns all sessions, optionally filtered by `animal`.
  The `notes` column is excluded unless `include_notes=True`.

The `summary` in the response always reflects the full manifest regardless of filters.

**Return structure:**

```text
manifest_file:    Path to the manifest file
summary:          Aggregate statistics (see Summarize output above)
sessions[]:       Per-session dictionaries with all manifest columns
total_sessions:   Number of sessions in the response
```

### Generation and lifecycle

| Tool                                  | Purpose                                                  |
|---------------------------------------|----------------------------------------------------------|
| `generate_project_manifest_tool`      | Scans the project and writes a new manifest file         |
| `get_manifest_generation_status_tool` | Reads the generation tracker to check outcome            |
| `clean_project_manifest_tool`         | Deletes manifest file, tracker, and companion lock files |

**`generate_project_manifest_tool` parameters:**

| Parameter           | Type  | Default    | Description                                   |
|---------------------|-------|------------|-----------------------------------------------|
| `project_directory` | `str` | (required) | Absolute path to the project's root directory |

Scans every session under the project root, evaluates processing status from tracker files, and
writes `{project_name}_manifest.feather` to the project root. A `ProcessingTracker` at
`{project_root}/manifest_processing_tracker.yaml` records the generation outcome.

**`get_manifest_generation_status_tool`** takes `project_directory` (str, required). Returns the
tracker's per-job status. An absent tracker means no prior generation has been run.

**`clean_project_manifest_tool`** takes `project_directory` (str, required). Deletes four files
from the project root: the `.feather` manifest, `manifest_processing_tracker.yaml`, and their
companion `.lock` files.

---

## Manifest file location

- **Manifest:** `{project_root}/{project_name}_manifest.feather` where `project_name` is the
  directory stem
- **Tracker:** `{project_root}/manifest_processing_tracker.yaml`
- **Lock files:** `.feather.lock` and `.yaml.lock` (companion files for concurrent access)
- **Format:** Uncompressed Apache Arrow IPC; supports memory-mapped reads

---

## Workflows

### Read an existing manifest

1. Confirm the manifest file path with the user (typically
   `{project_root}/{project_name}_manifest.feather`)
2. Call `get_project_manifest_tool` with the appropriate mode:
   - For a project overview: bulk mode, no filters
   - For a specific animal: bulk mode with `animal` parameter
   - For a specific session's details: targeted mode with `session` parameter
3. Present the `summary` and relevant session data to the user
4. Answer follow-up queries using the column semantics documented above

### Generate a new manifest

1. Confirm the `project_directory` with the user
2. Check existing status via `get_manifest_generation_status_tool` — an absent tracker means no
   prior generation; an existing tracker shows whether the last generation succeeded or failed
3. Call `generate_project_manifest_tool` — this scans every session and may take several seconds
   for large projects
4. After success, read the generated manifest via `get_project_manifest_tool` and present the
   summary to the user

### Regenerate from scratch

1. Call `clean_project_manifest_tool` to remove all manifest artifacts
2. Call `generate_project_manifest_tool` to regenerate
3. Verify by reading the new manifest

### When to regenerate

The manifest is a snapshot — it does not update automatically. Regenerate after:
- Running checksum verification (updates `integrity` column)
- Completing cindra, behavior, or video processing
- Transferring or deleting sessions
- Adding new sessions to the project

---

## Error routing

| Error                                     | Resolution                                                   |
|-------------------------------------------|--------------------------------------------------------------|
| `Manifest file does not exist`            | Generate the manifest first, or verify the file path         |
| `Path is not a file`                      | The path points to a directory; provide the `.feather` path  |
| `Unable to read manifest file`            | File may be corrupt; clean and regenerate                    |
| `Session 'X' not found in manifest`       | Session may not exist or manifest is stale; regenerate       |
| `Animal ID 'N' not found in manifest`     | Verify the animal ID against the `animals` list in summary   |
| `Manifest generation failed`              | Check project directory path; ensure sessions exist          |
| `No manifest tracker found`               | Manifest has not been generated yet                          |
| MCP tool unavailable                      | Invoke `/forging-mcp-environment-setup`                      |

---

## Related skills

| Skill                                | Relationship                                                      |
|--------------------------------------|-------------------------------------------------------------------|
| `/forging-mcp-environment-setup`     | Prerequisite: MCP server connectivity                             |
| assets plugin `/session-discovery`   | Upstream: discover sessions before generating a manifest          |
| `/checksum-verification`             | Upstream: integrity column reflects checksum status               |
| `/session-transfer`                  | Upstream: regenerate manifest after transfer or deletion          |
| `/behavior-processing`               | Upstream: behavior column reflects extraction pipeline status     |
| `/configuration:project-hierarchy`   | Reference: project directory layout                               |

---

## Verification checklist

```text
Project Manifest:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Confirmed manifest file path or project directory with user
- [ ] For reading: called get_project_manifest_tool with appropriate retrieval mode
- [ ] Presented summary and session data to user
- [ ] Answered user queries using column semantics and status flag chain
- [ ] For generation: confirmed project directory; generated and verified manifest
- [ ] For regeneration: cleaned old artifacts before regenerating
```
