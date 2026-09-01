---
name: mesoscope-vr-snapshots
description: >-
  Reads and writes the Mesoscope-VR per-session frozen position snapshots (ZaberPositions, MesoscopePositions) via the
  `sle mcp` server, and documents the Mesoscope-VR raw-data layout contract (MesoscopeRawDataFiles,
  MesoscopeDirectories, MesoscopeRawData.build). Owns the position snapshot write tools. Use when inspecting motor
  positions recorded during a session, patching positions after a manual adjustment, recovering a corrupted snapshot, or
  resolving a Mesoscope-VR raw-data path.
user-invocable: false
---

# Mesoscope-VR position snapshots

Reads and writes the per-session frozen position snapshot YAML files written by the runtime during a `sle mesoscope run
<session-type>` session. Every session type writes `zaber_positions.yaml`, and the `experiment` and `window-checking`
session types additionally write `mesoscope_positions.yaml`. Uses the `sle mcp` server. This skill is the **exclusive**
owner of `write_session_zaber_positions_tool` and `write_session_mesoscope_positions_tool`. No other skill in the
marketplace may call these.

These position snapshots are specific to the Mesoscope-VR acquisition system. Both the snapshot schemas and the MCP
tools that read and write them are bound to Mesoscope-VR hardware, which is the Zaber motor groups and the mesoscope
objective, and neither carries an `acquisition_system` schema selector. The *live* Zaber motor interface is
platform-general and reusable across acquisition systems, and `experiment:zaber-interface` owns it. This skill manages
the *frozen per-session records* alone.

The third per-session snapshot, `MesoscopeHardwareState`, is **not** covered by this skill. It lives on the slsa MCP
server, because the dataclass is shared between the acquisition runtime and the processing pipeline. Hand off to
`assets:session-hardware-state` for the generic read, write, and describe tooling over `hardware_state.yaml`, and to
`/mesoscope-vr-session-schema` for the field-level Mesoscope-VR schema those tools operate on.

---

## Scope

**Covers:**
- Reading and writing `ZaberPositions` (frozen Zaber motor positions recorded near the end of the session)
- Reading and writing `MesoscopePositions` (frozen mesoscope objective positions recorded at the end of the session)
- The Mesoscope-VR raw-data layout contract (`MesoscopeRawDataFiles`, `MesoscopeDirectories`, and the
  `MesoscopeRawData.build` field resolution), which this skill owns exclusively and `assets:session-data` defers to

**Does not cover:**
- Reading or writing `MesoscopeHardwareState` (`hardware_state.yaml`), which `assets:session-hardware-state` owns,
  and its field-level schema, which `/mesoscope-vr-session-schema` owns
- Reading the `SessionData` marker file (see `assets:session-data`)
- Reading or writing session descriptors (see `assets:session-descriptors`)
- Reading the frozen system or experiment configuration files at session start (those are read via `/mesoscope-vr`'s
  `read_session_system_configuration_tool` and the assets plugin's `assets:experiment-configuration`
  `read_experiment_configuration_tool`)
- Reading subject metadata (see `assets:data-assets`)
- Live Zaber motor configuration during runtime (see `experiment:zaber-interface`)
- The slsa registration contract a new system's `<system>/raw_data.py` satisfies (see `assets:library-extension`)
- The sollertia-experiment seams a new system's snapshot tools plug into (see `experiment:library-extension`)
- The runtime that writes these snapshots (see `/mesoscope-vr-runtime`), and the `sle mesoscope` command and option
  surface that starts it (see `/mesoscope-vr-cli-reference`)

---

## What is a position snapshot

During a `sle mesoscope run <session-type>` acquisition session, the runtime records the positions of the motorized
stages and writes them to YAML files inside the session directory. The Zaber positions are queried automatically from
the motors. The mesoscope objective positions are queried from the ScanImage software over MQTT, except for the red-dot
alignment Z position (`red_dot_alignment_z`), which the ScanImage software cannot report and which the operator enters
through a terminal prompt. That prompt defaults to the value held by the animal's persistent snapshot. These **frozen
snapshots** are used post-hoc to:

