---
name: mesoscope-vr
description: >-
  Knowledge repository for the Mesoscope-VR data acquisition system: its hardware subsystem
  inventory, the MesoscopeSystemConfiguration dataclass and YAML lifecycle, the per-subsystem
  binding classes, and the hardware/calibration modification workflows. Use when configuring,
  modifying, or auditing Mesoscope-VR's hardware and configuration layer.
user-invocable: false
---

# Mesoscope-VR system

Knowledge repository for the Mesoscope-VR data acquisition system, currently the only system the
Sollertia platform supports. This skill documents the system's hardware composition, configuration
surface, and binding-class layer.

For the platform-general acquisition-system design pattern, see `experiment:acquisition-system-design`.
For Mesoscope-VR's runtime behavior (state machine, training modes, CLI), see
`/mesoscope-vr-runtime`.

---

## Scope

**Covers:**
- Mesoscope-VR system overview (hardware composition, controllers, cameras, motors, Mesoscope acquisition)
- `MesoscopeSystemConfiguration` dataclass — top-level configuration class structure and file lifecycle
- Per-subsystem configuration dataclass surface — pointer to the full registry in `references/configuration-fields.md`
- Per-subsystem binding classes (`MicroControllerInterfaces`, `VideoSystems`, `ZaberMotors`) and the
  `MesoscopeDriver` MQTT interface — composition and lifecycle wiring
- MCP tool surface for reading, writing, and validating the configuration YAML
- Configuration authoring and modification workflows

**Does not cover** (delegated):
- The platform-general design pattern this system implements — see `experiment:acquisition-system-design`
- Mesoscope-VR runtime behavior (state machine, training modes, visualizers, session descriptors,
  CLI commands) — see `/mesoscope-vr-runtime`
- Per-firmware-module Python wrappers and slmc firmware Modules — see `experiment:microcontroller-interface`
- Low-level VideoSystem API — see `ataraxis@video:camera-interface`
- Low-level Zaber motor API — see `experiment:zaber-interface`
- Low-level MicroControllerInterface API — see `ataraxis@communication:microcontroller-interface`
- Per-session metadata, task templates, experiment configuration — owned by the assets plugin
- Server transfer configuration — owned by the forging plugin

---

## System overview

Mesoscope-VR is a head-fixed 2-Photon Random Access Mesoscope (2P-RAM) imaging system with a
Virtual Reality environment. It is composed of the following hardware subsystems:

| Subsystem             | Devices                                                            | Binding class                       |
|-----------------------|--------------------------------------------------------------------|-------------------------------------|
| Microcontrollers      | 3 × Teensy 4.1 boards (ACTOR, SENSOR, ENCODER)                     | `MicroControllerInterfaces`         |
| Cameras               | 2 × GenICam scientific cameras (face camera, body camera)          | `VideoSystems`                      |
| Zaber motors          | 3 × motor groups (HeadBar Z/Pitch/Roll, Wheel X, LickPort Z/Y/X)   | `ZaberMotors`                       |
| Unity VR (MQTT)       | 1 × MQTT broker bridging the runtime to the Unity game engine      | (consumed directly by orchestrator) |
| Mesoscope acquisition | ScanImage 2P-RAM Mesoscope driven over MQTT via runAcquisition.m   | `MesoscopeDriver`                   |

The lifecycle orchestrator (`MesoscopeVRSystem` in
`sollertia_experiment/mesoscope_vr/system_controller.py`) composes the three binding classes and
drives the system state machine. The orchestrator is documented in `/mesoscope-vr-runtime`.

---

## Authoritative bases

Read these skills before reading the hardware-subsystem sections below:

| Concern                                              | Authority                                              |
|------------------------------------------------------|--------------------------------------------------------|
| Pattern this system implements                       | `experiment:acquisition-system-design`                 |
| Per-firmware-module wrappers (slmc + sle pair)       | `experiment:microcontroller-interface`                 |
| Low-level VideoSystem API                            | `ataraxis@video:camera-interface`                      |
| Low-level Zaber motor API                            | `experiment:zaber-interface`                           |
| Low-level MicroControllerInterface API               | `ataraxis@communication:microcontroller-interface`     |

This skill documents only Mesoscope-VR-specific composition. All base mechanics are inherited from
the skills above.

---

## MesoscopeSystemConfiguration

The system's configuration is captured by `MesoscopeSystemConfiguration`, a `SystemConfiguration`-derived
dataclass defined in `sollertia_experiment/mesoscope_vr/system.py`.

### Top-level structure

`MesoscopeSystemConfiguration` composes six nested dataclasses plus a top-level `name` field:

| Section            | Dataclass                   | What it parameterizes                                            |
|--------------------|-----------------------------|------------------------------------------------------------------|
| `name`             | (str, top-level)            | Human-readable system label (default: `"mesoscope"`)             |
| `filesystem`       | `MesoscopeFileSystem`       | Local raw / processed / NAS / server / mesoscope-DAQ directories |
| `sheets`           | `MesoscopeGoogleSheets`     | Google Sheet IDs for surgery log and water log                   |
| `cameras`          | `MesoscopeCameras`          | Face / body camera indices and H.265 encoding parameters         |
| `microcontrollers` | `MesoscopeMicroControllers` | Per-board ports + per-module calibration data                    |
| `acquisition`      | `MesoscopeAcquisition`      | Mesoscope motion-estimation and z-stack acquisition parameters   |
| `assets`           | `MesoscopeVRAssets`         | Zaber motor ports + nested `vr_task` Unity MQTT configuration    |

