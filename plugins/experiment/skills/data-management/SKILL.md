---
name: data-management
description: >-
  Manages the post-acquisition session lifecycle through the `sle mcp` server: the shared preprocessing primitives,
  preprocessing sessions singly or in bulk, migrating animals between projects, and deleting sessions with explicit
  confirmation. Use when the user asks to preprocess, migrate, or delete session data.
user-invocable: false
---

# Managing session data

Guides agents through the post-acquisition session lifecycle: log assembly, video renaming, surgery snapshotting,
transfer to long-term storage, deletion, and animal migration between projects.

Every acquisition system exposes the same three session-lifecycle operations and differs only in the system-specific
work each one performs. The reusable building blocks live in `cross_system/data_preprocessing.py`, and each system
supplies its own orchestration around them. `AcquisitionSystems` currently holds one member, so Mesoscope-VR is the
only registered acquisition system, and `mesoscope:mesoscope-vr-runtime` owns its lifecycle specifics.

---

## Scope

**Covers:**
- The six shared preprocessing primitives of `cross_system/data_preprocessing.py`
- The behavior logger-name contract and the `StorageDestinations` interface
- Preprocessing acquisition sessions, individually or in bulk
- Migrating an animal's sessions between projects
- Deleting sessions, with explicit confirmation at both the tool gate and the acquisition host terminal

**Does not cover:**
- Project and session discovery, and project creation (`assets:project-hierarchy` and `assets:session-discovery`)
- The `SessionData` marker and the `nk.bin` uninitialized-session flag (`assets:session-data`)
- Session creation and data acquisition (`/acquisition-system-runtime`)
- System and hardware configuration, including the storage destinations a system declares (`mesoscope:mesoscope-vr`)
- The current worked example's preprocessing, migration, and purge specifics (`mesoscope:mesoscope-vr-runtime`)
- Processing the transferred session (`forging:batch-processing`) and dataset forging (`forging:dataset-forging`)

---

## MCP server requirements

This skill uses the single `sollertia-experiment` MCP server (`sle mcp`). If it is unavailable, run
`/experiment-mcp-environment-setup`.

| Server                 | CLI command | Purpose                                               |
|------------------------|-------------|-------------------------------------------------------|
| `sollertia-experiment` | `sle mcp`   | Session preprocessing, deletion, and animal migration |

---

## Session lifecycle tools

The `sle mcp` server discovers every `interfaces/*_tools.py` module by filesystem glob when it starts, so a system's
session-lifecycle tools live in that system's own `interfaces/<system>_tools.py` module, discovered by
`_register_tool_modules()` in `interfaces/mcp_server.py`. Resolve the concrete tool names from the connected server's
tool listing. For the current worked example's tool names and their semantics, see `mesoscope:mesoscope-vr-runtime`.

| Operation  | Argument surface                                             | Result                        |
|------------|--------------------------------------------------------------|-------------------------------|
| Preprocess | one absolute `session_path`                                  | Success string, or `Error: …` |
| Delete     | one absolute `session_path` plus a `confirm_deletion` policy | Success string, or `Error: …` |
| Migrate    | source project, destination project, animal identifier       | Success string, or `Error: …` |

All three return a plain string, so they report failure through a leading `Error: ` prefix rather than through a
raised exception. Tools that return a mapping instead signal failure with a single `error` key. This skill is the
**exclusive** owner of the active system's preprocess, delete, and migrate tools. No other skill may call them.

Every `session_path` MUST be an absolute path to a session directory located **inside the data root**, resolved by
`get_data_root()` (`sollertia-shared-assets/src/sollertia_shared_assets/configuration/configuration_utilities.py`)
and set through `slsa configure data-root`. The tools reject any path outside the data root, including paths that
resolve onto long-term storage destinations.

A system's tool and its CLI command may normalize that path differently. The current worked example tests containment
with the unresolved path in both `preprocess_session_tool` and `delete_session_tool`
(`interfaces/mesoscope_vr_tools.py`) and resolves both operands
in the CLI, so a `..` segment or a symlink is treated differently by the two routes. You SHOULD pass a resolved
absolute path.

---

## The logger-name contract

`BEHAVIOR_LOGGER_NAME = "behavior"` names the `DataLogger` instance that records the behavior data of every acquisition
system (`cross_system/data_preprocessing.py`). That name composes the `_LOG_DIRECTORY_NAME` constant in the same
module, which yields the `behavior_data_log` directory that `assemble_session_logs` looks for, so every acquisition
system MUST name its behavior `DataLogger` `"behavior"`.

---

## StorageDestinations