- Reproduce the exact stage configuration that was active during the session
- Diagnose drift between the recorded positions and what the binding class expected
- Recover lost positions after a manual stage adjustment

The runtime writes the snapshots (not this skill). `generate_zaber_snapshot` returns early when at least one of the
three managed Zaber motor groups is disconnected, and again while the `nk.bin` marker is still present
(`mesoscope_vr/acquisition_components.py`). A session that trips either gate ends with no `zaber_positions.yaml` at all
rather than a partial one. For the `lick-training`, `run-training`, and `experiment` session types, which drive the
`MesoscopeVRSystem` controller, the controller writes the snapshot once, during `stop()`, after
`mark_runtime_initialized()` has cleared `nk.bin` (`mesoscope_vr/system_controller.py`). An aborted initialization
leaves `nk.bin` in place, so the write is skipped and no file is produced. The `window-checking` session type runs its
own sequence and writes the snapshot after it marks the session initialized and before it finalizes the session
descriptor (`window_checking_logic` in `mesoscope_vr/data_acquisition.py`). Both write sites run before the motors are
returned to their parking position, so the snapshot records the pose the session ended in rather than the parked pose.
`generate_mesoscope_position_snapshot` applies the same `nk.bin` gate, queries the driver, prompts for the red-dot Z,
and writes the session copy alongside the animal's persistent copy (`mesoscope_vr/acquisition_components.py`). The
`experiment` and `window-checking` session types seed a precursor mesoscope objective snapshot at session start and
record the queried objective positions at the end of the session, so `lick-training` and `run-training` session
directories carry only `zaber_positions.yaml`. This skill exists to **read** them for inspection and to **patch** them
when a snapshot file is corrupted or out of sync with reality.

Unlike `zaber_positions.yaml`, a failed end-of-session query does not leave `mesoscope_positions.yaml` missing. The
precursor is a copy of the animal's persistent snapshot (`mesoscope_vr/system_controller.py`, and
`window_checking_logic` in `mesoscope_vr/data_acquisition.py`), so when the ScanImagePC query or the red-dot prompt
fails during teardown, the session keeps a complete, non-zero file holding the *previous* session's coordinates. For an
`experiment` session, the controller runs that query through `run_shutdown_step`, which logs the error and continues to
the next teardown step, so the session completes normally with the stale file in place. For a `window-checking` session,
the call is unguarded, so the failure propagates and also skips the remaining success-path steps, including
preprocessing. A non-zero mesoscope read is therefore not proof that the session's own positions were recorded, and the
all-zero corruption heuristic does not catch this case. Compare the session's `raw_data` copy against the animal's
`persistent_data/mesoscope_positions.yaml`, and treat a field-for-field identical match as the signature of a skipped
query rather than a real record.

| Snapshot             | File                       | Captures                                                         |
|----------------------|----------------------------|------------------------------------------------------------------|
| `ZaberPositions`     | `zaber_positions.yaml`     | Headbar / lickport / wheel positions, captured before parking    |
| `MesoscopePositions` | `mesoscope_positions.yaml` | Mesoscope objective and ScanImage virtual axes, plus laser power |

---

## MCP tool surface

| Tool                                     | MCP server | Purpose                                                              |
|------------------------------------------|------------|----------------------------------------------------------------------|
| `read_session_zaber_positions_tool`      | `sle mcp`  | Reads `ZaberPositions` for a session                                 |
| `write_session_zaber_positions_tool`     | `sle mcp`  | Writes (full replace) `ZaberPositions` (exclusive to this skill)     |
| `read_session_mesoscope_positions_tool`  | `sle mcp`  | Reads `MesoscopePositions` for a session                             |
| `write_session_mesoscope_positions_tool` | `sle mcp`  | Writes (full replace) `MesoscopePositions` (exclusive to this skill) |

