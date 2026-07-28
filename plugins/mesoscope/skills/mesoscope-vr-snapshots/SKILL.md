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

Reads and writes the per-session frozen position snapshot YAML files written by the runtime during a
`sle mesoscope run <session-type>` session. Every session type writes `zaber_positions.yaml`, and the
`experiment` and `window-checking` session types additionally write `mesoscope_positions.yaml`.
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
- Reading and writing `ZaberPositions` (frozen Zaber motor positions recorded near the end of the session)
- Reading and writing `MesoscopePositions` (frozen mesoscope objective positions recorded at the end of the session)

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

The runtime writes the snapshots (not this skill). Every session type ends up with exactly one
`zaber_positions.yaml`, written once the session is marked initialized. `generate_zaber_snapshot` returns early
while the `nk.bin` marker is still present, so for the `lick-training`, `run-training`, and `experiment` session
types, which drive the `MesoscopeVRSystem` controller, the surviving write is the one in `stop()`. The
`window-checking` session type runs its own sequence and writes the snapshot after it marks the session
initialized and before it finalizes the session descriptor. The `experiment` and `window-checking` session types
seed a precursor mesoscope objective snapshot at session start and record the queried objective positions at the
end of the session, so `lick-training` and `run-training` session directories carry only `zaber_positions.yaml`.
This skill exists to **read** them for inspection and to **patch** them when a snapshot file is corrupted or out
of sync with reality.

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

The `sle mcp` server exposes no describe-schema tool for either snapshot, so the field rosters below are the
reference. Each write tool round-trips `positions_payload` through its dataclass, and every field of both
dataclasses carries a default. A missing or misspelled key therefore validates without error and writes the
default value for that position. You MUST read the current snapshot first and send the complete field roster
on every write.

`ZaberPositions` (`zaber_positions.yaml`) holds seven `int` fields, each an absolute position in native motor
units:

| Field           | Captures                                     |
|-----------------|----------------------------------------------|
| `headbar_z`     | HeadBar z-axis motor position                |
| `headbar_pitch` | HeadBar pitch-axis motor position            |
| `headbar_roll`  | HeadBar roll-axis motor position             |
| `lickport_z`    | LickPort z-axis motor position               |
| `lickport_y`    | LickPort y-axis motor position               |
| `lickport_x`    | LickPort x-axis motor position               |
| `wheel_x`       | Running wheel platform x-axis motor position |

`MesoscopePositions` (`mesoscope_positions.yaml`) holds nine `float` fields:

| Field                 | Captures                                                                 |
|-----------------------|--------------------------------------------------------------------------|
| `mesoscope_x`         | Objective X-axis position, in micrometers                                |
| `mesoscope_y`         | Objective Y-axis position, in micrometers                                |
| `mesoscope_roll`      | Objective Roll-axis position, in degrees                                 |
| `mesoscope_z`         | Objective Z-axis position, in micrometers                                |
| `mesoscope_fast_z`    | ScanImage FastZ virtual Z-axis position, in micrometers                  |
| `mesoscope_tip`       | ScanImage Tip position, in degrees                                       |
| `mesoscope_tilt`      | ScanImage Tilt position, in degrees                                      |
| `laser_power_mw`      | Laser excitation power at the sample, in milliwatts                      |
| `red_dot_alignment_z` | Objective Z-axis position used for the red-dot alignment, in micrometers |

For the `MesoscopeHardwareState` read/write/describe trio (`read_session_hardware_state_tool`,
`write_session_hardware_state_tool`, `describe_session_hardware_state_schema_tool`), hand off to the
**`assets:session-hardware-state` skill**.

---

## Mesoscope-VR raw-data layout contract

Descriptors, hardware state, trial types, experiment configuration, and **raw-data layout** form a universal
per-system contract: every Sollertia acquisition system is expected to define its own concrete instance. The
generic registry dispatch and the create/write/validate/describe tooling are system-agnostic and live in the
assets plugin (see `assets:session-data`). This section documents only Mesoscope-VR's **concrete instance** of
the raw-data layout contract — the system-specific filenames, subdirectories, and path-resolution fields keyed by
`SYSTEM_RAW_DATA_REGISTRY[AcquisitionSystems.MESOSCOPE_VR] -> MesoscopeRawData`.