For the full field-by-field registry (every field name, type, default, units, and meaning), see
[`references/configuration-fields.md`](references/configuration-fields.md). That file is a state
snapshot of the current Mesoscope-VR schema and MUST be updated whenever any dataclass field is
added, removed, or renamed.

### YAML file lifecycle

The configuration is persisted as `mesoscope_system_configuration.yaml` in the platform
configuration directory resolved by `get_system_configuration_path()`. The host's data root must be
set first via `assets:working-directory` skill (`slsa configure data-root`).

`MesoscopeSystemConfiguration` implements two non-default behaviors documented in
`experiment:acquisition-system-design`:

- **`__post_init__`** normalizes the valve calibration table from its YAML-natural `dict` shape to
  the in-memory `tuple[tuple[int | float, int | float], ...]` shape that `WaterValveInterface` consumes.
  It also validates that every entry is a 2-tuple of numeric values; malformed entries raise
  `TypeError`.
- **`save()`** override temporarily converts the valve calibration table back to a `dict` before
  writing to YAML so existing files retain the mapping layout, then restores the tuple form.

`MesoscopeSystemConfiguration` is registered with the shared cross-system configuration registry at
import time, and the package exposes a typed `get_system_configuration() -> MesoscopeSystemConfiguration`
accessor. Creating the file (`create_system_configuration_file()`), resolving its path
(`get_system_configuration_path()`), and loading it are handled by the shared `cross_system` helpers,
following the pattern in `experiment:acquisition-system-design`. The CLI entry point for authoring is
`sle mesoscope configure`.

### MCP tool surface

The configuration's read/write/validate surface is hosted on `sle mcp`
(`sollertia-experiment`). This skill is the **exclusive** owner of the write tool — no other skill
in the marketplace may call `write_system_configuration_tool`.

| Tool                                        | Purpose                                                                  |
|---------------------------------------------|--------------------------------------------------------------------------|
| `describe_system_configuration_schema_tool` | Returns the field schema for `MesoscopeSystemConfiguration`              |
| `read_system_configuration_tool`            | Reads the active system configuration YAML from the working directory    |
| `write_system_configuration_tool`           | Writes a new system configuration YAML (**exclusive to this skill**)     |
| `validate_system_configuration_tool`        | Validates the loaded system configuration and reports mount status       |
| `verify_camera_configuration_tool`          | Diffs each camera's live GenICam configuration against its stored config |
| `check_system_mounts_tool`                  | Checks every filesystem path declared in the configuration               |
| `read_session_system_configuration_tool`    | Reads the frozen system configuration captured at session start          |

The companion enumeration `list_supported_acquisition_systems_tool` lives on `slsa mcp`
(`sollertia-shared-assets`) since the `AcquisitionSystems` enum remains the shared vocabulary
across the platform.

The write tool accepts the full nested dictionary that maps onto the dataclass tree. It validates
the shape against the schema before writing and refuses partial updates — to change a single
field, read first, mutate the dictionary, then write the whole thing back.

---

## Hardware subsystem: Microcontrollers

The Mesoscope-VR system uses **three Teensy 4.1 microcontrollers** in dedicated roles:

| Board   | Controller ID | Role                                             | Modules                                         |
|---------|---------------|--------------------------------------------------|-------------------------------------------------|
| ACTOR   | 101           | Output control (irregular, command-driven)       | brake, reward valve, gas-puff valve, screens    |
| SENSOR  | 152           | Input sensing (regular polling, no interrupts)   | lick sensor, torque sensor, mesoscope-frame TTL |
| ENCODER | 203           | Quadrature encoder (hardware-interrupt-isolated) | wheel encoder                                   |

Board allocation reasoning is documented in `experiment:microcontroller-interface`'s "Controller
board allocation principles" section. The Mesoscope-VR three-board split is one valid application
of the platform-general allocation rules — primarily driven by interrupt isolation for the encoder.

### MesoscopeMicroControllers dataclass

`MesoscopeMicroControllers` (in `sollertia_experiment/mesoscope_vr/system.py`) holds:

- **Port assignments**: `actor_port`, `sensor_port`, `encoder_port` (OS-specific USB device paths;
  Linux defaults shown, overridden per host)
- **Keepalive**: `keepalive_interval_ms` (500 ms default)
- **Per-module calibration**: ~25 fields that parameterize the eight module wrappers running on the
  three boards (brake strength, lick thresholds, torque calibration, encoder PPR, wheel diameter,
  cm-per-Unity-unit, screen pulse duration, sensor polling delays, valve calibration table)

For the full per-field documentation, see [`references/configuration-fields.md`](references/configuration-fields.md)
under the "MesoscopeMicroControllers" section.