Both write tools, `write_session_zaber_positions_tool` and `write_session_mesoscope_positions_tool`
(`interfaces/mesoscope_vr_tools.py`), take the snapshot payload as the `positions_payload` argument and accept a
keyword-only `overwrite` flag that defaults to `True`. That default replaces an existing snapshot silently. The system
configuration writer that `/mesoscope-vr` owns, `write_system_configuration_tool` (`interfaces/mesoscope_vr_tools.py`),
defaults the same flag to `False`, so the safer default does not carry over to these two tools. You MUST pass
`overwrite=False` on any write you did not intend as a deliberate rewrite of a recorded history.

Every tool here takes `session_path`, which `_resolve_session_root` accepts as either a session root holding a
`raw_data` subdirectory or the `raw_data` directory itself, returning the parent in the second case
(`interfaces/mesoscope_vr_tools.py`). Any other existing path fails with `Could not locate the raw_data directory
under <path>`, and a path that does not exist fails with `Session path does not exist: <path>`. Because the
resolution is anchored on `raw_data`, these tools cannot reach the per-animal `persistent_data` copies of the snapshots
that the recovery workflow draws on.

The `sle mcp` server exposes no describe-schema tool for either snapshot, so the field rosters below are the reference.
Each write tool round-trips `positions_payload` through `write_yaml_validated`, the four-stage shared write helper of
the `sle mcp` server (`interfaces/mcp_instance.py`). Stage 1 is the existence gate that `overwrite` releases.
Stage 2 rejects payload keys the dataclass does not declare, reporting them as dotted names. Stage 3 writes the payload
to a temporary sibling file, loads it back through the dataclass, and re-runs `__post_init__` when the class defines
one. Stage 4 rejects a field whose value violates its declared annotation, then persists through `to_yaml`, since
neither snapshot class defines a `save()` method. The `slsa mcp` server carries its own `write_yaml_validated`
(`sollertia-shared-assets/src/sollertia_shared_assets/interfaces/mcp_instance.py`), which runs stages 1 and 3 alone, so
the permissive write semantics the assets skills document do not transfer here:

| Payload defect      | Outcome                                                                                                  |
|---------------------|----------------------------------------------------------------------------------------------------------|
| Omitted key         | Accepted silently, written at the field's dataclass default (`0` or `0.0`)                               |
| Misspelled key      | Rejected at stage 2 with `Validation failed for <Class>: the payload carries the unknown key(s) <name>.` |
| Wrongly typed value | Rejected at stage 4 with `Validation failed for <Class>: <field> is <type>, expected <annotation>.`      |

Because the omitted-key case is the silent one, you MUST read the current snapshot first and send the complete field
roster on every write.

On the read side, both read tools detect only file-level corruption: a missing file, malformed YAML, or a document that
does not load as its dataclass. A snapshot whose keys were dropped or misspelled loads cleanly, with `0` or `0.0`
standing in for every field the file does not name, so a successful read is not evidence that the file is intact. You
MUST treat an all-zero or partially zero read as suspect and check it against the animal's `persistent_data` copy before
using it.

`ZaberPositions` (`zaber_positions.yaml`) holds seven `int` fields defaulting to `0`, each an absolute position in
native motor units (`mesoscope_vr/system.py`):

| Field           | Captures                                     |
|-----------------|----------------------------------------------|
| `headbar_z`     | HeadBar z-axis motor position                |
| `headbar_pitch` | HeadBar pitch-axis motor position            |
| `headbar_roll`  | HeadBar roll-axis motor position             |
| `lickport_z`    | LickPort z-axis motor position               |
| `lickport_y`    | LickPort y-axis motor position               |
| `lickport_x`    | LickPort x-axis motor position               |
| `wheel_x`       | Running wheel platform x-axis motor position |

`MesoscopePositions` (`mesoscope_positions.yaml`) holds nine `float` fields defaulting to `0.0`
(`mesoscope_vr/system.py`):

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
spurious sub-micrometer and sub-millidegree precision the ScanImage software reports. That rounding runs through
`_floor_to_three_decimals`, which `generate_mesoscope_position_snapshot` applies to every field before it persists the
snapshot (both in `mesoscope_vr/acquisition_components.py`). A patched payload should carry at most three decimals to
stay consistent with the runtime-written records.

