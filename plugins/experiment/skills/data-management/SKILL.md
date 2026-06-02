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
mesoscope-frame compression.

---

## Scope

**Covers:**
- Preprocessing acquisition sessions, individually or in bulk, via `preprocess_session_tool`
- Migrating an animal's sessions between projects via `migrate_animal_tool`
- Deleting sessions, with mandatory confirmation, via `delete_session_tool`

**Does not cover:**
- Project and session discovery, and project creation (assets plugin `/project-hierarchy` and
  `/session-discovery`; the `slsa configure project` CLI command)
- Session creation and data acquisition (acquisition-runtime skills)
- System and hardware configuration (`/mesoscope-vr`)

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

- assets plugin `/project-hierarchy` — walks the data root (`get_data_root_overview_tool`) to
  enumerate projects, animals, and sessions.
- assets plugin `/session-discovery` — filters sessions by project, animal, date range, or name and
  returns confirmed absolute session paths.

Use the session paths those skills return as the `session_path` arguments here. The data root layout
is `{data_root}/{project}/{animal_id}/{session_name}/`.

---

## Preprocessing workflow

Preprocessing aggregates the session's data, applies any system-specific conversion or compression
(for the Mesoscope-VR system this compresses the acquired mesoscope frames), updates the Google Sheets
logs when they are configured (some systems do not use Google Sheets at all), and transfers data to
every configured long-term storage destination (`filesystem.storage_directories`). When no storage
destinations are configured, the transfer and the local-copy removal are skipped, and preprocessing is
limited to on-premises data conversion and aggregation — the data remains on the acquisition host.

### Single session

```text
preprocess_session_tool(session_path=...)  # absolute path to a session directory inside the data root
```

### Multiple sessions (by project, animal, or all)

1. **Discover sessions** via the assets plugin's `/project-hierarchy` or `/session-discovery`.
2. **Present the list** of discovered session paths to the user for confirmation.
3. **Preprocess each session** sequentially with `preprocess_session_tool`.
4. **Report results**, including any failures.

```text
Bulk preprocessing progress:
- [ ] Discovered sessions via assets plugin /project-hierarchy or /session-discovery
- [ ] Presented discovered sessions to the user for confirmation
- [ ] Preprocessed each session sequentially
- [ ] Reported results (success count, failures)
```

---

## Animal migration workflow

Migration transfers all sessions for an animal from one project to another and reassigns them. The
strategy depends on whether the host has any long-term storage destinations configured
(`filesystem.storage_directories`):

- **With configured destinations** — the first configured destination is treated as the source of
  truth. Each session is pulled back from it, re-preprocessed under the target project, and the
  obsolete source-project copies are purged. This mode requires that **no non-preprocessed sessions
  remain in the local data root** for the source animal; `migrate_animal_tool` aborts with an error if
  any do.
- **Without configured destinations** — all data lives only on the acquisition host, so migration
  relocates each locally stored session directory to the target project and reassigns it entirely
  on-premises.

In both modes the **target project must already exist**, or `migrate_animal_tool` aborts. Project
directories are created by the `slsa configure project` CLI command (`slsa configure project -p <name>
-r <root>`), not by an MCP tool — `SessionData.create` raises `FileNotFoundError` when the project is
missing. Use the assets plugin `/project-hierarchy` to verify whether the destination project exists.

```text
Animal migration progress:
- [ ] Destination project exists (verify via /project-hierarchy); created with `slsa configure project` if missing
- [ ] If storage destinations are configured, preprocessed any non-preprocessed local sessions first
- [ ] Confirmed migration with the user (source, destination, animal_id)
- [ ] Executed migrate_animal_tool
- [ ] Reported completion
```

```text
migrate_animal_tool(source_project=..., destination_project=..., animal_id=...)
```

---

## Session deletion workflow

**CRITICAL: Session deletion is irreversible and removes data from the local data root and every
configured long-term storage destination. You MUST always obtain explicit user confirmation before
proceeding.**

You MUST follow this exact workflow for ALL deletion requests:

1. **Warn the user** about the consequences of deletion.
2. **Use AskUserQuestion** to get explicit confirmation.
3. **Only proceed** if the user explicitly confirms.

Never call `delete_session_tool` with `confirm_deletion=True` without first obtaining user
confirmation through AskUserQuestion. Called with `confirm_deletion=False` (the default), the tool
returns a safety warning instead of deleting data — use that to preview.

```text
Session deletion progress:
- [ ] Verified the session path exists inside the data root
- [ ] Displayed the deletion warning to the user
- [ ] Used AskUserQuestion to get explicit confirmation
- [ ] If confirmed, executed delete_session_tool with confirm_deletion=True
- [ ] Reported the result
```

```text
delete_session_tool(session_path=...)                        # preview: returns a safety warning, deletes nothing
delete_session_tool(session_path=..., confirm_deletion=True) # deletes — needs explicit AskUserQuestion confirmation
```

### Bulk deletion

For multiple deletions you MUST obtain confirmation for EACH session individually — batch
confirmation is NOT allowed. Present the full list, then for each session display the path and
warning, confirm via AskUserQuestion, delete only if confirmed, and report before the next.

---

## Error handling

| Error                                            | Cause                                            | Solution                                                   |
|--------------------------------------------------|--------------------------------------------------|------------------------------------------------------------|
| "Session directory must be inside the data root" | Path is on a storage destination, not local      | Only process sessions under `get_data_root()`              |
| "target project does not exist"                  | Destination project not created                  | Create it with `slsa configure project`                    |
| "non-preprocessed session data"                  | Non-preprocessed local sessions exist for animal | Preprocess all local sessions before migration             |
| "requires explicit confirmation"                 | `confirm_deletion` not set to `True`             | Get user confirmation via AskUserQuestion, then set `True` |

---

## Related skills

| Skill                                  | Relationship                                                             |
|----------------------------------------|--------------------------------------------------------------------------|
| `/experiment-mcp-environment-setup`    | Run first if the `sle mcp` server is not connected                       |
| assets plugin `/project-hierarchy`     | Enumerates projects/animals/sessions (read-only)                         |
| assets plugin `/session-discovery`     | Filters sessions and returns confirmed session paths to feed these tools |
| assets plugin `/session-data`          | Owns the `SessionData` marker that defines each session                  |
| `/mesoscope-vr`                        | Defines `filesystem.storage_directories`, the transfer destinations      |
| forging plugin `/server-configuration` | Remote storage transfer configuration consumed downstream                |
| `/pipeline`                            | Phase 7 (post-process and manage) is owned by this skill                 |

---

## Verification checklist

```text
Session data management:
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] Session paths are absolute and inside the data root
- [ ] Discovery handed off to the assets plugin (no sle project/session listing tool exists)
- [ ] Preprocessing reported per-session results
- [ ] Migration prerequisites confirmed before calling migrate_animal_tool
- [ ] Every deletion obtained its own explicit AskUserQuestion confirmation before confirm_deletion=True
```