### MicroControllerInterfaces binding class

`MicroControllerInterfaces` (in `sollertia_experiment/mesoscope_vr/binding_classes.py`) composes
the three `MicroControllerInterface` instances and the eight Python wrappers from
`sollertia_experiment/cross_system/module_interfaces.py`.

Construction signature:

```python
MicroControllerInterfaces(
    data_logger: DataLogger,
    microcontroller_configuration: MesoscopeMicroControllers,
)
```

The constructor instantiates wrappers in three blocks (ACTOR, SENSOR, ENCODER) using the
configuration dataclass fields, then wraps each block in a `MicroControllerInterface` with the
corresponding controller ID and port. Wrappers are exposed as public attributes:

- ACTOR: `self.brake`, `self.valve`, `self.gas_puff_valve`, `self.screens`
- SENSOR: `self.mesoscope_frame`, `self.lick`, `self.torque`
- ENCODER: `self.wheel_encoder`

`start()` brings up all three controllers in order (actor → sensor → encoder), calls
`initialize_local_assets()` on wrappers with SharedMemoryArray (`wheel_encoder`, `valve`,
`gas_puff_valve`, `mesoscope_frame`, `lick`), then pushes runtime parameters to each module via
`set_parameters()` calls.

`stop()` tears down all three controllers in the same forward order (no reverse ordering needed
because controllers are independent; the Kernel handles its own module shutdown).

For per-wrapper mechanics, the cross-side contract with the firmware, and the
slmc + sle convention details, see `experiment:microcontroller-interface`.

---

## Hardware subsystem: Cameras

The Mesoscope-VR system uses **two GenICam scientific cameras** (Harvester-managed) for animal
behavior recording:

| Camera        | System ID | Role                                       |
|---------------|-----------|--------------------------------------------|
| face camera   | 51        | Animal's face and eye                      |
| body camera   | 62        | Animal's body (running posture)            |

System IDs are allocated from the DataLogger source-ID convention. The non-contiguous values (51,
62) preserve room for additional cameras if the rig is extended.

### MesoscopeCameras dataclass

`MesoscopeCameras` holds per-camera parameters:

- **Indices**: `face_camera_index`, `body_camera_index` (positions in the Harvester-managed camera list)
- **Display rates**: `face_camera_display_frame_rate`, `body_camera_display_frame_rate` (live preview
  FPS, independent of save rate)
- **Encoding**: `face_camera_quantization`, `body_camera_quantization`, `face_camera_preset`,
  `body_camera_preset` (H.265 quantization parameter and `EncoderSpeedPresets` enum)
- **Configuration paths** (optional): `face_camera_configuration_path`, `body_camera_configuration_path`
  — absolute paths to per-camera GenICam configuration YAMLs (an `ataraxis-video-system`
  `GenicamConfiguration` file) recording each camera's expected node configuration. An empty path
  (`Path()`) means none. By convention they live in the working-directory `configuration/` folder
  next to `*_system_configuration.yaml` (e.g. `face_camera_configuration.yaml`).

For the full per-field documentation, see [`references/configuration-fields.md`](references/configuration-fields.md)
under the "MesoscopeCameras" section.

### Camera GenICam configuration: verify, dump, restore

Declaring a camera's `*_configuration_path` records *where* its expected GenICam node configuration
lives, so agents do not have to be handed the path on every operation. The path is declarative only
— the acquisition runtime does not auto-apply it. The workflow is:

