---
name: mesoscope-vr-snapshots
description: >-
  Reads and writes the Mesoscope-VR per-session frozen position snapshots (ZaberPositions, MesoscopePositions) via
  the `sle mcp` server, and documents the Mesoscope-VR raw-data layout contract (MesoscopeRawDataFiles,
  MesoscopeDirectories, MesoscopeRawData.build). Owns the position snapshot write tools. Use when inspecting motor
  positions recorded during a session, patching positions after a manual adjustment, recovering a corrupted
  snapshot, or resolving a Mesoscope-VR raw-data path.
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

The third per-session snapshot, `MesoscopeHardwareState`, is **not** covered by this skill. It lives on the slsa
MCP server, because the dataclass is shared between the acquisition runtime and the processing pipeline. Hand off to
`assets:session-hardware-state` for the generic read, write, and describe tooling over `hardware_state.yaml`, and to
`mesoscope:mesoscope-vr-session-schema` for the field-level Mesoscope-VR schema those tools operate on.

---

## Scope

**Covers:**
- Reading and writing `ZaberPositions` (frozen Zaber motor positions recorded near the end of the session)
- Reading and writing `MesoscopePositions` (frozen mesoscope objective positions recorded at the end of the session)
- The Mesoscope-VR raw-data layout contract (`MesoscopeRawDataFiles`, `MesoscopeDirectories`, and the
  `MesoscopeRawData.build` field resolution), which this skill owns exclusively and `assets:session-data` defers to

**Does not cover:**
- Reading or writing `MesoscopeHardwareState` (`hardware_state.yaml`), which `assets:session-hardware-state` owns,
  and its field-level schema, which `mesoscope:mesoscope-vr-session-schema` owns
