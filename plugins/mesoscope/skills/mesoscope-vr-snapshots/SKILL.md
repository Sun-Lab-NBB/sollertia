---
name: mesoscope-vr-snapshots
description: >-
  Reads and writes the Mesoscope-VR per-session frozen position snapshots (ZaberPositions,
  MesoscopePositions) via the `sle mcp` server. Owns the position snapshot write tools. Use when
  inspecting motor positions recorded during a session, patching positions after a manual
  adjustment, or recovering a corrupted snapshot.
user-invocable: false
---

# Sollertia Mesoscope-VR position snapshots

Reads and writes the per-session frozen position snapshot YAML files (`zaber_positions.yaml` and
`mesoscope_positions.yaml`) written by the runtime during a `sle mesoscope run <session-type>` session.
Uses the `sle mcp` server. This skill is the **exclusive** owner of `write_session_zaber_positions_tool`
and `write_session_mesoscope_positions_tool` — no other skill in the marketplace may call these.

These position snapshots are specific to the Mesoscope-VR acquisition system: both the snapshot
schemas and the MCP tools that read and write them are bound to Mesoscope-VR hardware (the Zaber
motor groups and the mesoscope objective), with no `acquisition_system` schema selector. The
*live* Zaber motor interface is platform-general and reusable across acquisition systems (see
`experiment:zaber-interface`); this skill manages the *frozen per-session records*, not the reusable
interface.

The third per-session snapshot, `MesoscopeHardwareState`, is **not** covered by this skill. It lives
on the slsa MCP server (because the dataclass is shared between the acquisition runtime and the
processing pipeline) and is owned by the **`assets:session-hardware-state` skill**. Hand
off there for any read, write, or schema work on `hardware_state.yaml`.

---

## Scope

**Covers:**
- Reading and writing `ZaberPositions` (frozen Zaber motor positions at session start)
- Reading and writing `MesoscopePositions` (frozen mesoscope objective positions at session start)

**Does not cover:**
- Reading or writing `MesoscopeHardwareState` (`hardware_state.yaml`) — owned by the **assets
  plugin's `assets:session-hardware-state`**
- Reading the `SessionData` marker file (see `assets:session-data`)
- Reading or writing session descriptors (see `assets:session-descriptors`)
- Reading the frozen system or experiment configuration files at session start (those are read via
  `/mesoscope-vr`'s `read_session_system_configuration_tool` and the assets plugin's
  `assets:experiment-configuration` `read_experiment_configuration_tool`)
- Reading subject metadata (see `assets:data-assets`)
- Live Zaber motor configuration during runtime (see `experiment:zaber-interface`)

---

## What is a position snapshot

During a `sle mesoscope run <session-type>` acquisition session, the runtime records the positions of the
motorized stages and writes them to YAML files inside the session directory. The Zaber positions are
queried automatically from the motors; the mesoscope objective positions are queried from the ScanImage
software over MQTT, except for the red-dot alignment Z position (`red_dot_alignment_z`), which the
ScanImage software cannot report and which the operator enters through a terminal prompt that defaults to
the previous runtime's value. These **frozen snapshots** are used post-hoc to:

- Reproduce the exact stage configuration that was active during the session
- Diagnose drift between the recorded positions and what the binding class expected
- Recover lost positions after a manual stage adjustment

The runtime writes the snapshots (not this skill): the Zaber snapshot is captured at session start and
refreshed at session stop, and the mesoscope objective snapshot is recorded at session stop. This skill
exists to **read** them for inspection and to **patch** them when a snapshot file is corrupted or out of
sync with reality.

| Snapshot             | File                       | Captures                                                         |
|----------------------|----------------------------|------------------------------------------------------------------|
| `ZaberPositions`     | `zaber_positions.yaml`     | Headbar / lickport / wheel motor positions in native motor units |
| `MesoscopePositions` | `mesoscope_positions.yaml` | Mesoscope objective and ScanImage virtual axes, plus laser power |

---

## MCP tool surface

| Tool                                     | MCP server | Purpose                                                         |
|------------------------------------------|------------|-----------------------------------------------------------------|
| `read_session_zaber_positions_tool`      | `sle mcp`  | Reads `ZaberPositions` for a session                            |
| `write_session_zaber_positions_tool`     | `sle mcp`  | Writes (patches) `ZaberPositions` (exclusive to this skill)     |
| `read_session_mesoscope_positions_tool`  | `sle mcp`  | Reads `MesoscopePositions` for a session                        |
| `write_session_mesoscope_positions_tool` | `sle mcp`  | Writes (patches) `MesoscopePositions` (exclusive to this skill) |

Both write tools take the snapshot payload as the `positions_payload` argument and accept a keyword-only
`overwrite` flag (default `True`); pass `overwrite=False` to refuse replacing an existing snapshot file.

For the `MesoscopeHardwareState` read/write/describe trio (`read_session_hardware_state_tool`,
`write_session_hardware_state_tool`, `describe_session_hardware_state_schema_tool`), hand off to the
**`assets:session-hardware-state` skill**.