`StorageDestination(name, session_path)` is a frozen dataclass that names one resolved long-term storage destination,
and `StorageDestinations(destinations=())` is the ordered collection of them, which is the system-agnostic interface
the shared utilities operate on (`cross_system/data_preprocessing.py`). Each acquisition system resolves its own
destinations from its own configuration and hands over the resolved paths, so no shared utility reads a system
configuration. An empty collection is meaningful, because it tells the utilities that the host keeps its data locally.

---

## Shared preprocessing primitives

A system's three lifecycle orchestrators compose these six functions and add their own conversion, compression, and
cleanup steps around them.

### assemble_session_logs

`assemble_session_logs(session_data, processes)` returns without acting when the log directory is absent, and when it
holds neither unarchived entries nor archives. It raises `RuntimeError` when the directory holds both `.npy` entries
and `.npz` archives, because archiving overwrites existing archives. It archives in place, removes a stale
`behavior_data` directory with a WARNING, then renames the log directory onto `behavior_data`, so an interrupted run
resumes on the next call (`cross_system/data_preprocessing.py`).

### rename_session_videos

`rename_session_videos(session_data)` resolves the camera manifest from `behavior_data/` or from the un-archived log
directory, and returns early when neither holds one. It renames each acquired video to
`<session_name>_<source.name>.mp4` and skips a source whose video file is missing. The manifest carries the mapping
from source identifiers to names, so the acquisition system supplies **no** static source-ID map
(`cross_system/data_preprocessing.py`).

### snapshot_surgery_data

`snapshot_surgery_data(session_data, animal_id, credentials_path, surgery_sheet_id)` writes the animal's surgery record
into the session's raw data and returns the **open** `SurgeryLog` handle. The caller owns that handle and MUST `close()`
it from a `try/finally` block to release the underlying SSL socket (`cross_system/data_preprocessing.py`). See
`/google-sheets-processing`.

### push_session_data

`push_session_data(session_data, destinations, threads)` warns and returns early on an empty destination collection,
keeping the local copy intact. Otherwise it computes an xxHash3-128 checksum over the raw-data directory, fans out
`transfer_directory(remove_source=False)` across a process pool, and propagates any transfer failure through
`future.result()`. On success it deletes the **entire local session directory**, `processed_data` included, because it
removes the parent of the transferred raw-data directory (`cross_system/data_preprocessing.py`). Warn the user
about that whole-directory removal before you preprocess a session carrying processed data.

### delete_session_directories

`delete_session_directories(candidates, session_name, *, require_confirmation)` is destructive and irreversible. When
confirmation is requested it echoes a WARNING and calls `request_confirmation(default=False)`, returning `False` and
deleting nothing when the user declines (`cross_system/data_preprocessing.py`). That call opens a blocking
`questionary` prompt on the host terminal, which an agent cannot answer (`cross_system/terminal_prompts.py`).

### migrate_session_directory

`migrate_session_directory(remote_session_path, local_session_path, old_session_data_path, target_project, threads)`
pulls one session from a storage destination with `verify_integrity=False`. It copies the pulled
`raw_data/session_data.yaml` back to the source project's host path, recreating the `raw_data` directory that
preprocessing removed, then sets `project_name`, saves, and returns a reloaded `SessionData`
(`cross_system/data_preprocessing.py`).

---

## When to use this skill

- **Preprocessing**: "Preprocess all sessions for project X" / "for animal 12345" / "the session at /path"
- **Migrating**: "Move animal 12345 from project A to project B"
- **Deleting**: "Delete the session at /path" / "Remove failed session data"

---

## Discovering sessions

This server has no discovery tools. To build a list of session paths, hand off to the assets plugin, which owns project
and session discovery against the data root:

- `assets:project-hierarchy` walks the data root through `get_data_root_overview_tool` to enumerate projects, animals,
  and sessions.
- `assets:session-discovery` filters sessions by project, animal, date range, or name and returns confirmed absolute
  session paths.

Use the session paths those skills return as the `session_path` arguments here. The data root layout is
`{data_root}/{project}/{animal_id}/{session_name}/`.

When you need the session types the local host can run, query `list_supported_session_types_tool` **scoped to this
host's acquisition system**, because this skill always operates within a single configured system.
`list_session_type_support_tool` returns the full system-to-session-type map when you need to compare systems.

---

## Preprocessing workflow

Preprocessing aggregates the session's data, applies the system's own conversion and compression steps, updates the
Google Sheets logs when they are configured, and transfers the raw data to the configured long-term storage
destinations. When no destinations are configured, the transfer and the local-copy removal are skipped and the data
remains on the acquisition host (the empty-destination branch of `push_session_data` in
`cross_system/data_preprocessing.py`).