For the `MesoscopeHardwareState` read/write/describe trio (`read_session_hardware_state_tool`,
`write_session_hardware_state_tool`, `describe_session_hardware_state_schema_tool`), hand off to
`assets:session-hardware-state`. For the fields those tools carry, hand off to `/mesoscope-vr-session-schema`.

---

## Mesoscope-VR raw-data layout contract

Descriptors, hardware state, trial types, experiment configuration, and **raw-data layout** form a universal per-system
contract, and every Sollertia acquisition system defines its own concrete instance of each. The generic registry
dispatch and the create/write/validate/describe tooling are system-agnostic and live in the assets plugin (see
`assets:session-data`). This section documents only Mesoscope-VR's **concrete instance** of the raw-data layout
contract, which is the system-specific filenames, subdirectories, and path-resolution fields keyed by
`SYSTEM_RAW_DATA_REGISTRY[AcquisitionSystems.MESOSCOPE_VR] -> MesoscopeRawData`
(`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`). That registry is the one dispatch table absent
from the package root exports, so code imports it from the submodule as `from sollertia_shared_assets.registries import
SYSTEM_RAW_DATA_REGISTRY`.

Unlike the descriptor, hardware-state, and experiment-configuration contracts, which register a `YamlConfig`
describe/read/write trio, the raw-data contract registers a **path-resolution dataclass**. `MesoscopeRawData` does not
read or write any file. It resolves the absolute on-disk locations of the system-specific raw assets under a session's
`raw_data` directory (`sollertia-shared-assets/src/sollertia_shared_assets/mesoscope_vr/raw_data.py`). The snapshots
that this skill writes are two of those assets. The snapshot MCP tools reach those same two files on their own, joining
the module-local `_ZABER_POSITIONS_FILENAME` and `_MESOSCOPE_POSITIONS_FILENAME` constants onto the resolved session
root (`interfaces/mesoscope_vr_tools.py`), so they duplicate the canonical filenames that this dataclass resolves.

`MesoscopeRawDataFiles` enumerates the canonical filenames at the root of `raw_data` written exclusively by the
Mesoscope-VR acquisition system:

| Member                | Value                      | Captures                                                        |
|-----------------------|----------------------------|-----------------------------------------------------------------|
| `ZABER_POSITIONS`     | `zaber_positions.yaml`     | Position snapshot written at session end, before motor parking  |
| `MESOSCOPE_POSITIONS` | `mesoscope_positions.yaml` | Objective position snapshot from experiment and window-checking |
| `WINDOW_SCREENSHOT`   | `window_screenshot.png`    | Cranial imaging window screenshot captured at session start     |

`window_screenshot.png` arrives through `setup_mesoscope`, which blocks the operator until exactly one `.png` file sits
at the root of the ScanImagePC mesoscope data directory, moves that file to the session's `window_screenshot_path`, and
copies it to the animal's persistent directory (`mesoscope_vr/acquisition_components.py`). `setup_mesoscope`
runs only for `experiment` and `window-checking` sessions, so `lick-training` and `run-training` sessions carry no
`window_screenshot.png`. That absence is the expected layout for those session types rather than a missing file.

`MesoscopeDirectories` enumerates the canonical subdirectory names under `raw_data` written exclusively by the
Mesoscope-VR acquisition system:

| Member           | Value            | Captures                                                                      |
|------------------|------------------|-------------------------------------------------------------------------------|
| `MESOSCOPE_DATA` | `mesoscope_data` | LERC-compressed TIFF stacks and acquisition metadata written by preprocessing |

`MesoscopeRawData.build(root)` is the single source of truth for the enum-to-field mapping. `SessionData` constructs the
instance when the session's acquisition system is `MESOSCOPE_VR`, resolving each field as `root.joinpath(<enum value>)`:

