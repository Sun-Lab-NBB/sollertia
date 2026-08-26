---
name: mesoscope-vr
description: >-
  Knowledge repository for the Mesoscope-VR data acquisition system: its hardware subsystem inventory, the
  MesoscopeSystemConfiguration dataclass and YAML lifecycle, the per-subsystem binding classes, and the
  hardware/calibration modification workflows. Use when configuring, modifying, or auditing Mesoscope-VR's hardware
  and configuration layer, or reading the frozen per-session system-configuration snapshot.
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
- `read_session_system_configuration_tool`, the reader for the frozen per-session system-configuration snapshot
- The `video_tracking` section that binds the acquisition stack to the sollertia-video-tracking (slvt) inference tool
- Configuration authoring and modification workflows

**Does not cover** (delegated):
- The platform-general design pattern this system implements — see `experiment:acquisition-system-design`
- Mesoscope-VR runtime behavior (state machine, training modes, visualizers, CLI commands) — see
  `/mesoscope-vr-runtime`
- Session descriptor field schemas — see `/mesoscope-vr-session-schema`, and the runtime behavior that populates
  them — see `/mesoscope-vr-runtime`
- Per-firmware-module Python wrappers and slmc firmware Modules — see `experiment:microcontroller-interface`
- Low-level VideoSystem API — see `video:camera-interface` (ataraxis marketplace)
- Low-level Zaber motor API — see `experiment:zaber-interface`
- Low-level MicroControllerInterface API — see `communication:microcontroller-interface`
- Per-session metadata, task templates, and experiment configuration — see `assets:session-data`,
  `assets:task-templates`, and `assets:experiment-configuration`
- Server transfer configuration — see `forging:server-configuration`

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

| Concern                                        | Authority                                 |
|------------------------------------------------|-------------------------------------------|
| Pattern this system implements                 | `experiment:acquisition-system-design`    |
| Per-firmware-module wrappers (slmc + sle pair) | `experiment:microcontroller-interface`    |
| Low-level VideoSystem API                      | `video:camera-interface`                  |
| Low-level Zaber motor API                      | `experiment:zaber-interface`              |
| Low-level MicroControllerInterface API         | `communication:microcontroller-interface` |

This skill documents only Mesoscope-VR-specific composition. All base mechanics are inherited from
the skills above.

---

## MesoscopeSystemConfiguration

The system's configuration is captured by `MesoscopeSystemConfiguration`, a `SystemConfiguration`-derived
dataclass defined in `sollertia_experiment/mesoscope_vr/system.py`.

### Top-level structure

`MesoscopeSystemConfiguration` composes seven nested dataclasses plus a top-level `name` field:

| Section            | Dataclass                   | What it parameterizes                                                         |
|--------------------|-----------------------------|-------------------------------------------------------------------------------|
| `name`             | (str, top-level)            | Human-readable system label (default: `"mesoscope"`)                          |
| `filesystem`       | `MesoscopeFileSystem`       | Mesoscope-DAQ acquisition mount plus the named long-term storage destinations |
| `sheets`           | `MesoscopeGoogleSheets`     | Google Sheet IDs for surgery log and water log                                |
| `cameras`          | `MesoscopeCameras`          | Face / body camera indices and H.265 encoding parameters                      |
| `microcontrollers` | `MesoscopeMicroControllers` | Per-board ports + per-module calibration data                                 |
| `acquisition`      | `MesoscopeAcquisition`      | Mesoscope motion-estimation and z-stack acquisition parameters                |
| `assets`           | `MesoscopeVRAssets`         | Zaber motor ports + nested `vr_task` Unity MQTT configuration                 |
| `video_tracking`   | `MesoscopeVideoTracking`    | DeepLabCut face-camera pose-inference environment and parameters              |

For the full field-by-field registry (every field name, type, default, units, and meaning), see
[`references/configuration-fields.md`](references/configuration-fields.md). That file is a state
snapshot of the current Mesoscope-VR schema and MUST be updated whenever any dataclass field is
added, removed, or renamed.

### YAML file lifecycle