### Uninitialized sessions are purged

A session whose `raw_data` still carries the `nk.bin` uninitialized-session marker never finished initialization and
holds no valid data, so a system's preprocessing entry point purges it instead of preprocessing it.
`SessionData.create()` writes the marker when the session is minted, and `mark_runtime_initialized()` removes it once
acquisition is ready to begin (`sollertia-shared-assets/src/sollertia_shared_assets/data_hierarchy/session_data.py`).
The purge removes the session from every storage location covered by [Session deletion
workflow](#session-deletion-workflow) and runs with no confirmation prompt, because the worked example's `purge_session`
requests confirmation only for a session that lacks the marker (`mesoscope_vr/data_preprocessing.py`).

Treat a preprocessing request as potentially destructive whenever a session may have crashed during initialization.
The tool returns its normal success string on both paths, so establish which sessions carry the marker before you call
it. The assets plugin reports a per-session `uninitialized` flag through `get_data_root_overview_tool` and
`inspect_sessions_tool`, so list the marked sessions up front and report them as purged afterwards.

### Multiple sessions (by project, animal, or all)

```text
Bulk preprocessing progress:
- [ ] Discovered sessions via assets:project-hierarchy or assets:session-discovery
- [ ] Presented discovered sessions to the user for confirmation
- [ ] Listed the sessions carrying the nk.bin marker, which preprocessing purges
- [ ] Preprocessed each session sequentially
- [ ] Reported results (success count, purges, failures)
```

---

## Animal migration workflow

Migration transfers all sessions for an animal from one project to another and reassigns them. The strategy depends on
whether the host has any long-term storage destinations configured:

- **With configured destinations** the first configured destination is treated as the source of truth. Any session
  that resides only in the local data root is **preprocessed automatically** first, which moves it to the
  destinations. Each session is then pulled back and re-preprocessed under the target project, and its obsolete
  source-project copies are purged from the host and from every configured destination before the next session
  starts. No manual preprocessing step is required before migration.
- **Without configured destinations** all data lives only on the acquisition host, so migration relocates each locally
  stored session directory to the target project and reassigns it entirely on-premises.

Both modes then relocate the animal's cross-session persistent data and clean up the source project. The
source-project animal directory is deleted in full from the data root and from every configured storage destination,
so each animal is found under at most one project directory. Unconfigured roots are skipped, because their unset
values resolve to relative paths that are unsafe to delete. A system adds its own machines to both the relocation and
the cleanup, see `mesoscope:mesoscope-vr-runtime`.

In both modes the **target project must already exist**, or the migration aborts with `FileNotFoundError` and a message
naming the missing target project (`migrate_animal_between_projects` in `mesoscope_vr/data_preprocessing.py`).
Project directories are created with the `create_project_tool` MCP tool or the `slsa configure project -p <name>` CLI
command. Use `assets:project-hierarchy` to verify whether the destination project exists, or to create it.

`migrate_animal_tool` maps its `destination_project` argument to the library's `target_project` keyword
(`interfaces/mesoscope_vr_tools.py`), so the two names refer to the same project.

Migration **fails with an error if any session cannot be preprocessed or migrated**, for example when the animal is
absent from the surgery sheet that preprocessing reads. Each session is an isolated unit that cleans up after itself
on failure, so re-running the migrate tool after the fix resumes from the failed session.

```text
Animal migration progress:
- [ ] Destination project exists (verify via assets:project-hierarchy), created via create_project_tool
      or `slsa configure project` when missing
- [ ] Confirmed migration with the user (source project, destination project, animal identifier)
- [ ] Warned the user that migration also moves the animal's persistent data and deletes the source-project
      animal directory from the data root and from every configured storage destination
- [ ] Executed the migrate tool
- [ ] On failure, resolved the reported error and re-ran the migrate tool to resume
- [ ] Reported completion
```

---

## Session deletion workflow

**CRITICAL: Session deletion is irreversible. It removes the session from the local data root, from every configured
long-term storage destination, and from the acquisition system's other machines. You MUST always obtain explicit user
confirmation before proceeding.**

You MUST follow this exact workflow for ALL deletion requests:

1. **Warn the user** about the consequences of deletion.
2. **Use AskUserQuestion** to get explicit confirmation.
3. **Only proceed** if the user explicitly confirms.

The delete tool carries a tri-state `confirm_deletion` policy argument rather than a boolean, so a falsy default can
never reach the purge. Omitting the argument returns an `Error:` string that spells out the irreversibility and names
the two accepted values. A `"no"` value returns an abandonment notice. Only `"yes"` reaches the purge.

Clearing that gate does not remove the second one. A session that lacks the `nk.bin` marker still needs a human at the
acquisition host, because `delete_session_directories` opens a blocking terminal prompt the agent cannot answer. A
session that carries the marker skips that prompt. Two outcomes need care in your reporting:

- The user declines the terminal prompt. The purge deletes nothing, and the tool still returns its success string.
  Verify the session directory is gone before you report success.
- The host has no attached terminal. The prompt fails and the tool returns an opaque `Error:` message. Ask the user to
  run the deletion from a terminal session on the acquisition host.

A system's CLI delete command may carry no confirmation gate of its own, which is the case for the current worked
example, so it reaches the purge on invocation and relies on the terminal prompt alone. Never offer the CLI form as a
way around the tool's policy argument.

```text
Session deletion progress:
- [ ] Verified the session path is absolute and inside the data root
- [ ] Displayed the deletion warning to the user
- [ ] Used AskUserQuestion to get explicit confirmation
- [ ] If confirmed, called the delete tool with confirm_deletion="yes"
- [ ] Told the user to answer the blocking confirmation prompt on the acquisition host terminal
- [ ] Verified the session directory is gone before treating the success string as a completed deletion
- [ ] Reported the result
```

### Bulk deletion

For multiple deletions you MUST obtain confirmation for EACH session individually, because batch confirmation is NOT
allowed. Present the full list, then for each session display the path and warning, confirm via AskUserQuestion, delete
only if confirmed, and report before moving to the next.

---

## Downstream handoff

Once the session's raw data reaches long-term storage, it is ready for the sollertia-forgery pipeline. Hand off to
`forging:project-state` for the project manifest file and its tooling, then to `forging:batch-processing` for the
per-session pipelines. `forging:server-configuration` owns the remote compute server's access configuration.

---

## Error handling

| Error                                            | Cause                                                      | Solution                                               |
|--------------------------------------------------|------------------------------------------------------------|--------------------------------------------------------|
| "Session directory must be inside the data root" | Path is outside the local data root                        | Only process sessions under `get_data_root()`          |
| "The target project does not exist"              | Destination project not created                            | Create it with `create_project_tool`                   |
| Preprocessing failure during migration           | A session cannot be preprocessed or migrated               | Resolve the error and re-run the migration to resume   |
| `Error:` naming the accepted confirmation values | `confirm_deletion` was omitted                             | Confirm via AskUserQuestion, then retry with `"yes"`   |
| Success string with data still present           | The host terminal prompt was declined                      | Verify the session is gone before reporting success    |
| Opaque `Error:` from the delete tool             | No terminal attached on the acquisition host               | Re-run the deletion from the acquisition host terminal |
| `RuntimeError` from log assembly                 | The log directory holds `.npy` entries and `.npz` archives | Back up and remove the existing archives, then retry   |

---

## Related skills

| Skill                               | Relationship                                                               |
|-------------------------------------|----------------------------------------------------------------------------|
| `/cli-reference`                    | Owns the `sle` command surface a user runs by hand when the server is down |
| `/experiment-mcp-environment-setup` | Run first if the `sle mcp` server is not connected                         |
| `/google-sheets-processing`         | Owns the `SurgeryLog` and `WaterLog` processors preprocessing invokes      |
| `/library-extension`                | Catalogues the preprocessing primitives as a reusable seam                 |
| `/pipeline`                         | Phase 7 (post-process and manage) is owned by this skill                   |
| `assets:project-hierarchy`          | Enumerates projects, animals, and sessions, read-only                      |
| `assets:session-data`               | Owns the `SessionData` marker and the `nk.bin` marker semantics            |
| `assets:session-discovery`          | Filters sessions and returns the paths these tools consume                 |
| `mesoscope:mesoscope-vr`            | Owns the worked example's storage-destination configuration                |
| `mesoscope:mesoscope-vr-runtime`    | Owns the worked example's preprocessing and purge specifics                |
| `forging:project-state`             | Owns the project manifest downstream processing reads                      |
| `forging:batch-processing`          | Consumes the transferred raw data                                          |
| `forging:server-configuration`      | Owns the remote transfer and cloud compute configuration                   |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Session data management:
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] Session paths are absolute and inside the data root
- [ ] Discovery handed off to the assets plugin (no sle project or session listing tool exists)
- [ ] Preprocessing reported per-session results, including any session purged for carrying the nk.bin marker
- [ ] Migration prerequisites confirmed before calling the migrate tool
- [ ] Every deletion obtained its own explicit AskUserQuestion confirmation before confirm_deletion="yes"
```
