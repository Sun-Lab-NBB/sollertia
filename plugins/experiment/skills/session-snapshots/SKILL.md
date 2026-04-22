---
name: session-snapshots
description: >-
  Reads and writes per-session frozen position snapshots (ZaberPositions, MesoscopePositions)
  via the `sle get` MCP server. Owns the position snapshot write tools. Use when inspecting
  motor positions captured at session start, patching positions after a manual adjustment, or
  recovering a corrupted snapshot.
user-invocable: true
---

# Sollertia session position snapshots

Reads and writes the per-session frozen position snapshot YAML files (`zaber_positions.yaml` and
`mesoscope_positions.yaml`) captured at session start by `sle run`. Uses the `sle get mcp` MCP server.
This skill is the **exclusive** owner of `write_session_zaber_positions_tool` and
`write_session_mesoscope_positions_tool` — no other skill in the marketplace may call these.

The third per-session snapshot, `MesoscopeHardwareState`, is **not** covered by this skill. It lives
on the slsa MCP server (because the dataclass is shared between the acquisition runtime and the
processing pipeline) and is owned by the **assets plugin's `/session-hardware-state` skill**. Hand
off there for any read, write, or schema work on `hardware_state.yaml`.

---

## Scope

**Covers:**
- Reading and writing `ZaberPositions` (frozen Zaber motor positions at session start)
- Reading and writing `MesoscopePositions` (frozen mesoscope objective positions at session start)

**Does not cover:**
- Reading or writing `MesoscopeHardwareState` (`hardware_state.yaml`) — owned by the **assets
  plugin's `/session-hardware-state`**
- Reading the `SessionData` marker file (see assets plugin `/session-data`)
- Reading or writing session descriptors (see assets plugin `/session-descriptors`)
- Reading the frozen system or experiment configuration files at session start (those are read via
  `/system-configuration` and the assets plugin's `/experiment-configuration` `read_session_*` tools)
- Reading subject metadata (see assets plugin `/subject-metadata`)
- Live Zaber motor configuration during runtime (see this plugin's `/zaber-interface`)

---

## What is a position snapshot

When `sle run` starts a runtime acquisition session, it captures the positions of all motorized stages
(Zaber stages and the mesoscope objective) and writes them to YAML files inside the session
directory. These **frozen snapshots** are used post-hoc to:

- Reproduce the exact stage configuration that was active during the session
- Diagnose drift between the recorded positions and what the binding class expected
- Recover lost positions after a manual stage adjustment

The snapshots are written **once** at session start by `sle run` (the runtime, not this skill). This
skill exists to **read** them for inspection and to **patch** them when a snapshot file is corrupted or
out of sync with reality.

| Snapshot             | File                       | Captures                                          |
|----------------------|----------------------------|---------------------------------------------------|
| `ZaberPositions`     | `zaber_positions.yaml`     | Headbar / lickport / wheel motor positions in NVM |
| `MesoscopePositions` | `mesoscope_positions.yaml` | Mesoscope objective X/Y/Z and rotation positions  |

---

## MCP tool surface

| Tool                                          | MCP server   | Purpose                                                              |
|-----------------------------------------------|--------------|----------------------------------------------------------------------|
| `read_session_zaber_positions_tool`           | `sle get mcp` | Reads `ZaberPositions` for a session                                 |
| `write_session_zaber_positions_tool`          | `sle get mcp` | Writes (patches) `ZaberPositions` (exclusive to this skill)          |
| `read_session_mesoscope_positions_tool`       | `sle get mcp` | Reads `MesoscopePositions` for a session                             |
| `write_session_mesoscope_positions_tool`      | `sle get mcp` | Writes (patches) `MesoscopePositions` (exclusive to this skill)      |

For the `MesoscopeHardwareState` read/write/describe trio (`read_session_hardware_state_tool`,
`write_session_hardware_state_tool`, `describe_session_hardware_state_schema_tool`), hand off to the
**assets plugin's `/session-hardware-state` skill**.

---

## Workflows

### Inspecting position snapshots for a session

1. **Verify prerequisites:** `sle get mcp` (sollertia-experiment) is connected. If not, hand
   off to `/experiment-mcp-environment-setup`.
2. **Read the snapshots:**
   ```text
   read_session_zaber_positions_tool(session_path="<absolute>")
   read_session_mesoscope_positions_tool(session_path="<absolute>")
   ```
3. **Report to the user.** Optionally cross-reference with the live system configuration via
   `/system-configuration` to identify drift, or hand off to the assets plugin's
   `/session-hardware-state` to also pull the hardware state snapshot.

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

If a position snapshot fails to parse, the read tool returns an error. To recover:

1. Read the other position snapshot in the same session to reconstruct context.
2. Hand off to the assets plugin's `/session-hardware-state` to read `MesoscopeHardwareState` for
   additional rig context.
3. Read the frozen system configuration via `/system-configuration`
   (`read_session_system_configuration_tool`) to recover hardware ID assignments.
4. Hand the recovery proposal to the user before writing a replacement snapshot.

### Coordinated multi-snapshot patching

If you need to patch position snapshots **and** the hardware state snapshot together (e.g., after
replacing the entire acquisition rig), patch the position snapshots from this skill and hand off to
the assets plugin's `/session-hardware-state` for the hardware state write. Do not attempt to call
`write_session_hardware_state_tool` from this skill — that ownership has moved.

---

## Verification checklist

```text
- [ ] sle get mcp (sollertia-experiment) is connected
- [ ] User confirmed the planned snapshot patch (snapshots are historical records)
- [ ] write_session_*_positions_tool succeeded without errors
- [ ] read_session_*_positions_tool returned the expected content after every write
- [ ] Did not call write_session_hardware_state_tool from this skill — handed off to
      /session-hardware-state instead
- [ ] Did not touch SessionData, descriptors, or subject metadata from this skill
```

---

## Related skills

| Skill                                              | Relationship                                                              |
|----------------------------------------------------|---------------------------------------------------------------------------|
| this plugin `/experiment-mcp-environment-setup`    | Run first if `sle get mcp` is not connected                                |
| assets plugin `/session-hardware-state`            | Sibling — owns `MesoscopeHardwareState` (the third per-session snapshot)  |
| assets plugin `/session-data`                      | Owns the `SessionData` marker file                                        |
| assets plugin `/session-descriptors`               | Owns the per-session descriptor files                                     |
| this plugin `/system-configuration`                | Provides `read_session_system_configuration_tool` for cross-reference     |
| assets plugin `/experiment-configuration`          | Provides `read_session_experiment_configuration_tool` for cross-reference |
| this plugin `/zaber-interface`                     | Live Zaber motor configuration during runtime — does not touch snapshots  |