| Field                      | Resolves to                         | Source enum member                          |
|----------------------------|-------------------------------------|---------------------------------------------|
| `zaber_positions_path`     | `raw_data/zaber_positions.yaml`     | `MesoscopeRawDataFiles.ZABER_POSITIONS`     |
| `mesoscope_positions_path` | `raw_data/mesoscope_positions.yaml` | `MesoscopeRawDataFiles.MESOSCOPE_POSITIONS` |
| `window_screenshot_path`   | `raw_data/window_screenshot.png`    | `MesoscopeRawDataFiles.WINDOW_SCREENSHOT`   |
| `mesoscope_data_path`      | `raw_data/mesoscope_data/`          | `MesoscopeDirectories.MESOSCOPE_DATA`       |

`zaber_positions_path` and `mesoscope_positions_path` are the on-disk targets the position-snapshot tools read and
write, which is this skill's surface. The runtime and preprocessing produce `window_screenshot_path` and
`mesoscope_data_path`, which this skill never writes. For the generic registry dispatch and the system-agnostic tools
that operate over the raw-data layout, hand off to `assets:session-data`.

`SessionData` exposes this instance as the `system_raw_data` attribute, built by `_build_sub_dataclasses` during
`create()` and `load()` alone. An instance built straight through `from_yaml` carries no `system_raw_data`, and touching
the attribute raises `AttributeError`. The `SessionData` class docstring and `SessionData._build_sub_dataclasses` state
and enforce that rule, both in `sollertia-shared-assets/src/sollertia_shared_assets/data_hierarchy/session_data.py`.

`MesoscopeRawData` is declared `@dataclass(frozen=True, slots=True)`, so assigning to a resolved path raises
`FrozenInstanceError`. Rebuild the instance with `MesoscopeRawData.build(root=...)` instead of mutating one.

### Platform seams this instance fills

Mesoscope-VR is currently the only registered acquisition system, so this skill doubles as the worked example an
extender copies when authoring a new system's snapshot surface. Each row names the Mesoscope-VR choice, the
platform-general seam it fills, and the skill that owns that seam.

| Mesoscope-VR choice                                                   | Platform-general seam                                          | Seam owner                              |
|-----------------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------|
| `MesoscopeRawData` keyed into `SYSTEM_RAW_DATA_REGISTRY`              | The slsa raw-data layout registry a new system claims          | `assets:library-extension`              |
| `ZaberPositions` and `MesoscopePositions` in `mesoscope_vr/system.py` | A system's own snapshot dataclasses, held outside any registry | `experiment:library-extension`          |
| The four snapshot tools in `interfaces/mesoscope_vr_tools.py`         | The `<system>_tools.py` module the MCP server globs on import  | `experiment:library-extension`          |
| The private `_RAW_DATA_DIR` and filename constants                    | Per-system tool-module constants, shared with nothing          | `experiment:library-extension`          |
| `write_yaml_validated` and `read_yaml` reused unchanged               | The shared validation plumbing every system's tools inherit    | `experiment:library-extension`          |
| The `nk.bin` gate both snapshot writers apply                         | `mark_runtime_initialized()` and the initialization marker     | `experiment:acquisition-system-runtime` |

---

## Workflows

### Inspecting position snapshots for a session

1. **Verify prerequisites:** `sle mcp` (sollertia-experiment) is connected. If not, hand off to
   `experiment:experiment-mcp-environment-setup`.
2. **Read the snapshots:**
   ```text
   read_session_zaber_positions_tool(session_path="<absolute>")
   read_session_mesoscope_positions_tool(session_path="<absolute>")
   ```
   Only the `experiment` and `window-checking` session types write `mesoscope_positions.yaml`. For a
   `lick-training` or `run-training` session, read the Zaber snapshot alone, because the mesoscope read returns a
   file-not-found error that reflects the expected layout.