- Reading the `SessionData` marker file (see `assets:session-data`)
- Reading or writing session descriptors (see `assets:session-descriptors`)
- Reading the frozen system or experiment configuration files at session start (those are read via
  `mesoscope:mesoscope-vr`'s `read_session_system_configuration_tool` and the assets plugin's
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

The runtime writes the snapshots (not this skill). `generate_zaber_snapshot` returns early when at least one of the
managed Zaber motor groups is disconnected, and while the `nk.bin` marker is still present, so a session that trips
either gate ends with no `zaber_positions.yaml` at all rather than a partial one. For the `lick-training`,
`run-training`, and `experiment` session types, which drive the `MesoscopeVRSystem` controller, the controller writes
the snapshot once, during `stop()`, after `mark_runtime_initialized()` has cleared `nk.bin`. An aborted initialization
leaves `nk.bin` in place, so the write is skipped and no file is produced. The `window-checking` session type runs its
own sequence and writes the snapshot after it marks the session initialized and before it finalizes the session
descriptor. Both write sites run before the motors are returned to their parking position, so the snapshot records the
pose the session ended in rather than the parked pose. The `experiment` and `window-checking` session types
seed a precursor mesoscope objective snapshot at session start and record the queried objective positions at the
end of the session, so `lick-training` and `run-training` session directories carry only `zaber_positions.yaml`.
This skill exists to **read** them for inspection and to **patch** them when a snapshot file is corrupted or out
of sync with reality.

| Snapshot             | File                       | Captures                                                         |
|----------------------|----------------------------|------------------------------------------------------------------|
| `ZaberPositions`     | `zaber_positions.yaml`     | Headbar / lickport / wheel positions, captured before parking    |
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

Every tool here takes `session_path`, which accepts either the session root or its `raw_data` subdirectory. Any other
existing path fails with `Could not locate the raw_data directory under <path>`, and a path that does not exist fails
with `Session path does not exist: <path>`. Because the resolution is anchored on `raw_data`, these tools cannot reach
the per-animal `persistent_data` copies of the snapshots that the recovery workflow draws on.

The `sle mcp` server exposes no describe-schema tool for either snapshot, so the field rosters below are the reference.
Each write tool round-trips `positions_payload` through its dataclass. The `sle mcp` write helper is stricter than the
`slsa mcp` one, so the permissive write semantics the assets skills document do not transfer here:

| Payload defect      | Outcome                                                                                       |
|---------------------|-----------------------------------------------------------------------------------------------|
| Omitted key         | Accepted silently, written at the field's dataclass default (`0` or `0.0`)                    |
| Misspelled key      | Rejected with `Validation failed for <Class>: the payload carries the unknown key(s) <name>.` |
| Wrongly typed value | Rejected with `Validation failed for <Class>: <field> is <type>, expected <annotation>.`      |

Because the omitted-key case is the silent one, you MUST read the current snapshot first and send the complete field
roster on every write.

On the read side, both read tools detect only file-level corruption: a missing file, malformed YAML, or a document that
does not load as its dataclass. A snapshot whose keys were dropped or misspelled loads cleanly, with `0` or `0.0`
standing in for every field the file does not name, so a successful read is not evidence that the file is intact. You
MUST treat an all-zero or partially zero read as suspect and check it against the animal's `persistent_data` copy
before using it.

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

The runtime floors every `MesoscopePositions` field to at most three decimal places before writing, discarding the
spurious sub-micrometer and sub-millidegree precision the ScanImage software reports. A patched payload should carry at
most three decimals to stay consistent with the runtime-written records.

For the `MesoscopeHardwareState` read/write/describe trio (`read_session_hardware_state_tool`,
`write_session_hardware_state_tool`, `describe_session_hardware_state_schema_tool`), hand off to
`assets:session-hardware-state`. For the fields those tools carry, hand off to
`mesoscope:mesoscope-vr-session-schema`.

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
| `ZABER_POSITIONS`     | `zaber_positions.yaml`     | Position snapshot written at session end, before motor parking  |
| `MESOSCOPE_POSITIONS` | `mesoscope_positions.yaml` | Objective position snapshot from experiment and window-checking |
| `WINDOW_SCREENSHOT`   | `window_screenshot.png`    | Cranial imaging window screenshot captured at session start     |

`window_screenshot.png` arrives through `setup_mesoscope`, which blocks the operator until exactly one `.png` file sits
in the ScanImagePC `mesodata` directory, moves that file to the session's `window_screenshot_path`, and copies it to
the animal's persistent directory. `setup_mesoscope` runs only for `experiment` and `window-checking` sessions, so
`lick-training` and `run-training` sessions carry no `window_screenshot.png`. That absence is the expected layout for
those session types rather than a missing file.

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

`MesoscopeRawData` is declared `@dataclass(frozen=True, slots=True)`, so assigning to a resolved path raises
`FrozenInstanceError`. Rebuild the instance with `MesoscopeRawData.build(root=...)` instead of mutating one.

This section is also the reference an extender copies when authoring a new acquisition system's `<system>/raw_data.py`.
See `assets:library-extension` for the registration contract that file must satisfy.

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
   `mesoscope:mesoscope-vr` to identify drift, or hand off to `assets:session-hardware-state` to also pull the hardware
   state snapshot. Only the `MesoscopeVRSystem` controller writes `hardware_state.yaml`, so that handoff applies to
   `lick-training`, `run-training`, and `experiment` sessions. A `window-checking` session runs its own sequence and
   carries no `hardware_state.yaml` at all.

### Rescuing a bad auto-generated `ZaberPositions` snapshot

A snapshot that exists is always complete, because all seven `ZaberPositions` fields are built in one constructor call
from a single query pass. A disconnected motor group or a session still carrying `nk.bin` yields no snapshot file at
all rather than a truncated one. What a session-time fault does produce is a complete snapshot recording the pose the
motors reached after the fault, which is a faithful record of the wrong stage configuration. Use this workflow to
overwrite such a snapshot with the positions the session would have recorded had it completed normally, restoring a
faithful frozen record of the stage configuration for downstream use. The write tool targets the session's `raw_data`
copy only. The per-animal `persistent_data` copy that seeds the next runtime is a separate file and is not modified
here.

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
   otherwise a close approximation. Reject an all-zero persistent `mesoscope_positions.yaml` as a recovery source. A
   window-checking session seeds that file with all-zero defaults at session start for an animal it has never imaged,
   so an animal whose first window-checking session never reached teardown carries defaults rather than a recorded
   pose.
2. **Pull rig context to validate a candidate** (not to recover positions). Hand off to
   `assets:session-hardware-state` for `MesoscopeHardwareState`, and read the frozen system configuration via
   `mesoscope:mesoscope-vr` (`read_session_system_configuration_tool`) for the three Zaber USB port strings it carries:
   `headbar_port`, `lickport_port`, and `wheel_port`. The frozen configuration holds no per-motor calibration. The
   park, maintenance, and mount positions, the motion limits, and the device labels live in the motors' non-volatile
   memory and are read through `experiment:zaber-interface` and its `get_zaber_device_settings_tool`. Use both sources
   to confirm a candidate snapshot matches the rig that ran the session.
3. Hand the recovery proposal to the user before writing a replacement snapshot.

### The per-animal persistent-data layout

The runtime caches four files per animal so that each new session can seed itself from the previous one. They live in
the animal's persistent directory, which is `AnimalData.persistent_data_path` and is owned by
`assets:project-hierarchy`:

| File                             | Written when                                                            |
|----------------------------------|-------------------------------------------------------------------------|
| `zaber_positions.yaml`           | Session end, together with the session's `raw_data` copy                |
| `mesoscope_positions.yaml`       | Session end, for `experiment` and `window-checking` sessions only       |
| `window_screenshot.png`          | Session start, after the screenshot moves into the session's `raw_data` |
| `<session-type>_descriptor.yaml` | Session end, copied from the session's finalized descriptor             |

The descriptor filename varies with the session type, and `mesoscope:mesoscope-vr-session-schema` carries the four
exact names. **No MCP tool reaches any of these files.** The snapshot tools resolve their targets under a session's
`raw_data` directory, and neither server exposes a tool that takes a persistent-directory path. You MUST read a
persistent copy with a plain filesystem read.

### Coordinated multi-snapshot patching

If you need to patch position snapshots **and** the hardware state snapshot together (e.g., after
replacing the entire acquisition rig), patch the position snapshots from this skill and hand off to
`assets:session-hardware-state` for the hardware state write. `assets:session-hardware-state` owns
`write_session_hardware_state_tool`.

---

## Related skills

| Skill                                         | Relationship                                                     |
|-----------------------------------------------|------------------------------------------------------------------|
| `experiment:experiment-mcp-environment-setup` | Run first if `sle mcp` is not connected                          |
| `assets:session-hardware-state`               | Sibling: owns `MesoscopeHardwareState`, the third snapshot       |
| `mesoscope:mesoscope-vr-session-schema`       | Owns the field-level descriptor and hardware-state schema        |
| `assets:session-data`                         | Owns the `SessionData` marker file and the raw-data dispatch     |
| `assets:session-descriptors`                  | Owns the per-session descriptor files                            |
| `assets:project-hierarchy`                    | Owns `AnimalData.persistent_data_path`, the snapshot cache       |
| `mesoscope:mesoscope-vr`                      | Provides `read_session_system_configuration_tool` to cross-check |
| `assets:experiment-configuration`             | Provides `read_experiment_configuration_tool` to cross-check     |
| `experiment:zaber-interface`                  | Owns live motor configuration and per-motor calibration          |

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