The configuration is persisted as `mesoscope_system_configuration.yaml` in the platform configuration directory
resolved by `get_system_configuration_path()`. The host's **working** directory must be set first via
`assets:working-directory` (`slsa configure directory`), since the file resolves to
`<working_directory>/configuration/mesoscope_system_configuration.yaml`.

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
`sle mesoscope configure system`. `configure` is a click group, so running it bare prints the group help and writes
nothing, and every error message the source raises names the full `sle mesoscope configure system` form.

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

The write tool accepts the full nested dictionary that maps onto the dataclass tree. It does NOT refuse a partial
payload. Unknown keys and annotation violations are rejected by name, but every field the payload omits takes its
dataclass default and the write persists that default, silently resetting configuration the caller never intended to
touch. Because omissions are silent, every amendment MUST be a read-mutate-write of the complete record: read the
configuration first, mutate the returned dictionary, then write the whole thing back.

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

`start()` brings up all three controllers in order (actor → sensor → encoder), calls `initialize_local_assets()` on the
wrappers backed by a SharedMemoryArray (`wheel_encoder`, `valve`, `gas_puff_valve`, `mesoscope_frame`, `lick`), then
pushes runtime parameters via `set_parameters()` to the five wrappers that accept them: `wheel_encoder`, `screens`,
`lick`, `torque`, and `mesoscope_frame`. `brake` and `valve` are parameterized at construction instead.

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

For the full per-field documentation, see [`references/configuration-fields.md`](references/configuration-fields.md)
under the "MesoscopeCameras" section.

### Camera GenICam configuration: verify, dump, restore

The verify / dump / restore workflow for a camera's stored GenICam node configuration is documented in
[`references/modification-workflows.md`](references/modification-workflows.md).

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
`video:camera-interface`.

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

Method surface:

| Method                         | Role                                                                                         |
|--------------------------------|----------------------------------------------------------------------------------------------|
| `prepare_motors()`             | Homes all seven axes in parallel, establishing the reference every other movement needs      |
| `park_position()`              | Moves every axis to its parking position, the shutdown pose that supports the next homing    |
| `maintenance_position()`       | Moves every axis to the Mesoscope-VR system maintenance pose                                 |
| `mount_position()`             | Positions the motors to facilitate mounting the animal into the enclosure                    |
| `unmount_position()`           | Retracts only the LickPort axes to their mounting positions, holding all other axes in place |
| `generate_position_snapshot()` | Queries all seven axes, caches and returns a `ZaberPositions` snapshot                       |
| `unpark_motors()`              | Unparks every axis, allowing movement through this library and the Zaber GUI                 |
| `park_motors()`                | Parks every axis, blocking movement until it is unparked again                               |
| `wait_until_idle()`            | Blocks in place while at least one managed axis is still moving                              |
| `disconnect()`                 | Shuts down all managed motors and closes the three motor-group connections                   |
| `is_connected`                 | Property. True only when all three motor-group connections are active                        |
| `restore_position()`           | Restores every axis to its previous-runtime position, or to the stored mount / park pose     |

`mount_position()` carries non-obvious semantics. Only the LickPort motors always move to their mounting positions.
When previous-runtime positions are available, the HeadBar and Wheel motors are restored to those positions instead,
because mounting is facilitated primarily by moving the LickPort away from the animal.

For when the runtime invokes these methods and in what order, see `/mesoscope-vr-runtime`.

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
high-definition reference z-stack that the ScanImagePC generates at the start of each runtime.

For the full per-field documentation (types, defaults, units, and `__post_init__` validation), see
[`references/configuration-fields.md`](references/configuration-fields.md) under the
"MesoscopeAcquisition" section.

### MesoscopeDriver interface class

`MesoscopeDriver` (in `sollertia_experiment/mesoscope_vr/mesoscope_driver.py`) encapsulates all MQTT
communication with the `runAcquisition` MATLAB function on the ScanImagePC. Its construction
signature, method surface, command-acknowledgement semantics, MQTT topic namespace, per-command
payload contents, the `runAcquisition` MATLAB counterpart, and the pre-flight bridge check that
confirms that counterpart is running are documented in
[`references/mesoscope-driver.md`](references/mesoscope-driver.md).

