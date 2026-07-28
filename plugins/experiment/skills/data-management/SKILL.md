---
name: data-management
description: >-
  Manages acquisition session data via the `sle mcp` server: preprocessing sessions (single or in
  bulk), migrating animals between projects, and deleting sessions with mandatory confirmation. Use
  when the user asks to preprocess, migrate, or delete session data.
user-invocable: false
---

# Managing session data

Guides agents through managing acquisition session data using the `sle mcp` server's session
lifecycle tools: preprocessing, animal migration between projects, and session deletion with
mandatory safety confirmations.

These tools form a system-agnostic session-management interface: every Sollertia acquisition system
exposes the same preprocess, delete, and migrate tools and differs only in the system-specific work
performed during preprocessing. The reusable, system-agnostic building blocks (transfer, deletion, and
migration helpers) live in `sollertia_experiment/cross_system/data_preprocessing.py`, and each
acquisition system supplies its own concrete orchestration. Mesoscope-VR is the current example: its
`preprocess_session_data`, `purge_session`, and `migrate_animal_between_projects` orchestrators live in
`sollertia_experiment/mesoscope_vr/data_preprocessing.py` and add system-specific steps such as
mesoscope-frame compression. The shared primitives a system's orchestrators compose are
`assemble_session_logs`, `push_session_data`, `delete_session_directories`, `migrate_session_directory`,
`rename_session_videos`, and `snapshot_surgery_data`.

---

## Scope

**Covers:**
- Preprocessing acquisition sessions, individually or in bulk, via `preprocess_session_tool`
- Migrating an animal's sessions between projects via `migrate_animal_tool`
- Deleting sessions, with mandatory confirmation, via `delete_session_tool`

**Does not cover:**
- Project and session discovery, and project creation (`assets:project-hierarchy` and
  `assets:session-discovery`; project creation via `create_project_tool` or the `slsa configure project` CLI)
- Session creation and data acquisition (acquisition-runtime skills)
- System and hardware configuration (`mesoscope:mesoscope-vr`)

---

## MCP server requirements

This skill uses the single `sollertia-experiment` MCP server (`sle mcp`). If it is unavailable, run
`/experiment-mcp-environment-setup`.

| Server                 | CLI command | Purpose                                               |
|------------------------|-------------|-------------------------------------------------------|
| `sollertia-experiment` | `sle mcp`   | Session preprocessing, deletion, and animal migration |

This skill is the **exclusive** owner of `preprocess_session_tool`, `delete_session_tool`, and
`migrate_animal_tool` — no other skill in the marketplace may call them.

---

## Available MCP tools

| Tool                      | Signature                                                         | Purpose                                               |
|---------------------------|-------------------------------------------------------------------|-------------------------------------------------------|
| `preprocess_session_tool` | `(session_path: str)`                                             | Preprocesses a single session's data                  |
| `delete_session_tool`     | `(session_path: str, *, confirm_deletion: bool = False)`          | Removes a session from all storage destinations       |
| `migrate_animal_tool`     | `(source_project: str, destination_project: str, animal_id: str)` | Transfers all sessions for an animal between projects |

Every `session_path` MUST be an absolute path to a session directory located **inside the data root**
(resolved by `get_data_root()` in sollertia-shared-assets, set via `slsa configure data-root`). The
tools reject any path that is not located inside the data root, including paths on long-term storage
destinations.

