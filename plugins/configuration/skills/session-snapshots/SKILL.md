---
name: configure-session-snapshots
description: >-
  Reads and writes the per-session frozen runtime snapshot YAML files (MesoscopeHardwareState,
  ZaberPositions, MesoscopePositions) via the sl-configure MCP server. Owns the snapshot write tools.
  Use when repairing a corrupted snapshot, patching positions after manual stage adjustment, or
  inspecting the runtime context that was captured when a session started.
user-invocable: true
---

# Sollertia session snapshots

Reads and writes the per-session frozen runtime snapshot YAML files captured at session start. This
skill is the **exclusive** owner of the three snapshot write tools — no other skill in the marketplace
may call `write_session_hardware_state_tool`, `write_session_zaber_positions_tool`, or
`write_session_mesoscope_positions_tool`.

---

## Scope

**Covers:**
- Reading and writing `MesoscopeHardwareState` (frozen hardware state at session start)
- Reading and writing `ZaberPositions` (frozen Zaber motor positions at session start)
- Reading and writing `MesoscopePositions` (frozen mesoscope objective positions at session start)

**Does not cover:**
- Reading the `SessionData` marker file (see `/session-data`)
- Reading or writing session descriptors (see `/session-descriptors`)
- Reading the frozen system or experiment configuration files at session start (those are read via
  `/system-configuration` and `/experiment-configuration` `read_session_*` tools)
- Reading subject metadata (see `/subject-metadata`)
- Live Zaber motor configuration during runtime (see the experiment plugin's `/zaber-interface`)
- Initial working directory setup (see `/working-directory`)

---

## What is a session snapshot

When `sl-run` starts a runtime acquisition session, it captures the current state of the acquisition
hardware and the positions of all motorized stages and writes them to YAML files inside the session
directory. These **frozen snapshots** are used post-hoc to:

- Reproduce the exact hardware configuration that was active during the session
- Diagnose drift between the recorded state and what the binding class expected
- Recover lost positions after a manual stage adjustment

The snapshots are written **once** at session start by `sl-run` (the runtime, not this skill). This
skill exists to **read** them for inspection and to **patch** them when a snapshot file is corrupted or
out of sync with reality.

| Snapshot                  | File                          | Captures                                              |
|---------------------------|-------------------------------|-------------------------------------------------------|
| `MesoscopeHardwareState`  | `mesoscope_hardware_state.yaml` | Microcontroller IDs, camera indices, MQTT topics    |
| `ZaberPositions`          | `zaber_positions.yaml`        | Headbar / lickport / wheel motor positions in NVM    |
| `MesoscopePositions`      | `mesoscope_positions.yaml`    | Mesoscope objective X/Y/Z and rotation positions     |

---

## MCP tool surface

| Tool                                          | Purpose                                                              |
|-----------------------------------------------|----------------------------------------------------------------------|
| `read_session_hardware_state_tool`            | Reads `MesoscopeHardwareState` for a session                         |
| `write_session_hardware_state_tool`           | Writes (repairs) `MesoscopeHardwareState` (exclusive to this skill)  |
| `read_session_zaber_positions_tool`           | Reads `ZaberPositions` for a session                                 |
| `write_session_zaber_positions_tool`          | Writes (patches) `ZaberPositions` (exclusive to this skill)          |
| `read_session_mesoscope_positions_tool`       | Reads `MesoscopePositions` for a session                             |
| `write_session_mesoscope_positions_tool`      | Writes (patches) `MesoscopePositions` (exclusive to this skill)      |

---

## Workflows

### Inspecting the runtime context for a session

1. **Verify prerequisites:**
   - MCP server connected (else `/mcp-environment-setup`).
   - Working directory set (else `/working-directory`).
2. **Read the snapshots:**
   ```text
   read_session_hardware_state_tool(session_path="<absolute>")
   read_session_zaber_positions_tool(session_path="<absolute>")
   read_session_mesoscope_positions_tool(session_path="<absolute>")
   ```
3. **Report to the user.** Optionally cross-reference with the live system configuration via
   `/system-configuration` to identify drift.

### Patching `ZaberPositions` after a manual stage adjustment

Use this workflow when a user has physically moved a Zaber stage and the recorded positions no longer
match reality.

1. **Verify the user actually wants to modify the frozen snapshot.** Patching a snapshot rewrites
   history — the original captured state is lost.
2. **Read the current positions:**
   ```text
   read_session_zaber_positions_tool(session_path="<absolute>")
   ```
3. **Build the corrected dictionary** with the new motor positions.
4. **Confirm with the user before writing.**
5. **Write the corrected positions:**
   ```text
   write_session_zaber_positions_tool(session_path="<absolute>", positions={ ... })
   ```
6. **Re-read to verify.**

### Recovering a corrupted snapshot

If a snapshot file fails to parse, the snapshot tool returns an error. To recover:

1. Read the other two snapshots in the same session to reconstruct context.
2. Read the frozen system configuration via `/system-configuration` (`read_session_system_configuration_tool`)
   to recover hardware ID assignments.
3. Hand the recovery proposal to the user before writing a replacement snapshot.

### Cross-skill snapshot patching

If you need to patch all three snapshots together (e.g., after replacing the entire acquisition rig),
call the three write tools in sequence within this skill. Do not split the work across skills — all
three writes are owned here.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] User confirmed the planned snapshot patch (snapshots are historical records)
- [ ] write_session_*_tool succeeded without errors
- [ ] read_session_*_tool returned the expected content after every write
- [ ] Did not touch SessionData, descriptors, or subject metadata from this skill
```

---

## Related skills

| Skill                                  | Relationship                                                              |
|----------------------------------------|---------------------------------------------------------------------------|
| `/working-directory`                   | Required prerequisite — must be run first                                 |
| `/mcp-environment-setup`               | Run first if the MCP server is not connected                              |
| `/session-data`                        | Sibling — owns the `SessionData` marker file                              |
| `/session-descriptors`                 | Sibling — owns the per-session descriptor files                           |
| `/system-configuration`                | Provides `read_session_system_configuration_tool` for cross-reference     |
| `/experiment-configuration`            | Provides `read_session_experiment_configuration_tool` for cross-reference |
| experiment plugin `/zaber-interface`   | Live Zaber motor configuration during runtime — does not touch snapshots  |