For when the orchestrator invokes these methods within the runtime state machine, see
`/mesoscope-vr-runtime`.

---

## Auxiliary configuration sections

`MesoscopeFileSystem` and the `MesoscopeData` path resolver it feeds, `MesoscopeGoogleSheets`, the
Unity-VR portion of `MesoscopeVRAssets`, and `MesoscopeVideoTracking` are documented field by field in
[`references/configuration-fields.md`](references/configuration-fields.md).

---

## Workflows

Authoring a configuration on a new host, verifying a camera's GenICam configuration, reindexing or
re-porting hardware, recalibrating a module, adding a new module to an existing microcontroller,
adding a new module that needs a new microcontroller board, adding a new camera, and adding a new
Zaber motor group are documented in
[`references/modification-workflows.md`](references/modification-workflows.md).

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

- **SKILL.md** (this file) — durable Mesoscope-VR composition. Update when a hardware subsystem is
  added/removed or the system's binding-class structure changes.
- **[`references/configuration-fields.md`](references/configuration-fields.md)** — state snapshot
  of every dataclass field. Update whenever any configuration dataclass field is added, removed,
  renamed, or has its type/units changed.
- **[`references/mesoscope-driver.md`](references/mesoscope-driver.md)** — the `MesoscopeDriver` construction and
  method surface, the ScanImagePC MQTT contract, and the pre-flight bridge check. Update whenever the driver's
  surface, a topic, a command payload, or the bridge-check surface changes.
- **[`references/modification-workflows.md`](references/modification-workflows.md)** — the configuration
  authoring, hardware, and calibration workflows. Update whenever a workflow's steps change.

Out-of-date field documentation is worse than missing documentation — an agent acting on stale
field info will write malformed YAML that fails schema validation or, worse, validates but
parameterizes the system incorrectly. When in doubt, re-read
`sollertia_experiment/mesoscope_vr/system.py` and reconcile the references file against
ground truth.

---

## Related skills

| Skill                                     | Relationship                                                                                    |
|-------------------------------------------|-------------------------------------------------------------------------------------------------|
| `experiment:acquisition-system-design`    | The platform-general pattern this system implements. Required reading.                          |
| `experiment:microcontroller-interface`    | The slmc + sle wrapper layer the microcontroller binding class composes.                        |
| `/mesoscope-vr-runtime`                   | Mesoscope-VR runtime behavior (state machine, training modes, CLI).                             |
| `/mesoscope-vr-session-schema`            | Field schemas of the per-session descriptors this skill delegates.                              |
| `experiment:zaber-interface`              | Zaber motor mechanics consumed by `ZaberMotors`.                                                |
| `experiment:vr-driver-interface`          | The Unity VR task driver (`VRTaskDriver`) configured by `assets.vr_task`.                       |
| `video:camera-interface`                  | VideoSystem mechanics consumed by `VideoSystems`.                                               |
| `communication:microcontroller-interface` | MicroControllerInterface mechanics consumed by `MicroControllerInterfaces`.                     |
| `experiment:acquisition-system-setup`     | Source of camera indices, microcontroller ports, Zaber ports via hardware discovery.            |
| `/mesoscope-vr-snapshots`                 | Per-session Zaber position snapshots consumed by `ZaberMotors.restore_position()`.              |
| `experiment:google-sheets-processing`     | Reads and writes the `MesoscopeGoogleSheets` sheets (`surgery_sheet_id`, `water_log_sheet_id`). |
| `assets:working-directory`                | Required prerequisite for configuration authoring.                                              |
| `forging:server-configuration`            | Sibling configuration file for remote storage transfer.                                         |
| `assets:project-hierarchy`                | The on-disk hierarchy `MesoscopeData` resolves session paths within.                            |
| `experiment:data-management`              | Transfer and removal workflows that consume the resolved storage destinations.                  |

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
- [ ] video_tracking.conda_environment confirmed by hand, since it is the only video_tracking field outside the report
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