- **Verify** the live camera against its stored config: call `verify_camera_configuration_tool` (this
  skill's MCP server). It reads the configured paths, dumps each camera's live GenICam configuration,
  and returns a per-camera diff (`match`, identity match, `value_mismatches`, nodes present in only
  one side). Cameras with no path set are reported as `{"configured": false}`.
- **Dump** the current configuration to the stored path (e.g. after tuning nodes): use
  `ataraxis@video:camera-setup`'s `dump_genicam_config` (axvs MCP) with `output_file` set to the
  path declared in the system configuration.
- **Restore** a known-good configuration onto a camera: use `ataraxis@video:camera-setup`'s
  `load_genicam_config` with `config_file` set to the declared path.

Source the path from `read_system_configuration_tool` (`cameras.<role>_camera_configuration_path`)
so the dump/restore targets the declared file. For the GenICam node mechanics themselves, hand off
to `ataraxis@video:camera-setup`.

### VideoSystems binding class

`VideoSystems` (in `sollertia_experiment/mesoscope_vr/binding_classes.py`) composes the two
`VideoSystem` instances.

Construction signature:

```python
VideoSystems(
    data_logger: DataLogger,
    camera_configuration: MesoscopeCameras,
    output_directory: Path,
)
```

Both cameras are instantiated with `CameraInterfaces.HARVESTERS`, `VideoEncoders.H265`,
`OutputPixelFormats.YUV420` (monochrome), and `gpu=0` for hardware-accelerated encoding. Per-camera
parameters (index, display rate, quantization, preset) come from the configuration.

The binding class exposes per-camera lifecycle granularity rather than a single `start()`:

- `start_face_camera()` / `start_body_camera()` — begin frame acquisition (does NOT save)
- `save_face_camera_frames()` / `save_body_camera_frames()` — begin saving frames to disk
- `stop()` — stops both acquisition and saving for all cameras

This split exists because acquisition is started early in the session (for live preview / monitoring)
but saving is started later (after the animal is mounted and the runtime begins).

For VideoSystem mechanics, encoding configuration, and frame acquisition patterns, see
`ataraxis@video:camera-interface`.

---

## Hardware subsystem: Zaber motors

The Mesoscope-VR system uses **three Zaber motor groups** for positioning the headbar, wheel, and
lickport:

| Group    | Daisy-chain      | Axes                                        | Purpose                             |
|----------|------------------|---------------------------------------------|-------------------------------------|
| HeadBar  | Z → Pitch → Roll | 3-axis (Z linear + Pitch + Roll rotational) | Animal head positioning             |
| Wheel    | X only           | 1-axis (X linear)                           | Running wheel longitudinal position |
| LickPort | Z → Y → X        | 3-axis (Z, Y, X linear)                     | Lickport spatial positioning        |

Each group connects to its own USB serial port. The daisy-chain order is hardware-cabled and MUST
match the order the binding class assumes (Z first, then Pitch then Roll for HeadBar, etc.).

### Configuration in MesoscopeVRAssets

The Zaber serial port assignments live in `MesoscopeVRAssets` (alongside the Unity MQTT
configuration):

- `headbar_port` (default `/dev/ttyUSB0`)
- `wheel_port` (default `/dev/ttyUSB2`)
- `lickport_port` (default `/dev/ttyUSB1`)

These defaults use the Linux device-path form; the value is OS-specific (e.g. `COMx` on Windows) and is
overridden per host.

For the full per-field documentation including Unity MQTT settings, see
[`references/configuration-fields.md`](references/configuration-fields.md) under the
"MesoscopeVRAssets" section.

### ZaberMotors binding class

`ZaberMotors` (in `sollertia_experiment/mesoscope_vr/binding_classes.py`) composes three
`ZaberConnection` instances and exposes per-axis `ZaberAxis` accessors.

Construction signature:

```python
ZaberMotors(
    zaber_positions: ZaberPositions | None,
    zaber_configuration: MesoscopeVRAssets,
)
```

Note: this binding class does NOT take a `DataLogger` — Zaber motor state is per-session position
data captured by the `/mesoscope-vr-snapshots` skill, not real-time logged events.

The constructor connects to all three ports synchronously and retrieves the per-axis handles
assuming the documented daisy-chain order. If the previous session's `ZaberPositions` snapshot is
provided, `restore_position()` moves motors to those positions; otherwise it falls back to the
mounting/parking positions stored in each motor's non-volatile memory.

For motor mechanics (park/unpark safety, position management, configuration tooling, the
`ZaberConnection` / `ZaberDevice` / `ZaberAxis` hierarchy, MCP discovery), see
`experiment:zaber-interface`.

---

## Hardware subsystem: Mesoscope acquisition

The Mesoscope-VR system images through a ScanImage-controlled 2P-RAM Mesoscope hosted on a separate
machine (the ScanImagePC). The acquisition runtime does not drive ScanImage directly: it publishes
commands over MQTT to the `runAcquisition` MATLAB function, which configures the online
motion-estimation reference, acquires the high-definition reference z-stack, and arms and runs frame
acquisition. The Mesoscope frame stream is observed on the PC side as the SENSOR board's
mesoscope-frame TTL, not over MQTT.

### MesoscopeAcquisition dataclass

`MesoscopeAcquisition` (in `sollertia_experiment/mesoscope_vr/system.py`) is the `acquisition` field
of `MesoscopeSystemConfiguration`. It parameterizes the reference motion estimator and the
high-definition reference z-stack that the ScanImagePC generates at the start of each runtime:

- **Z-stack geometry**: `z_step_um`, `z_range_um`, `z_exclusion_um` (plane spacing, inclusive
  imaging range, and optional two-plane exclusion zone, all in micrometers) and `acquisition_order`
  (`MesoscopeAcquisitionOrder`, with members `INTERLEAVED` — one frame per plane per volume — and
  `SMOOTH` — all averaged frames at one plane before advancing).
- **Estimator / z-stack settings**: `registration_channel` (channel used for online registration
  and the z-stack), `field_curvature_correction` (bool), `frames_per_reference_plane` (frames
  averaged per reference plane), and `zstack_scale_factor` (X/Y resolution scaling for the
  high-definition z-stack).

`__post_init__` validates that `z_step_um`, `registration_channel`, `frames_per_reference_plane`,
and `zstack_scale_factor` are positive; that `z_range_um` is positive and ordered as
`(minimum, maximum)`; and that a configured (unequal) `z_exclusion_um` zone is ordered and falls
within `z_range_um`. For the full per-field documentation (types, defaults, units, validation), see
[`references/configuration-fields.md`](references/configuration-fields.md) under the
"MesoscopeAcquisition" section.

### MesoscopeDriver interface class

`MesoscopeDriver` (in `sollertia_experiment/mesoscope_vr/mesoscope_driver.py`) encapsulates all MQTT
communication with the `runAcquisition` MATLAB function on the ScanImagePC.

Construction signature:

```python
MesoscopeDriver(
    configuration: VRTaskConfiguration,
    acquisition: MesoscopeAcquisition,
)
```

Mesoscope control is tightly coupled to the Virtual Reality task, so the driver reuses the shared
Virtual Reality MQTT broker discovery fields (`ip` / `port`) from `assets.vr_task` rather than
defining its own. The orchestrator constructs the driver from `assets.vr_task` and
`acquisition`, and `connect()`s it only for mesoscope experiment sessions.

Method surface:

| Method                       | Role                                                                          |
|------------------------------|-------------------------------------------------------------------------------|
| `connect()` / `disconnect()` | Open / close the MQTT connection to the ScanImagePC                           |
| `await_alive()`              | Probe ScanImagePC liveness with a request-reply handshake on the Status topic |
| `is_alive(timeout_ms)`       | One-shot bounded liveness probe (non-blocking) for the pre-flight check       |
| `preload(project, animal)`   | Preload the persisted per-animal reference estimator as an alignment aid      |
| `generate_reference()`       | Generate the fresh session estimator + high-definition z-stack and arm        |
| `begin_acquisition()`        | Begin acquiring session frames                                                |
| `abort()`                    | Abort or end the ongoing frame acquisition                                    |
| `recover()`                  | Reload the session estimator and re-arm after a transient interruption        |
| `query_state()`              | Return a `MesoscopePositions` snapshot of the stage / fast-Z / laser state    |

Each command is published, resent on each acknowledgement timeout, and confirmed by a reception
acknowledgement on the Status topic; setup commands additionally block for a terminal status state.
The driver probes liveness by publishing an empty `MesoscopeAlive` request and waiting for the
Status-topic acknowledgement — the absence of a reply within the timeout means the `runAcquisition`
command loop is not running. Actual frame acquisition start/stop is confirmed by the caller through
the hardware TTL frame stream, not over MQTT.

The driver and the MATLAB function exchange messages on a flat `Mesoscope`-prefixed PascalCase topic
namespace that does not overlap with the Unity task topics, so both surfaces share one broker. Ten
topics are defined: the VRPC publishes the command topics `MesoscopeAlive`, `MesoscopePreload`,
`MesoscopeGenerateReference`, `MesoscopeBeginAcquisition`, `MesoscopeAbort`, `MesoscopeRecover`, and
`MesoscopeQueryState`; the ScanImagePC publishes the reply topics `MesoscopeStatus` (reception
acknowledgement and progress), `MesoscopeError` (failure detail), and `MesoscopeState` (the
stage / fast-Z / laser snapshot answering a state query).

The `acquisition` parameters are carried in the command payloads, not configured on the ScanImagePC:
`generate_reference()` ships the full acquisition parameter set, `recover()` ships only the
plane-geometry parameters (`z_step_um`, `z_range_um`, `z_exclusion_um`, `acquisition_order`) needed
to re-derive the imaging planes, and `preload()` ships the `project` and `animal` identifiers the
ScanImagePC uses to resolve the persisted estimator path under its local data root. The remaining
commands carry empty payloads. This makes the `MesoscopeAcquisition` section the single source of
truth for the acquisition geometry.

The ScanImagePC counterpart is the `runAcquisition` MATLAB function
(`assets/mesoscope_vr/runAcquisition.m`), a top-level script deployed to the ScanImagePC. It connects
to the shared broker, then enters an MQTT command loop that acts as a state machine dispatching these
commands to the ScanImage software (preload, generate reference, begin / abort / recover, and answer
the liveness probe and state queries). Only the ScanImagePC-local output root and the broker address
remain function arguments; every acquisition parameter arrives in the command payloads.

For when the orchestrator invokes these methods within the runtime state machine, see
`/mesoscope-vr-runtime`.

### Pre-flight bridge check

Before a Mesoscope imaging session (`window-checking` or `experiment`), confirm the ScanImagePC's
`runAcquisition` control loop is reachable. `runAcquisition` is a **lock-in** command loop: the operator launches
it once in the MATLAB command line on the ScanImagePC, and it runs continuously — holding the command line for the
whole runtime — until interrupted with Ctrl-C or the broker drops. An unreachable bridge means it is not running,
so the runtime cannot arm or command the Mesoscope.

This skill owns the mesoscope-specific bridge check; the platform-general `experiment:system-health-check` hands
off here for it (the ScanImage bridge is exclusive to Mesoscope-VR, unlike the shared Unity bridge). The check is
backed by `MesoscopeDriver.is_alive()` — a single bounded `MesoscopeAlive` probe that, unlike the runtime's
interactive `await_alive()`, does not block or re-prompt. It is exposed on two surfaces, both mesoscope-specific
(NOT on the hardware-agnostic `sle get` surface):

- **MCP**: `check_mesoscope_bridge_tool` (`sle mcp`) — returns `{"reachable": ..., "status": ...}`, or
  `{"error": ...}` on failure.
- **CLI**: `sle mesoscope check-bridge` — prints a status line at SUCCESS (reachable) or WARNING (unreachable).

Both load the active configuration, resolve the shared broker (`assets.vr_task.ip` / `port`), probe once, and
disconnect. On an unreachable result, the remediation is to launch `runAcquisition(hSI, hSICtl, ...)` in MATLAB on
the ScanImagePC, then re-check.

---

## Auxiliary configuration sections

### MesoscopeFileSystem

Captures the filesystem layout — two fields:

- `mesoscope_directory` (`Path`) — the local-filesystem-mounted directory where the Mesoscope-DAQ
  PC aggregates acquired data during runtime.
- `storage_directories` (`dict[str, Path]`) — maps each long-term storage destination name to its
  local-filesystem-mounted project-root path. Seeded with the two `MesoscopeStorageDestination`
  members (`"NAS"` and `"Server"`), but a system may configure any number of destinations under
  arbitrary names. A destination left as an empty path is treated as not configured and is skipped
  during data transfer and removal; the mapping order defines the pull-back preference order.

The local **data root** (the directory under which all projects are stored on this machine) is NOT
part of this section — it is the platform-shared data root, resolved with `get_data_root()` and set
with the `slsa configure data-root` command (`assets:working-directory`).

Both fields default to empty paths; the user MUST set them per-host. The `check_system_mounts_tool`
MCP tool validates that every declared path exists and is writable before any session starts.

### MesoscopeGoogleSheets

Captures Google Sheets identifiers — two `str` fields:

- `surgery_sheet_id` — surgical intervention metadata sheet
- `water_log_sheet_id` — water restriction and handling log

Both default to empty strings; the user MUST populate them per-host. Sheet IDs are the long
alphanumeric segments in the Google Sheets URL.

### MesoscopeVRAssets (Unity-VR portion)

In addition to the three Zaber ports, this section holds a nested `vr_task` field of type
`VRTaskConfiguration` (from `sollertia_experiment/vr_task/configuration.py`) that stores the MQTT
broker discovery fields used to reach the Unity game engine:

- `assets.vr_task.ip` (default `"127.0.0.1"`) — IP address of the MQTT broker
- `assets.vr_task.port` (default `1883`) — port number of the MQTT broker

`VRTaskConfiguration` holds only the MQTT discovery fields. The geometric VR parameters (cue
catalog, corridor geometry, cm-per-Unity-unit conversion) are NOT stored here — they are resolved
at experiment start from the matching `TaskTemplate` YAML in the shared VR task templates directory.
Scene activation and Play Mode are driven over the editor MCP Bridge on a fixed loopback endpoint
(`127.0.0.1:8090`) and are deliberately NOT configured here. For the VR task driver that consumes this
configuration and drives the bridge, see `experiment:vr-driver-interface`.

For the full per-field documentation of the auxiliary sections, see
[`references/configuration-fields.md`](references/configuration-fields.md).

---

## Configuration authoring workflow

### Step 1: Verify prerequisites

- The `sle mcp` server is connected. If not, run `experiment:experiment-mcp-environment-setup`.
- The host working directory is set. If not, run `assets:working-directory`.

### Step 2: Determine whether to create or modify

Call `read_system_configuration_tool`. If it returns a configuration, you are modifying. If it
returns an empty/not-found response, you are creating from scratch.

### Step 3: Inspect the schema

Call `describe_system_configuration_schema_tool()`. The schema response gives canonical field
names, types, defaults, and units. Use this **and**
[`references/configuration-fields.md`](references/configuration-fields.md) as the source of truth —
do not guess field names.

### Step 4: Gather values from the user (creation case)

For a new host, the user-supplied values are:

- **Camera indices** — must come from `experiment:acquisition-system-setup`, which uses
  `ataraxis@video:camera-setup` for hardware discovery. Do NOT call camera discovery tools from
  this skill.
- **Microcontroller ports** — must come from `experiment:acquisition-system-setup`, which uses
  `ataraxis@communication:microcontroller-setup`. The user must confirm which physical Teensy
  plays the actor / sensor / encoder role.
- **Zaber motor ports** — must come from `experiment:zaber-interface` discovery
  (`get_zaber_devices_tool`).
- **Filesystem paths** — `mesoscope_directory` and the `storage_directories` destination paths.
  Ask the user for absolute paths. The platform data root is set separately via
  `slsa configure data-root` (`assets:working-directory`).
- **Google Sheet IDs** — ask the user for the sheet IDs (the long alphanumeric segment in the URL).
- **Calibration data** — defaults in
  [`references/configuration-fields.md`](references/configuration-fields.md) are reasonable
  starting points. Only override if the user has freshly measured values.

### Step 5: Write the configuration

```text
write_system_configuration_tool(
    configuration_payload={ ... full nested dict ... },
    overwrite=True,
)
```

The tool validates the dictionary against the schema and writes the YAML atomically.

### Step 6: Verify

Call `read_system_configuration_tool` and confirm the returned configuration matches what you
wrote. Then call `validate_system_configuration_tool` to confirm filesystem mounts and
hardware-port assumptions hold.

### Step 7: Hand off for server configuration

If the user is also setting up remote storage transfer, hand off to the forging plugin's
`forging:server-configuration` skill. This skill does not write the server configuration file.

---

## Modification workflows

### Reindex / re-port hardware

| Change                        | Section to mutate                                                   |
|-------------------------------|---------------------------------------------------------------------|
| Camera reindexed              | `cameras.face_camera_index` / `body_camera_index`                   |
| Teensy replaced / re-flashed  | `microcontrollers.actor_port` / `sensor_port` / `encoder_port`      |
| Zaber motor group reconnected | `assets.headbar_port` / `wheel_port` / `lickport_port`              |
| Unity broker relocated        | `assets.vr_task.ip` / `assets.vr_task.port`                         |
| Storage volume remounted      | `filesystem.storage_directories` / `filesystem.mesoscope_directory` |
| Google Sheet rotated          | `sheets.<sheet>_id`                                                 |

For each: read the configuration, mutate the relevant field, write back via
`write_system_configuration_tool`. No code changes needed.

### Recalibrate a module

| Change                        | Section to mutate                                                              |
|-------------------------------|--------------------------------------------------------------------------------|
| Valve recalibrated            | `microcontrollers.valve_calibration_data`                                      |
| Wheel diameter changed        | `microcontrollers.wheel_diameter_cm`                                           |
| Encoder PPR changed           | `microcontrollers.wheel_encoder_ppr`                                           |
| Lick threshold adjusted       | `microcontrollers.lick_threshold_adc` (and signal/delta if needed)             |
| Torque calibration updated    | `microcontrollers.torque_*` fields                                             |
| Brake strength bounds changed | `microcontrollers.minimum_brake_strength_g_cm` / `maximum_brake_strength_g_cm` |

For each: read, mutate, write. The calibration is consumed at session start when the binding class
instantiates the wrappers; existing sessions are unaffected.

### Add a new module to an existing microcontroller

This crosses repositories. Follow the workflow in `experiment:microcontroller-interface`'s
"Adding a paired Module + Interface" section first to author the firmware Module + Python wrapper.
Then in this skill:

1. **Add configuration fields** to `MesoscopeMicroControllers` for the new module's runtime
   parameters. Follow the `<device>_<parameter>_<unit>` naming convention. Update
   [`references/configuration-fields.md`](references/configuration-fields.md).
2. **Extend `MicroControllerInterfaces`** to instantiate the new wrapper inside the appropriate
   board's wrapper block (ACTOR / SENSOR / ENCODER) using the new configuration fields. Add it to
   the board's `module_interfaces` tuple.
3. **Extend `MicroControllerInterfaces.start()`** if the new wrapper needs `initialize_local_assets()`
   (i.e., uses `SharedMemoryArray`) or per-module runtime parameter pushes (i.e., has a
   `set_parameters()` method).
4. **Bump the `sollertia-experiment` version** in `pyproject.toml`.
5. **Regenerate the system configuration YAML** on every deployment so the new fields appear.
6. **Update this skill** — if the new module changes the boards' module inventory in the table at
   the top of [Hardware subsystem: Microcontrollers](#hardware-subsystem-microcontrollers), update that table.

### Add a new module that needs a new microcontroller board

Follow the workflow in `experiment:microcontroller-interface`'s "Adding a new controller board"
section to add the new target macro in slmc's `main.cpp`. Then in this skill:

1. **Add a new port field** to `MesoscopeMicroControllers` (e.g., `<role>_port`).
2. **Add a new construction block** in `MicroControllerInterfaces.__init__` for the new board,
   using the new port and a fresh `controller_id` not currently in use (101, 152, 203 are taken).
3. **Update this skill's hardware-subsystem table** to list the new board, its controller ID, role, and
   modules.
4. **Bump the `sollertia-experiment` version** and regenerate YAMLs.

### Add a new camera

1. **Add per-camera fields** to `MesoscopeCameras` following the existing `<role>_camera_<setting>`
   pattern, including an optional `<role>_camera_configuration_path: Path = Path()` for the camera's
   GenICam configuration YAML. Update [`references/configuration-fields.md`](references/configuration-fields.md).
2. **Extend `VideoSystems`** to instantiate a new `VideoSystem` with the new camera's parameters
   and a fresh `system_id` (51 and 62 are taken; pick a value that doesn't collide with the
   DataLogger source range used by other subsystems).
3. **Add per-camera lifecycle methods** (`start_<role>_camera`, `save_<role>_camera_frames`) and
   update `stop()` to include the new camera.
4. **Bump the `sollertia-experiment` version** and regenerate YAMLs.
5. **Update this skill's hardware-subsystem table.**

### Add a new Zaber motor group

1. **Add a `<group>_port` field** to `MesoscopeVRAssets`.
2. **Extend `ZaberMotors`** to instantiate a new `ZaberConnection` for the new port and pull out
   the per-axis `ZaberAxis` handles in the documented daisy-chain order.
3. **Add per-group position-restoration logic** to `restore_position()`, `prepare_motors()`, and
   `park_position()`.
4. **Update the `ZaberPositions` dataclass** in `sollertia_experiment/mesoscope_vr/system.py`
   to capture the new group's per-axis positions.
5. **Bump the `sollertia-experiment` version** and regenerate YAMLs.
6. **Update this skill's hardware-subsystem table.**

For motor-side mechanics (checksum validation, parking, position storage), see
`experiment:zaber-interface`.

---

## Lifecycle orchestrator handoff

The Mesoscope-VR runtime composes the three binding classes (`MicroControllerInterfaces`,
`VideoSystems`, `ZaberMotors`) inside `MesoscopeVRSystem` (in
`sollertia_experiment/mesoscope_vr/system_controller.py`), and also constructs the `VRTaskDriver`
(experiment sessions only) and the `MesoscopeDriver`. The orchestrator's responsibilities,
state machine, training modes, and CLI surface are documented in
`/mesoscope-vr-runtime`.

The handoff contract between this skill and the runtime skill:

- This skill owns the **hardware composition** (what binding classes exist, how they are
  constructed from configuration).
- The runtime skill owns the **temporal composition** (when binding classes start/stop, how
  state transitions happen, how training logic uses the binding classes).

A change to the binding classes (new wrapper, new construction parameter) is in scope for this
skill. A change to the runtime state machine or the CLI is in scope for the runtime skill.

---

## Maintenance contract

This skill is split:

- **SKILL.md** (this file) — durable Mesoscope-VR composition and workflows. Update when a
  hardware subsystem is added/removed, the system's binding-class structure changes, or a
  modification workflow's steps change.
- **[`references/configuration-fields.md`](references/configuration-fields.md)** — state snapshot
  of every dataclass field. Update whenever any configuration dataclass field is added, removed,
  renamed, or has its type/units changed.

Out-of-date field documentation is worse than missing documentation — an agent acting on stale
field info will write malformed YAML that fails schema validation or, worse, validates but
parameterizes the system incorrectly. When in doubt, re-read
`sollertia_experiment/mesoscope_vr/system.py` and reconcile the references file against
ground truth.

---

## Related skills

| Skill                                              | Relationship                                                                         |
|----------------------------------------------------|--------------------------------------------------------------------------------------|
| `experiment:acquisition-system-design`             | The platform-general pattern this system implements. Required reading.               |
| `experiment:microcontroller-interface`             | The slmc + sle wrapper layer the microcontroller binding class composes.             |
| `/mesoscope-vr-runtime`                  | Mesoscope-VR runtime behavior (state machine, training modes, CLI).                  |
| `experiment:zaber-interface`                       | Zaber motor mechanics consumed by `ZaberMotors`.                                     |
| `experiment:vr-driver-interface`                   | The Unity VR task driver (`VRTaskDriver`) configured by `assets.vr_task`.            |
| `ataraxis@video:camera-interface`                  | VideoSystem mechanics consumed by `VideoSystems`.                                    |
| `ataraxis@communication:microcontroller-interface` | MicroControllerInterface mechanics consumed by `MicroControllerInterfaces`.          |
| `experiment:acquisition-system-setup`              | Source of camera indices, microcontroller ports, Zaber ports via hardware discovery. |
| `/mesoscope-vr-snapshots`                | Per-session Zaber position snapshots consumed by `ZaberMotors.restore_position()`.   |
| `experiment:google-sheets-processing`              | Reads/writes the sheets identified by `MesoscopeGoogleSheets` (`surgery_sheet_id`, `water_log_sheet_id`). |
| `assets:working-directory`                 | Required prerequisite for configuration authoring.                                   |
| `forging:server-configuration`             | Sibling configuration file for remote storage transfer.                              |

---

## Verification checklist

```text
When authoring or modifying the Mesoscope-VR system configuration:

Prerequisites:
- [ ] assets:working-directory has been run on this host
- [ ] sle mcp server is connected
- [ ] describe_system_configuration_schema_tool was called and used as the source of truth for field names
- [ ] references/configuration-fields.md was consulted for field semantics

Authoring:
- [ ] Camera indices sourced from experiment:acquisition-system-setup, not guessed
- [ ] Microcontroller ports sourced from experiment:acquisition-system-setup, not guessed
- [ ] Zaber motor ports sourced from experiment:zaber-interface discovery, not guessed
- [ ] write_system_configuration_tool succeeded without schema errors
- [ ] read_system_configuration_tool returned the expected configuration after the write
- [ ] validate_system_configuration_tool reported all mounts healthy
- [ ] Did not call write_server_configuration_tool — handed off to forging:server-configuration if needed

Modifying:
- [ ] Hardware-subsystem modifications followed the cross-repo workflow (slmc + sle)
- [ ] Configuration field naming follows <device>_<parameter>_<unit>
- [ ] references/configuration-fields.md updated to reflect new/changed fields
- [ ] sollertia-experiment version bumped on any dataclass schema change
- [ ] Binding class or MesoscopeDriver extended for new wrappers / cameras / motor groups / acquisition parameters
- [ ] All affected deployments had their YAML regenerated
- [ ] This skill's hardware-subsystem tables updated if board/camera/motor/acquisition inventory changed
```
