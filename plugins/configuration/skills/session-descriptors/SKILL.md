---
name: session-descriptors
description: >-
  Reads, writes, and validates per-session descriptor YAML files (LickTrainingDescriptor,
  RunTrainingDescriptor, WindowCheckingDescriptor, MesoscopeExperimentDescriptor) via the sl-configure
  MCP server. Owns the session descriptor write tools and schema introspection. Use when repairing,
  amending, or inspecting a session descriptor for any of the four supported session types.
user-invocable: true
---

# Sollertia session descriptors

Reads, writes, and validates the per-session descriptor YAML files that live inside every Sollertia
session directory. This skill is the **exclusive** owner of `write_session_descriptor_tool` and
`describe_session_descriptor_schema_tool` — no other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Reading session descriptors of all four supported session types
- Writing (repairing or amending) session descriptors
- Schema introspection via `describe_session_descriptor_schema_tool`
- The relationship between `SessionTypes` enum values and descriptor classes

**Does not cover:**
- Reading the `SessionData` marker file (see `/session-data`)
- Reading the frozen runtime snapshots (`MesoscopeHardwareState`, `ZaberPositions`,
  `MesoscopePositions`) (see `/session-snapshots`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering sessions (see `/project-hierarchy` for `discover_sessions_tool`)
- Initial working directory setup (see `/working-directory`)

---

## Session types and descriptor classes

| `SessionTypes` value   | Descriptor file                        | Descriptor dataclass            |
|------------------------|----------------------------------------|---------------------------------|
| `lick training`        | `lick_training_descriptor.yaml`        | `LickTrainingDescriptor`        |
| `run training`         | `run_training_descriptor.yaml`         | `RunTrainingDescriptor`         |
| `window checking`      | `window_checking_descriptor.yaml`      | `WindowCheckingDescriptor`      |
| `mesoscope experiment` | `mesoscope_experiment_descriptor.yaml` | `MesoscopeExperimentDescriptor` |

Each descriptor captures the **per-session** metadata that varies between sessions of the same type
(reward volume actually delivered, water restriction status, observed behavior summary, experimenter
notes). It is **distinct from** `SessionData`, which is the canonical session marker file.

To enumerate the canonical `SessionTypes` strings, hand off to `/session-data` for the
`list_supported_session_types_tool` call.

---

## MCP tool surface

| Tool                                      | Purpose                                                                  |
|-------------------------------------------|--------------------------------------------------------------------------|
| `read_session_descriptor_tool`            | Reads the descriptor file for a session                                  |
| `write_session_descriptor_tool`           | Writes (or repairs) a descriptor file (exclusive to this skill)          |
| `describe_session_descriptor_schema_tool` | Returns the field schema for a given session type's descriptor           |

The discovery side of "which descriptors exist in a session directory" is owned by `/session-data`
(`discover_session_descriptors_tool`). Call it as a natural share when you need to confirm a descriptor
file exists before reading or writing it.

---

## Workflows

### Repairing a corrupted or stale descriptor

1. **Verify prerequisites:**
   - MCP server connected (else `/mcp-environment-setup`).
   - Working directory set (else `/working-directory`).
2. **Read the existing descriptor:**
   ```text
   read_session_descriptor_tool(session_path="<absolute>")
   ```
3. **Inspect the schema for the session type:**
   ```text
   describe_session_descriptor_schema_tool(session_type="<type>")
   ```
4. **Build the corrected descriptor dictionary** based on the schema and the values you can recover.
5. **Confirm the planned write with the user.** Descriptor edits are irreversible without a backup.
6. **Write the corrected descriptor:**
   ```text
   write_session_descriptor_tool(session_path="<absolute>", descriptor={ ... })
   ```
7. **Re-read to verify:**
   ```text
   read_session_descriptor_tool(session_path="<absolute>")
   ```

### Amending a descriptor with post-hoc data

Use the same workflow above but mutate only the fields the user wants to amend (e.g., adding the actual
water volume delivered after a manual recount). Always confirm with the user before writing.

### Inspecting a descriptor without modification

```text
read_session_descriptor_tool(session_path="<absolute>")
```

If you need to know which descriptor file is present (when the session type is unknown), hand off to
`/session-data` to call `discover_session_descriptors_tool` first.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] describe_session_descriptor_schema_tool was called before any write
- [ ] User confirmed the planned write before it was executed
- [ ] write_session_descriptor_tool succeeded without schema errors
- [ ] read_session_descriptor_tool returned the expected content after the write
- [ ] Did not touch SessionData, hardware state, Zaber positions, or mesoscope positions from this skill
```

---

## Related skills

| Skill                    | Relationship                                                          |
|--------------------------|-----------------------------------------------------------------------|
| `/working-directory`     | Required prerequisite — must be run first                             |
| `/mcp-environment-setup` | Run first if the MCP server is not connected                          |
| `/session-data`          | Sibling — owns `SessionData` and `list_supported_session_types_tool`  |
| `/session-snapshots`     | Sibling — owns hardware state / Zaber positions / mesoscope positions |
| `/subject-metadata`      | Sibling — owns subject records                                        |
| `/project-hierarchy`     | Provides `discover_sessions_tool` to locate sessions                  |