This server exposes **no** project- or session-listing tools. Discover projects and sessions through
the assets plugin (see [Discovering sessions](#discovering-sessions)).

**`preprocess_session_tool` deletes uninitialized sessions.** A session whose `raw_data` still carries the `nk.bin`
uninitialized-session marker never finished initialization and holds no valid data, so `preprocess_session_data` calls
`purge_session` and returns. The purge removes the session from every storage location covered by
[Session deletion workflow](#session-deletion-workflow) and runs with no confirmation prompt, because `purge_session`
passes `require_confirmation=not nk_path.exists()`. Treat a preprocessing request as potentially destructive whenever a
session may have crashed during initialization. The tool returns the same `Session preprocessed: {session_path}` string
for the preprocess path and the purge path, so establish which sessions carry the marker before you call it. The assets
plugin reports a per-session `uninitialized` flag through `get_data_root_overview_tool` and `inspect_sessions_tool`, so
list the marked sessions for the user up front and report them as purged afterwards.

---

## When to use this skill

**Preprocessing sessions:**
- "Preprocess all sessions for project X" / "for animal 12345" / "the session at /path" / "all sessions"

**Migrating animals:**
- "Move animal 12345 from project A to project B"

**Deleting sessions:**
- "Delete the session at /path" / "Remove failed session data"

---

## Discovering sessions

This server has no discovery tools. To build a list of session paths, hand off to the assets plugin,
which owns project and session discovery against the data root:

- `assets:project-hierarchy` — walks the data root (`get_data_root_overview_tool`) to
  enumerate projects, animals, and sessions.
- `assets:session-discovery` — filters sessions by project, animal, date range, or name and
  returns confirmed absolute session paths.

Use the session paths those skills return as the `session_path` arguments here. The data root layout
is `{data_root}/{project}/{animal_id}/{session_name}/`.

When you need the session types the local host can run (e.g. to validate a type before filtering),
query the assets plugin's `list_supported_session_types_tool` **scoped to this host's acquisition
system** — `data-management` always operates within a single configured system, so pass that system
rather than requesting the platform-wide list. `list_session_type_support_tool` returns the full
system-to-session-type map when you need to compare systems.

---

## Preprocessing workflow

Preprocessing aggregates the session's data, applies any system-specific conversion or compression
(for the Mesoscope-VR system this compresses the acquired mesoscope frames), updates the Google Sheets
logs when they are configured (some systems do not use Google Sheets at all), and transfers data to
the system's configured long-term storage destinations (the `StorageDestinations` collection in
`cross_system/data_preprocessing.py`; for Mesoscope-VR these come from `filesystem.storage_directories`
— see `mesoscope:mesoscope-vr`). When no storage destinations are configured, the transfer and the
local-copy removal are skipped, and preprocessing is limited to on-premises data conversion and
aggregation — the data remains on the acquisition host.

Sessions that still carry the `nk.bin` uninitialized-session marker take the purge path described under
[Available MCP tools](#available-mcp-tools), so they are deleted rather than preprocessed.

The Mesoscope-VR system also runs DeepLabCut face-camera eye-tracking inference during preprocessing. It launches
asynchronously right after the session videos are renamed, so it occupies the rig's otherwise-idle GPU while the
CPU-bound and disk-bound stages run, and it covers `SessionTypes.MESOSCOPE_EXPERIMENT` sessions only. The stage is
opt-in. It runs only when the host's `video_tracking` configuration section sets both `conda_environment` and
`dlc_project_path`, and it is skipped with a warning when the expected face-camera video is absent. Inference runs
`slvt infer` inside the configured conda environment through `conda run`, because `slvt` requires Python 3.12 and
numpy 1.x to satisfy DeepLabCut 3.0.0 while the rest of the Sollertia stack runs Python 3.14 and numpy 2. `slvt` ships
no MCP server and no plugin, so its CLI is the only agent-facing surface.

Preprocessing joins the inference subprocess immediately before the transfer to long-term storage. A non-zero exit
status or zero written `.h5` prediction files raises `RuntimeError`, aborts the push to long-term storage, and retains
the local session copy for a manual retry. The transient log at `<tmp>/slvt_infer_<session_name>.log` is removed on
success, is kept on failure, and its last 2000 characters are echoed into the error. A successful run writes the
DeepLabCut `.h5` file and its companion pickles beside the face-camera video in `raw_data/camera_data/`, so the raw-data
checksum covers them and they reach long-term storage as raw data.

### Single session

```text
preprocess_session_tool(session_path=...)  # absolute path to a session directory inside the data root
```

### Multiple sessions (by project, animal, or all)

1. **Discover sessions** via `assets:project-hierarchy` or `assets:session-discovery`.
2. **Present the list** of discovered session paths to the user for confirmation.
3. **Preprocess each session** sequentially with `preprocess_session_tool`.
4. **Report results**, including any failures.

```text
Bulk preprocessing progress:
- [ ] Discovered sessions via assets:project-hierarchy or assets:session-discovery
- [ ] Presented discovered sessions to the user for confirmation
- [ ] Preprocessed each session sequentially
- [ ] Reported results (success count, failures)
```

---

## Animal migration workflow

Migration transfers all sessions for an animal from one project to another and reassigns them. The
strategy depends on whether the host has any long-term storage destinations configured (the
`StorageDestinations` collection in `cross_system/data_preprocessing.py`; for Mesoscope-VR these come
from `filesystem.storage_directories` — see `mesoscope:mesoscope-vr`):

- **With configured destinations** — the first configured destination is treated as the source of
  truth. Any session that still resides only in the local data root is **preprocessed automatically**
  first (moving it to the destinations), then each session is pulled back from the source of truth and
  re-preprocessed under the target project. Each session's obsolete source-project copies are then purged from the
  host and from every configured storage destination before the loop moves to the next session. No manual
  preprocessing step is required before migration.
- **Without configured destinations** — all data lives only on the acquisition host, so migration
  relocates each locally stored session directory to the target project and reassigns it entirely
  on-premises.

Both modes then relocate the animal's cross-session persistent data and clean up the source project. The ScanImagePC
`mesoscope_directory/{source_project}/{animal}` tree moves to `{target_project}/{animal}`, preserving the animal's
`MotionEstimator` and ROI data, and the host's per-animal `persistent_data` directory moves the same way. Each move
overwrites an existing target directory. The source-project animal directory is then deleted in full from the mesoscope
directory, the data root, and every configured storage destination, so each animal is found under at most one project
directory everywhere. Unconfigured roots are skipped, because their unset values resolve to relative paths that are
unsafe to delete.

In both modes the **target project must already exist**, or `migrate_animal_tool` aborts with
`FileNotFoundError` (raised by `migrate_animal_between_projects` when the target project is missing).
Project directories are created with the `create_project_tool` MCP tool or the `slsa configure project`
CLI command (`slsa configure project -p <name>`). Use `assets:project-hierarchy` to verify whether the
destination project exists or to create it.

Migration **fails with an error if any session cannot be preprocessed or migrated** (for example, when
the animal is absent from the surgery sheet that preprocessing reads). Each session is handled as an
isolated unit that cleans up after itself on failure, so once the underlying problem is resolved,
re-running `migrate_animal_tool` resumes from the failed session — already-migrated sessions are not
reprocessed.

```text
Animal migration progress:
- [ ] Destination project exists (verify via assets:project-hierarchy); create via create_project_tool
      or `slsa configure project` if missing
- [ ] Confirmed migration with the user (source, destination, animal_id)
- [ ] Warned the user that migration also moves the animal's persistent data and deletes the source-project
      animal directory from the mesoscope directory, the data root, and every configured storage destination
- [ ] Executed migrate_animal_tool
- [ ] On failure, resolved the reported error and re-ran migrate_animal_tool to resume
- [ ] Reported completion
```

```text
migrate_animal_tool(source_project=..., destination_project=..., animal_id=...)
```

---

## Session deletion workflow

**CRITICAL: Session deletion is irreversible. It removes the session from the local data root, from every configured
long-term storage destination, and from the acquisition system's other machines, which for Mesoscope-VR means the
session-specific ScanImagePC mesoscope directory. `purge_session` then empties the shared ScanImagePC `mesoscope_data`
directory of any files left behind by the purged runtime. You MUST always obtain explicit user confirmation before
proceeding.**

You MUST follow this exact workflow for ALL deletion requests:

1. **Warn the user** about the consequences of deletion.
2. **Use AskUserQuestion** to get explicit confirmation.
3. **Only proceed** if the user explicitly confirms.

Never call `delete_session_tool` with `confirm_deletion=True` without first obtaining user
confirmation through AskUserQuestion. Called with `confirm_deletion=False` (the default), the tool
returns a safety warning instead of deleting data — use that to preview.

`confirm_deletion=True` clears the tool's own guard, and the deletion still needs a human at the acquisition host.
`purge_session` passes `require_confirmation=True` for any session that lacks the `nk.bin` marker, so
`delete_session_directories` issues a second, blocking `questionary` prompt on the host's terminal that the agent
cannot answer. A session that still carries the `nk.bin` marker skips that prompt and is deleted immediately. Two
outcomes need care in your reporting:

- The user declines the terminal prompt. `purge_session` returns having deleted nothing, and the tool still returns the
  success string `Session deleted: {session_path}`. Verify the session directory is gone before you report success.
- The host has no attached terminal. The prompt fails and the tool returns an opaque `Error:` message. Ask the user to
  run the deletion from a terminal session on the acquisition host.

```text
Session deletion progress:
- [ ] Verified the session path exists inside the data root
- [ ] Displayed the deletion warning to the user
- [ ] Used AskUserQuestion to get explicit confirmation
- [ ] If confirmed, executed delete_session_tool with confirm_deletion=True
- [ ] Told the user to answer the blocking confirmation prompt on the acquisition host terminal
- [ ] Verified the session directory is gone before treating the success string as a completed deletion
- [ ] Reported the result
```

```text
delete_session_tool(session_path=...)                        # preview: returns a safety warning, deletes nothing
delete_session_tool(session_path=..., confirm_deletion=True) # deletes. Needs AskUserQuestion and a host terminal answer
```

### Bulk deletion

For multiple deletions you MUST obtain confirmation for EACH session individually — batch
confirmation is NOT allowed. Present the full list, then for each session display the path and
warning, confirm via AskUserQuestion, delete only if confirmed, and report before the next.

---

## Error handling

| Error                                            | Cause                                          | Solution                                                   |
|--------------------------------------------------|------------------------------------------------|------------------------------------------------------------|
| "Session directory must be inside the data root" | Path is on a storage destination, not local    | Only process sessions under `get_data_root()`              |
| "target project does not exist"                  | Destination project not created                | Create it with `create_project_tool`                       |
| Preprocessing failure                            | Session cannot be preprocessed or migrated     | Resolve the error and re-run the migration to resume       |
| "requires explicit confirmation"                 | `confirm_deletion` not set to `True`           | Get user confirmation via AskUserQuestion, then set `True` |
| "Session deleted" with data still present        | Host terminal prompt was declined              | Verify the session is gone before reporting success        |
| Opaque `Error:` from `delete_session_tool`       | No terminal attached on the acquisition host   | Re-run the deletion from the host terminal                 |
| Face-camera inference failure                    | `slvt infer` exited non-zero or wrote no `.h5` | Read the inference log, fix the cause, re-preprocess       |

---

## Related skills

| Skill                               | Relationship                                                                                                               |
|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| `/experiment-mcp-environment-setup` | Run first if the `sle mcp` server is not connected                                                                         |
| `assets:project-hierarchy`          | Enumerates projects/animals/sessions (read-only)                                                                           |
| `assets:session-discovery`          | Filters sessions and returns confirmed session paths to feed these tools                                                   |
| `assets:session-data`               | Owns the `SessionData` marker that defines each session                                                                    |
| `mesoscope:mesoscope-vr`            | Defines `filesystem.storage_directories`, the transfer destinations                                                        |
| `/google-sheets-processing`         | Owns the `SurgeryLog` / `WaterLog` processors that preprocessing invokes to snapshot surgery data and update the water log |
| `forging:server-configuration`      | Remote storage transfer configuration consumed downstream                                                                  |
| `/pipeline`                         | Phase 7 (post-process and manage) is owned by this skill                                                                   |

---

## Verification checklist

```text
Session data management:
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] Session paths are absolute and inside the data root
- [ ] Discovery handed off to the assets plugin (no sle project/session listing tool exists)
- [ ] Preprocessing reported per-session results, including any session purged for carrying the nk.bin marker
- [ ] Migration prerequisites confirmed before calling migrate_animal_tool
- [ ] Every deletion obtained its own explicit AskUserQuestion confirmation before confirm_deletion=True
```
