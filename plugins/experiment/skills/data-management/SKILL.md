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

These tools are platform-general — preprocessing, transfer, and deletion live in
`sollertia_experiment/cross_system/data_preprocessing.py` and operate on any Sollertia acquisition
system's session data, not only Mesoscope-VR.

---

## MCP server requirements

This skill uses the single `sollertia-experiment` MCP server (`sle mcp`). If it is unavailable, run
`/experiment-mcp-environment-setup`.

| Server                  | CLI command | Purpose                                                  |
|-------------------------|-------------|----------------------------------------------------------|
| `sollertia-experiment`  | `sle mcp`   | Session preprocessing, deletion, and animal migration    |

This skill is the **exclusive** owner of `preprocess_session_tool`, `delete_session_tool`, and
`migrate_animal_tool` — no other skill in the marketplace may call them.

---

## Available MCP tools

| Tool                      | Signature                                                              | Purpose                                               |
|---------------------------|-----------------------------------------------------------------------|-------------------------------------------------------|
| `preprocess_session_tool` | `(session_path: str)`                                                 | Preprocesses a single session's data                  |
| `delete_session_tool`     | `(session_path: str, *, confirm_deletion: bool = False)`              | Removes a session from all storage destinations       |
| `migrate_animal_tool`     | `(source_project: str, destination_project: str, animal_id: str)`     | Transfers all sessions for an animal between projects  |

Every `session_path` MUST be an absolute path to a session directory located **inside the platform
data root** (resolved by `get_data_root()` in sollertia-shared-assets, set via `slsa configure
data-root`). The tools reject paths that point at long-term storage destinations.

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
which owns project and session discovery against the shared data root:

- assets plugin `/project-hierarchy` — walks the data root (`get_data_root_overview_tool`) to
  enumerate projects, animals, and sessions.
- assets plugin `/session-discovery` — filters sessions by project, animal, date range, or name and
  returns confirmed absolute session paths.

Use the session paths those skills return as the `session_path` arguments here. The data root layout
is `{data_root}/{project}/{animal_id}/{session_name}/`.

---

## Preprocessing workflow

Preprocessing aggregates session data, compresses mesoscope frames, updates the configured Google
Sheets logs, and transfers data to every configured long-term storage destination
(`filesystem.storage_directories`).

### Single session

```text
preprocess_session_tool(session_path="/data_root/project/animal/session")
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

Migration transfers all sessions for an animal from one project to another across all storage
destinations. The `migrate_animal_tool` enforces these health checks automatically:

1. **Target project must exist** — project directories are created implicitly by the first session
   created under them (via `SessionData.create`). If the destination project has no sessions yet,
   create one there first (assets plugin `/project-hierarchy`). There is no project-creation MCP tool.
2. **All local sessions must be preprocessed** — no unprocessed sessions can remain in the local data
   root for the source animal.
3. **Source animal must have sessions on a long-term storage destination** — migration pulls data
   from configured storage.

```text
Animal migration progress:
- [ ] Verified the destination project exists (assets plugin /project-hierarchy); created it if missing
- [ ] Checked for unprocessed local sessions for the source animal
- [ ] Preprocessed any unprocessed sessions first
- [ ] Confirmed migration with the user (source, destination, animal_id)
- [ ] Executed migrate_animal_tool
- [ ] Reported completion
```

```text
migrate_animal_tool(source_project="old_project", destination_project="new_project", animal_id="12345")
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
# Preview only (returns a safety warning, deletes nothing)
delete_session_tool(session_path="/data_root/project/animal/session")

# After explicit user confirmation via AskUserQuestion
delete_session_tool(session_path="/data_root/project/animal/session", confirm_deletion=True)
```

### Bulk deletion

For multiple deletions you MUST obtain confirmation for EACH session individually — batch
confirmation is NOT allowed. Present the full list, then for each session display the path and
warning, confirm via AskUserQuestion, delete only if confirmed, and report before the next.

---

## Error handling

| Error                                     | Cause                                      | Solution                                          |
|-------------------------------------------|--------------------------------------------|---------------------------------------------------|
| "Session directory must be inside the data root" | Path is on a storage destination, not local | Only process sessions under `get_data_root()`  |
| "target project does not exist"           | Destination project not created            | Hand off to assets plugin `/project-hierarchy`    |
| "non-preprocessed session data"           | Unprocessed sessions exist for the animal  | Preprocess all sessions before migration          |
| "requires explicit confirmation"          | `confirm_deletion` not set to `True`       | Get user confirmation via AskUserQuestion, then set `True` |

---

## Related skills

| Skill                                       | Relationship                                                             |
|---------------------------------------------|--------------------------------------------------------------------------|
| `/experiment-mcp-environment-setup`         | Run first if the `sle mcp` server is not connected                       |
| assets plugin `/project-hierarchy`          | Enumerates projects/animals/sessions; creates projects                   |
| assets plugin `/session-discovery`          | Filters sessions and returns confirmed session paths to feed these tools |
| assets plugin `/session-data`               | Owns the `SessionData` marker that defines each session                  |
| `/mesoscope-vr`                             | Defines `filesystem.storage_directories`, the transfer destinations      |
| forging plugin `/server-configuration`      | Remote storage transfer configuration consumed downstream                |
| `/pipeline`                                 | Phase 7 (post-process and manage) is owned by this skill                 |

---

## Verification checklist

```text
Session data management:
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] Session paths are absolute and inside the platform data root
- [ ] Discovery handed off to the assets plugin (no sle project/session listing tool exists)
- [ ] Preprocessing reported per-session results
- [ ] Migration prerequisites confirmed before calling migrate_animal_tool
- [ ] Every deletion obtained its own explicit AskUserQuestion confirmation before confirm_deletion=True
```