---

## Workflows

### Inspecting position snapshots for a session

1. **Verify prerequisites:** `sle mcp` (sollertia-experiment) is connected. If not, hand
   off to `experiment:experiment-mcp-environment-setup`.
2. **Read the snapshots:**
   ```text
   read_session_zaber_positions_tool(session_path="<absolute>")
   read_session_mesoscope_positions_tool(session_path="<absolute>")
   ```
3. **Report to the user.** Optionally cross-reference with the live system configuration via
   `/mesoscope-vr` to identify drift, or hand off to the assets plugin's
   `assets:session-hardware-state` to also pull the hardware state snapshot.

### Rescuing a bad auto-generated `ZaberPositions` snapshot

A session-time failure — a motor fault, an early abort, or a similar interruption — can leave the
runtime's auto-generated `zaber_positions.yaml` holding wrong or partial motor positions. Use this
workflow to overwrite that snapshot with the positions the session would have recorded had it
completed normally, restoring a faithful frozen record of the stage configuration for downstream use.
The write tool targets the session's `raw_data` copy only; the per-animal `persistent_data` copy that
seeds the next runtime is a separate file and is not modified here.

1. **Verify the user actually wants to modify the frozen snapshot.** Patching a snapshot rewrites
   history — the original captured state is lost.
2. **Read the current positions:**
   ```text
   read_session_zaber_positions_tool(session_path="<absolute>")
   ```
3. **Build the corrected dictionary** with the positions the session should have recorded —
   typically the same animal's last good snapshot (see [Recovering a corrupted
   snapshot](#recovering-a-corrupted-snapshot) for sourcing it from an adjacent session or the
   `persistent_data` copy).
4. **Confirm with the user before writing.**
5. **Write the corrected positions:**
   ```text
   write_session_zaber_positions_tool(session_path="<absolute>", positions_payload={ ... })
   ```
6. **Re-read to verify.**

### Recovering a corrupted snapshot

If a position snapshot fails to parse, the read tool returns an error. The two position snapshots
cover disjoint subsystems (Zaber stages vs. the Mesoscope objective) and share no fields, so the
*other* snapshot in the same session cannot supply the corrupted one's values. Recover from the same
snapshot **type** elsewhere: the runtime mirrors each snapshot to the animal's `persistent_data`
directory (overwriting it on every completed session) and re-seeds the next runtime from it, so the
same animal's positions drift little between sessions.

1. **Recover the values from the same-type snapshot of the same animal.** Read an adjacent same-animal
   session's snapshot of the same type with `read_session_zaber_positions_tool` /
   `read_session_mesoscope_positions_tool` (pass that session's `session_path`), or read the animal's
   persistent-directory copy directly — `<animal>/persistent_data/zaber_positions.yaml` or
   `mesoscope_positions.yaml`, the same schema as the session snapshot. The persistent copy reflects
   the most recent completed session: an exact match when the corrupted snapshot is the latest,
   otherwise a close approximation.
2. **Pull rig context to validate a candidate** (not to recover positions). Hand off to the assets
   plugin's `assets:session-hardware-state` for `MesoscopeHardwareState`, and read the frozen system
   configuration via `/mesoscope-vr` (`read_session_system_configuration_tool`) for hardware-ID
   assignments and per-motor calibration — use these to confirm a candidate snapshot matches the rig
   that ran the session.
3. Hand the recovery proposal to the user before writing a replacement snapshot.

### Coordinated multi-snapshot patching

If you need to patch position snapshots **and** the hardware state snapshot together (e.g., after
replacing the entire acquisition rig), patch the position snapshots from this skill and hand off to
`assets:session-hardware-state` for the hardware state write. Do not attempt to call
`write_session_hardware_state_tool` from this skill — that ownership has moved.

---

## Related skills

| Skill                                     | Relationship                                                                                      |
|-------------------------------------------|---------------------------------------------------------------------------------------------------|
| `experiment:experiment-mcp-environment-setup`       | Run first if `sle mcp` is not connected                                                           |
| `assets:session-hardware-state`   | Sibling — owns `MesoscopeHardwareState` (the third per-session snapshot)                          |
| `assets:session-data`             | Owns the `SessionData` marker file                                                                |
| `assets:session-descriptors`      | Owns the per-session descriptor files                                                             |
| `/mesoscope-vr`                           | Provides `read_session_system_configuration_tool` for cross-reference                             |
| `assets:experiment-configuration` | Provides `read_experiment_configuration_tool` for cross-reference (accepts session snapshot path) |
| `experiment:zaber-interface`                        | Live Zaber motor configuration during runtime — does not touch snapshots                          |

---

## Verification checklist

```text
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] User confirmed the planned snapshot patch (snapshots are historical records)
- [ ] write_session_*_positions_tool succeeded without errors
- [ ] read_session_*_positions_tool returned the expected content after every write
- [ ] Did not call write_session_hardware_state_tool from this skill — handed off to
      assets:session-hardware-state instead
- [ ] Did not touch SessionData, descriptors, or subject metadata from this skill
```