3. **Report to the user.** Optionally cross-reference with the live system configuration via `/mesoscope-vr` to
   identify drift, or hand off to `assets:session-hardware-state` to also pull the hardware state snapshot. Only the
   `MesoscopeVRSystem` controller writes `hardware_state.yaml`, so that handoff applies to `lick-training`,
   `run-training`, and `experiment` sessions. A `window-checking` session runs its own sequence and carries no
   `hardware_state.yaml` at all.

### Rescuing a bad auto-generated `ZaberPositions` snapshot

A snapshot that exists is always complete, because all seven `ZaberPositions` fields are built in one constructor call
from a single query pass. A disconnected motor group or a session still carrying `nk.bin` yields no snapshot file at all
rather than a truncated one. What a session-time fault does produce is a complete snapshot recording the pose the motors
reached after the fault, which is a faithful record of the wrong stage configuration. Use this workflow to overwrite
such a snapshot with the positions the session would have recorded had it completed normally, restoring a faithful
frozen record of the stage configuration for downstream use. The write tool targets the session's `raw_data` copy only.
The per-animal `persistent_data` copy that seeds the next runtime is a separate file and is not modified here.

1. **Verify the user actually wants to modify the frozen snapshot.** Patching a snapshot rewrites history, and
   the original captured state is lost.
2. **Read the current positions:**
   ```text
   read_session_zaber_positions_tool(session_path="<absolute>")
   ```