Unlike the descriptor / hardware-state / experiment-configuration contracts (which register a `YamlConfig`
describe/read/write trio), the raw-data contract registers a **path-resolution dataclass**. `MesoscopeRawData` does
not read or write any file: it only resolves the absolute on-disk locations of the system-specific raw assets under
a session's `raw_data` directory. The position snapshots that this skill reads and writes are two of those assets.
The snapshot MCP tools reach those same two files on their own, joining module-local filename constants onto the
resolved session root, so they duplicate the canonical filenames that this dataclass resolves.

`MesoscopeRawDataFiles` enumerates the canonical filenames at the root of `raw_data` written exclusively by the
Mesoscope-VR acquisition system:

| Member                | Value                      | Captures                                                        |
|-----------------------|----------------------------|-----------------------------------------------------------------|
| `ZABER_POSITIONS`     | `zaber_positions.yaml`     | Zaber motor position snapshot written near the session's end    |
| `MESOSCOPE_POSITIONS` | `mesoscope_positions.yaml` | Objective position snapshot from experiment and window-checking |
| `WINDOW_SCREENSHOT`   | `window_screenshot.png`    | Cranial imaging window screenshot captured at session start     |

`MesoscopeDirectories` enumerates the canonical subdirectory names under `raw_data` written exclusively by the
Mesoscope-VR acquisition system:

| Member           | Value            | Captures                                                                      |
|------------------|------------------|-------------------------------------------------------------------------------|
| `MESOSCOPE_DATA` | `mesoscope_data` | LERC-compressed TIFF stacks and acquisition metadata written by preprocessing |

`MesoscopeRawData.build(root)` is the single source of truth for the enum-to-field mapping. `SessionData`
constructs the instance when the session's acquisition system is `MESOSCOPE_VR`, resolving each field as
`root.joinpath(<enum value>)`:

| Field                      | Resolves to                         | Source enum member                          |
|----------------------------|-------------------------------------|---------------------------------------------|
| `zaber_positions_path`     | `raw_data/zaber_positions.yaml`     | `MesoscopeRawDataFiles.ZABER_POSITIONS`     |
| `mesoscope_positions_path` | `raw_data/mesoscope_positions.yaml` | `MesoscopeRawDataFiles.MESOSCOPE_POSITIONS` |
| `window_screenshot_path`   | `raw_data/window_screenshot.png`    | `MesoscopeRawDataFiles.WINDOW_SCREENSHOT`   |
| `mesoscope_data_path`      | `raw_data/mesoscope_data/`          | `MesoscopeDirectories.MESOSCOPE_DATA`       |

`zaber_positions_path` and `mesoscope_positions_path` are the on-disk targets the position-snapshot tools read and
write (this skill); `window_screenshot_path` and `mesoscope_data_path` are produced by the runtime and
preprocessing and are not written by this skill. For the generic registry dispatch and the system-agnostic tools
that operate over the raw-data layout, hand off to `assets:session-data`.

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
   Only the `experiment` and `window-checking` session types write `mesoscope_positions.yaml`. For a
   `lick-training` or `run-training` session, read the Zaber snapshot alone, because the mesoscope read returns a
   file-not-found error that reflects the expected layout.
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
   `persistent_data` copy). Carry all seven `ZaberPositions` fields listed under
   [MCP tool surface](#mcp-tool-surface), since an omitted key silently writes its default.
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

| Skill                                         | Relationship                                                                                                   |
|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| `experiment:experiment-mcp-environment-setup` | Run first if `sle mcp` is not connected                                                                        |
| `assets:session-hardware-state`               | Sibling — owns `MesoscopeHardwareState` (the third per-session snapshot)                                       |
| `assets:session-data`                         | Owns the `SessionData` marker file; dispatches the generic `SYSTEM_RAW_DATA_REGISTRY` (mesoscope tree -> here) |
| `assets:session-descriptors`                  | Owns the per-session descriptor files                                                                          |
| `/mesoscope-vr`                               | Provides `read_session_system_configuration_tool` for cross-reference                                          |
| `assets:experiment-configuration`             | Provides `read_experiment_configuration_tool` for cross-reference (accepts session snapshot path)              |
| `experiment:zaber-interface`                  | Live Zaber motor configuration during runtime — does not touch snapshots                                       |

---

## Verification checklist

```text
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] User confirmed the planned snapshot patch (snapshots are historical records)
- [ ] Every write payload carried the complete field roster for its snapshot type
- [ ] write_session_*_positions_tool succeeded without errors
- [ ] read_session_*_positions_tool returned the expected content after every write
- [ ] Did not call write_session_hardware_state_tool from this skill — handed off to
      assets:session-hardware-state instead
- [ ] Did not touch SessionData, descriptors, or subject metadata from this skill
```