3. **Build the corrected dictionary** with the positions the session should have recorded, which is typically the same
   animal's last good snapshot (see [Recovering a corrupted snapshot](#recovering-a-corrupted-snapshot) for sourcing it
   from an adjacent session or the `persistent_data` copy). Carry all seven `ZaberPositions` fields listed under
   [MCP tool surface](#mcp-tool-surface), since an omitted key silently writes its default.
4. **Confirm with the user before writing.**
5. **Write the corrected positions:**
   ```text
   write_session_zaber_positions_tool(session_path="<absolute>", positions_payload={ ... })
   ```
6. **Re-read to verify.**

### Recovering a corrupted snapshot

If a position snapshot fails to parse, the read tool returns an error. The two position snapshots cover disjoint
subsystems (Zaber stages vs. the Mesoscope objective) and share no fields, so the *other* snapshot in the same session
cannot supply the corrupted one's values. Recover from the same snapshot **type** elsewhere. The runtime mirrors each
snapshot to the animal's `persistent_data` directory, overwriting it on every completed session, and re-seeds the next
runtime from it, so the same animal's positions drift little between sessions.

1. **Recover the values from the same-type snapshot of the same animal.** Read an adjacent same-animal
   session's snapshot of the same type with `read_session_zaber_positions_tool` or
   `read_session_mesoscope_positions_tool`, passing that session's `session_path`. The alternative is a plain read of
   the animal's persistent-directory copy, which is `<animal>/persistent_data/zaber_positions.yaml` or
   `mesoscope_positions.yaml` and carries the same schema as the session snapshot. The persistent copy reflects
   the most recent completed session, so it is an exact match when the corrupted snapshot is the latest and a
   close approximation otherwise. Reject an all-zero persistent `mesoscope_positions.yaml` as a recovery source. A
   window-checking session seeds that file with all-zero defaults at session start for an animal it has never imaged,
   so an animal whose first window-checking session never reached teardown carries defaults rather than a recorded
   pose.
2. **Pull rig context to validate a candidate** (not to recover positions). Hand off to `assets:session-hardware-state`
   for `MesoscopeHardwareState`, and read the frozen system configuration via `/mesoscope-vr`
   (`read_session_system_configuration_tool`) for the three Zaber USB port strings it carries: `headbar_port`,
   `lickport_port`, and `wheel_port`. The frozen configuration holds no per-motor calibration. The park, maintenance,
   and mount positions, the motion limits, and the device labels live in the motors' non-volatile memory and are read
   through `experiment:zaber-interface` and its `get_zaber_device_settings_tool`. Use both sources to confirm a
   candidate snapshot matches the rig that ran the session.
3. Hand the recovery proposal to the user before writing a replacement snapshot.

### The per-animal persistent-data layout

The runtime caches four files per animal so that each new session can seed itself from the previous one. They live in
the animal's persistent directory, which is `AnimalData.persistent_data_path` and is owned by
`assets:project-hierarchy`:

| File                             | Written when                                                            |
|----------------------------------|-------------------------------------------------------------------------|
| `zaber_positions.yaml`           | Session end, together with the session's `raw_data` copy                |
| `mesoscope_positions.yaml`       | Session end for `experiment` and `window-checking` sessions, see below  |
| `window_screenshot.png`          | Session start, after the screenshot moves into the session's `raw_data` |
| `<session-type>_descriptor.yaml` | Session end, copied from the session's finalized descriptor             |

The `window-checking` runtime also writes the persistent `mesoscope_positions.yaml` at session start, seeding it with
all-zero defaults when the animal has no previous persistent snapshot (`window_checking_logic` in
`mesoscope_vr/data_acquisition.py`). The `experiment` runtime writes its all-zero precursor to the session's `raw_data`
copy alone, leaving the persistent file untouched until teardown.

The descriptor filename varies with the session type, and `/mesoscope-vr-session-schema` carries the four exact
names. **No MCP tool reaches any of these files.** The snapshot tools resolve their targets under a session's `raw_data`
directory, and neither server exposes a tool that takes a persistent-directory path. You MUST read a persistent copy
with a plain filesystem read.

### Coordinated multi-snapshot patching

If you need to patch position snapshots **and** the hardware state snapshot together (e.g., after replacing the entire
acquisition rig), patch the position snapshots from this skill and hand off to `assets:session-hardware-state` for the
hardware state write. `assets:session-hardware-state` owns `write_session_hardware_state_tool`.

---

## Related skills

| Skill                                         | Relationship                                                                      |
|-----------------------------------------------|-----------------------------------------------------------------------------------|
| `/mesoscope-vr-cli-reference`                 | Owns the `sle mesoscope run` commands whose runtime writes these snapshots        |
| `experiment:experiment-mcp-environment-setup` | Run first if `sle mcp` is not connected                                           |
| `assets:session-hardware-state`               | Sibling that owns `MesoscopeHardwareState`, the third snapshot                    |
| `/mesoscope-vr-session-schema`                | Owns the field-level descriptor and hardware-state schema                         |
| `/mesoscope-vr-runtime`                       | Owns the runtime that writes these snapshots                                      |
| `assets:session-data`                         | Owns the `SessionData` marker file and the raw-data dispatch                      |
| `assets:session-descriptors`                  | Owns the per-session descriptor files                                             |
| `assets:project-hierarchy`                    | Owns `AnimalData.persistent_data_path`, the snapshot cache                        |
| `/mesoscope-vr`                               | Provides `read_session_system_configuration_tool` to cross-check                  |
| `assets:experiment-configuration`             | Provides `read_experiment_configuration_tool` to cross-check                      |
| `experiment:zaber-interface`                  | Owns live motor configuration and per-motor calibration                           |
| `assets:library-extension`                    | Owns the slsa registration contract the raw-data layout satisfies                 |
| `experiment:library-extension`                | Owns the sollertia-experiment seams the snapshot tools plug into                  |
| `experiment:acquisition-system-runtime`       | Owns the initialization marker that gates both snapshot writers                   |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Prerequisites:
- [ ] sle mcp (sollertia-experiment) is connected
- [ ] User confirmed the planned snapshot patch, since snapshots are historical records

Writes:
- [ ] Every write payload carried the complete field roster for its snapshot type
- [ ] Passed overwrite=False on any write not intended as a replacement of an existing snapshot
- [ ] write_session_*_positions_tool succeeded without errors
- [ ] read_session_*_positions_tool returned the expected content after every write

Boundaries:
- [ ] Handed off to assets:session-hardware-state rather than calling write_session_hardware_state_tool
- [ ] Left SessionData, descriptors, and subject metadata to their owning skills
```
