---
name: mesoscope-vr
description: >-
  Knowledge repository for the Mesoscope-VR data acquisition system: its hardware subsystem inventory, the
  MesoscopeSystemConfiguration dataclass and YAML lifecycle, the per-subsystem binding classes, and the
  hardware/calibration modification workflows. Use when configuring, modifying, or auditing Mesoscope-VR's hardware and
  configuration layer, or reading the frozen per-session system-configuration snapshot.
user-invocable: false
---

# Mesoscope-VR system

Knowledge repository for the Mesoscope-VR data acquisition system, currently the only system the Sollertia platform
supports. Mesoscope-VR is also the worked example an agent copies when building a new acquisition system, so every
section below names the platform-general seam that its Mesoscope-VR choice fills. The seam catalog itself belongs to
`experiment:library-extension`, the platform-general design pattern to `experiment:acquisition-system-design`, and the
runtime behavior (state machine, training modes, CLI) to `/mesoscope-vr-runtime`.

---

## Scope

**Covers:**
- Mesoscope-VR system overview (hardware composition, controllers, cameras, motors, Mesoscope acquisition), and the
  seam map from each Mesoscope-VR choice back to the platform-general seam it fills
- `MesoscopeSystemConfiguration`, its registration call, the configuration file lifecycle, and the per-section field
  registry in `references/configuration-fields.md`
- `MesoscopeData`, the class that resolves the `filesystem` section into per-session paths
- Per-subsystem binding classes (`MicroControllerInterfaces`, `VideoSystems`, `ZaberMotors`) and the `MesoscopeDriver`
  out-of-band device driver, covering composition and lifecycle wiring
- MCP tools for reading, writing, and validating the configuration YAML, plus `read_session_system_configuration_tool`
  for the frozen per-session snapshot
- The `video_tracking` section that binds the acquisition stack to the sollertia-video-tracking (slvt) inference tool
- Configuration authoring and modification workflows

**Does not cover:**
- The platform-general design pattern this system implements. See `experiment:acquisition-system-design`
- The extension seams a new acquisition system touches across the four repositories. See `experiment:library-extension`
- Mesoscope-VR runtime behavior (state machine, training modes, visualizers, `sle mesoscope` CLI commands) and the
  session descriptors it populates. See `/mesoscope-vr-runtime` and `/mesoscope-vr-session-schema`
- Per-firmware-module Python wrappers and slmc firmware Modules. See `experiment:microcontroller-interface`
- Low-level VideoSystem, Zaber, and MicroControllerInterface APIs. See `video:camera-interface`,
  `experiment:zaber-interface`, and `communication:microcontroller-interface`
- Per-session metadata, task templates, and experiment configuration. See `assets:session-data`,
  `assets:task-templates`, and `assets:experiment-configuration`
- The remote compute server's SSH and SLURM access configuration. See `forging:server-configuration`

---

## System overview

Mesoscope-VR is a head-fixed 2-Photon Random Access Mesoscope (2P-RAM) imaging system with a Virtual Reality
environment. It spans two machines. The VRPC runs the acquisition runtime and owns every subsystem in the table below.
The ScanImagePC runs the ScanImage software and the `runAcquisition` MATLAB command loop that drives the microscope, and
it exports its data-output directory as a local mount that the VRPC reads through `filesystem.mesoscope_directory`. That
field is therefore the ScanImagePC data share as seen from the VRPC, and `MesoscopeData` raises `ValueError` when it is
left unset (`mesoscope_vr/system.py`).

| Subsystem             | Devices                                                            | Binding or driver class             |
|-----------------------|--------------------------------------------------------------------|-------------------------------------|
| Microcontrollers      | 3 × Teensy 4.1 boards (ACTOR, SENSOR, ENCODER)                     | `MicroControllerInterfaces`         |
| Cameras               | 2 × GenICam scientific cameras (face camera, body camera)          | `VideoSystems`                      |
| Zaber motors          | 3 × motor groups (HeadBar Z/Pitch/Roll, Wheel X, LickPort Z/Y/X)   | `ZaberMotors`                       |
| Unity VR (MQTT)       | 1 × MQTT broker bridging the runtime to the Unity game engine      | (consumed directly by orchestrator) |
| Mesoscope acquisition | ScanImage 2P-RAM Mesoscope driven over MQTT via `runAcquisition.m` | `MesoscopeDriver`                   |

The lifecycle orchestrator (`MesoscopeVRSystem` in `mesoscope_vr/system_controller.py`) composes the three binding
classes and drives the system state machine. The orchestrator is documented in `/mesoscope-vr-runtime`.

### Worked-example seam map

| Platform-general seam                       | The Mesoscope-VR choice that fills it                                                          |
|---------------------------------------------|------------------------------------------------------------------------------------------------|
| `AcquisitionSystems` member                 | `MESOSCOPE_VR = "mesoscope"` (`sollertia-shared-assets/src/sollertia_shared_assets/enums.py`)  |
| System configuration class and registration | `MesoscopeSystemConfiguration`, registered at import time (`mesoscope_vr/system.py`)           |
| Hardware binding classes                    | `MicroControllerInterfaces`, `VideoSystems`, `ZaberMotors` (`mesoscope_vr/binding_classes.py`) |
| Out-of-band device driver                   | `MesoscopeDriver`, an MQTT client for the ScanImagePC (`mesoscope_vr/mesoscope_driver.py`)     |
| VR task configuration seam                  | `assets.vr_task`, consumed by `VRTaskDriver` and reused for mesoscope broker discovery         |
| Runtime orchestrator                        | `MesoscopeVRSystem` (`mesoscope_vr/system_controller.py`), see `/mesoscope-vr-runtime`         |
| Per-system CLI group                        | `sle mesoscope`, see `/mesoscope-vr-cli-reference`                                             |
| Per-system MCP tool module                  | `interfaces/mesoscope_vr_tools.py`, see `/mesoscope-vr-runtime`                                |

The acquisition engine is written per system by design rather than derived from a runtime base class, and this package
is the worked example from which a new system's engine is scaffolded. `experiment:library-extension` owns that design
position in its "The acquisition engine is a human-in-the-loop rewrite" section, along with the full seam list.

---

## Authoritative bases

Read these skills before reading the hardware-subsystem sections below:

| Concern                                        | Authority                              |
|------------------------------------------------|----------------------------------------|
| Pattern this system implements                 | `experiment:acquisition-system-design` |
| Seam catalog for a new acquisition system      | `experiment:library-extension`         |
| Per-firmware-module wrappers (slmc + sle pair) | `experiment:microcontroller-interface` |

This skill documents only Mesoscope-VR-specific composition. All base mechanics are inherited from the skills above
and from the low-level device APIs listed in Related skills.

---

## MesoscopeSystemConfiguration

The system's configuration is captured by `MesoscopeSystemConfiguration`, a `SystemConfiguration`-derived dataclass
defined in `mesoscope_vr/system.py`.

### Top-level structure

`MesoscopeSystemConfiguration` composes seven nested dataclasses plus a top-level `name` field:

| Section            | Dataclass                   | What it parameterizes                                                       |
|--------------------|-----------------------------|-----------------------------------------------------------------------------|
| `name`             | (str, top-level)            | Human-readable system label (default: `"mesoscope"`)                        |
| `filesystem`       | `_MesoscopeFileSystem`      | ScanImagePC acquisition mount plus the named long-term storage destinations |
| `sheets`           | `MesoscopeGoogleSheets`     | Google Sheet IDs for surgery log and water log                              |
| `cameras`          | `MesoscopeCameras`          | Face / body camera indices and H.265 encoding parameters                    |
| `microcontrollers` | `MesoscopeMicroControllers` | Per-board ports + per-module calibration data                               |
| `acquisition`      | `MesoscopeAcquisition`      | Mesoscope motion-estimation and z-stack acquisition parameters              |
| `assets`           | `MesoscopeVRAssets`         | Zaber motor ports + nested `vr_task` Unity MQTT configuration               |
| `video_tracking`   | `MesoscopeVideoTracking`    | DeepLabCut face-camera pose-inference environment and parameters            |

For the full field-by-field registry (every field name, type, default, units, and meaning), see
[`references/configuration-fields.md`](references/configuration-fields.md).

### Registration and package exports

`register_system_configuration(system=AcquisitionSystems.MESOSCOPE_VR,
configuration_class=MesoscopeSystemConfiguration)` runs at module scope, so the class registers at import time
(`mesoscope_vr/system.py`). Registration is the only system-specific wiring the shared file lifecycle requires, and the
package adds one typed accessor on top of it. `get_system_configuration()` calls the shared
`get_system_configuration_data()` and raises `TypeError` when the host belongs to another system
(`mesoscope_vr/system.py`). The package re-exports `get_system_configuration_path` from `cross_system` unchanged
(`mesoscope_vr/__init__.py`), and the `__all__` list of that module holds 19 names.

### YAML file lifecycle

The configuration is persisted as `mesoscope_system_configuration.yaml` in the platform configuration directory
resolved by `get_system_configuration_path()`, because the shared helper derives the filename from the enum value.
The host's **working** directory must be set first via `assets:working-directory` (`slsa configure directory`), since
the file resolves to `<working_directory>/configuration/mesoscope_system_configuration.yaml`.

`MesoscopeSystemConfiguration` implements two non-default behaviors documented in
`experiment:acquisition-system-design`:

- **`__post_init__`** normalizes the valve calibration table from its YAML-natural `dict` shape to the in-memory
  `tuple[tuple[int | float, int | float], ...]` shape that `WaterValveInterface` consumes. It also validates that every
  entry is a 2-tuple of numeric values, and a malformed entry raises `TypeError` (`mesoscope_vr/system.py`).
- **`save()`** override temporarily converts the valve calibration table back to a `dict` before writing to YAML so
  existing files retain the mapping layout, then restores the tuple form in a `finally` block
  (`mesoscope_vr/system.py`).

Creating the file (`create_system_configuration_file()`), resolving its path (`get_system_configuration_path()`), and
loading it are handled by the shared `cross_system` helpers, following the pattern in
`experiment:acquisition-system-design`. The CLI entry point for authoring is `sle mesoscope configure system`, which
takes no options and whose full surface belongs to `/mesoscope-vr-cli-reference`.

### Experiment configuration files

`create_experiment_configuration_file(project, experiment, template, state_count, reward_size, reward_tone_duration,
puff_duration, *, overwrite=False)` builds a `MesoscopeExperimentConfiguration` through `from_task_template(...)` and
writes it to `<project>/<CONFIGURATION_DIRECTORY>/<experiment>.yaml` (`mesoscope_vr/system.py`). It raises
`ValueError` when the project does not exist under the data root, `FileExistsError` when the destination file exists
and `overwrite` is False, and `FileNotFoundError` when the named task template is absent from the task templates
directory. The field schema of the produced file belongs to `/mesoscope-vr-experiment-schema`, and template authoring
to `assets:task-templates`.

### MCP tool surface

The configuration's read, write, and validate surface is hosted on `sle mcp` (`sollertia-experiment`). This skill is
the **exclusive** owner of `write_system_configuration_tool`, which no other skill in the marketplace may call.

| Tool                                        | Purpose                                                                  |
|---------------------------------------------|--------------------------------------------------------------------------|
| `describe_system_configuration_schema_tool` | Returns the field schema for `MesoscopeSystemConfiguration`              |
| `read_system_configuration_tool`            | Reads the active system configuration YAML from the working directory    |
| `write_system_configuration_tool`           | Writes a new system configuration YAML (**exclusive to this skill**)     |
| `validate_system_configuration_tool`        | Validates the loaded system configuration and reports mount status       |
| `verify_camera_configuration_tool`          | Diffs each camera's live GenICam configuration against its stored config |
| `check_system_mounts_tool`                  | Checks every filesystem path declared in the configuration               |
| `check_mesoscope_bridge_tool`               | Probes the ScanImagePC `runAcquisition` command loop                     |
| `read_session_system_configuration_tool`    | Reads the frozen system configuration captured at session start          |

These eight are part of the fifteen Mesoscope-VR tools of `interfaces/mesoscope_vr_tools.py`. The remaining seven are
session-lifecycle and snapshot tools owned by `/mesoscope-vr-runtime` and `/mesoscope-vr-snapshots`. The companion
enumeration `list_supported_acquisition_systems_tool` lives on `slsa mcp` (`sollertia-shared-assets`) since the
`AcquisitionSystems` enum remains the shared vocabulary across the platform.

The write tool accepts the full nested dictionary that maps onto the dataclass tree. It does NOT refuse a partial
payload. Unknown keys and annotation violations are rejected by name, but every field the payload omits takes its
dataclass default and the write persists that default, silently resetting configuration the caller never intended to
touch. Because omissions are silent, every amendment MUST be a read-mutate-write of the complete record: read the
configuration first, mutate the returned dictionary, then write the whole thing back.

---

## Hardware subsystem: microcontrollers

The Mesoscope-VR system uses **three Teensy 4.1 microcontrollers** in dedicated roles:

| Board   | Controller ID | PlatformIO environment | Role                                             | Modules as `(module_type, module_id)`                                |
|---------|---------------|------------------------|--------------------------------------------------|----------------------------------------------------------------------|
| ACTOR   | 101           | `teensy41_actor`       | Output control (irregular, command-driven)       | brake (3,1), reward valve (5,1), gas-puff valve (5,2), screens (7,1) |
| SENSOR  | 152           | `teensy41_sensor`      | Input sensing (regular polling, no interrupts)   | mesoscope-frame TTL (1,1), lick sensor (4,1), torque (6,1)           |
| ENCODER | 203           | `teensy41_encoder`     | Quadrature encoder (hardware-interrupt-isolated) | wheel encoder (2,1)                                                  |

Each board takes the firmware of the environment its row names, uploaded from the slmc repository with
`pio run -e <environment> -t upload` while that board is the only microcontroller connected to the host. The
experimenter runs every upload. `experiment:microcontroller-interface` carries the procedure and the two rules an
upload follows in its `references/slmc-conventions.md`.

Board allocation reasoning is documented in `experiment:microcontroller-interface`'s "Controller board allocation
principles" section. The Mesoscope-VR three-board split is one valid application of the platform-general allocation
rules, driven primarily by interrupt isolation for the encoder.

The ACTOR board is where the multi-`module_id` pattern appears. `ValveModule` type 5 is instantiated twice on it,
once as the reward valve at id 1 and once as the gas-puff valve at id 2. The SENSOR board's only TTL instance is
input-only and records the frame clock that the ScanImagePC produces, which is why its wrapper is
`MesoscopeFrameTTLInterface` rather than the generic bidirectional wrapper. When the runtime enables and disables that
recorder relative to mesoscope acquisition is owned by `/mesoscope-vr-runtime`.

### MicroControllerInterfaces binding class

`MicroControllerInterfaces` (in `mesoscope_vr/binding_classes.py`) composes the three `MicroControllerInterface`
instances and the eight Python wrappers from `cross_system/module_interfaces.py`.

Construction signature: `MicroControllerInterfaces(data_logger: DataLogger, microcontroller_configuration:
MesoscopeMicroControllers)`.

The constructor instantiates wrappers in three blocks (ACTOR, SENSOR, ENCODER) using the configuration dataclass
fields, then wraps each block in a `MicroControllerInterface` with the corresponding controller ID, port, a
`buffer_size` of 8192, and the configured `keepalive_interval_ms`. Wrappers are exposed as public attributes:

- ACTOR: `self.brake`, `self.valve`, `self.gas_puff_valve`, `self.screens`
- SENSOR: `self.mesoscope_frame`, `self.lick`, `self.torque`
- ENCODER: `self.wheel_encoder`

`start()` sets `_started = True` **before** bringing the controllers up, so a failure partway through still routes
through `stop()` and tears down whichever controllers already started (`mesoscope_vr/binding_classes.py`). It then
starts the three controllers in order (actor, sensor, encoder) and calls `initialize_local_assets()` on the five
wrappers backed by a SharedMemoryArray (`wheel_encoder`, `valve`, `gas_puff_valve`, `mesoscope_frame`, `lick`). Finally,
it pushes runtime parameters via `set_parameters()` to the five wrappers that accept them: `wheel_encoder`, `screens`,
`lick`, `torque`, and `mesoscope_frame`. `brake` and `valve` are parameterized at construction instead, and
`gas_puff_valve` takes no configuration at all.

`stop()` returns immediately when the interfaces never started, then tears the three controllers down in the same
forward order, each call wrapped in `run_shutdown_step` so a failing teardown cannot strand the controllers that
follow it. `_started` clears only after all three steps, leaving the instance stoppable on a retry
(`mesoscope_vr/binding_classes.py`).

For per-wrapper mechanics, the cross-side contract with the firmware, and the slmc + sle convention details, see
`experiment:microcontroller-interface`.

---

## Hardware subsystem: cameras

The Mesoscope-VR system uses **two GenICam scientific cameras** (Harvester-managed) for animal behavior recording:

| Camera      | System ID | Role                            |
|-------------|-----------|---------------------------------|
| face camera | 51        | Animal's face and eye           |
| body camera | 62        | Animal's body (running posture) |

The system IDs are the `system_id` arguments of the two `VideoSystem` instances built by the `VideoSystems`
constructor (`mesoscope_vr/binding_classes.py`), and they are the DataLogger source IDs the camera logs carry.

### VideoSystems binding class

`VideoSystems` (in `mesoscope_vr/binding_classes.py`) composes the two `VideoSystem` instances.

Construction signature: `VideoSystems(data_logger: DataLogger, camera_configuration: MesoscopeCameras,
output_directory: Path)`.

Both cameras are instantiated with `CameraInterfaces.HARVESTERS`, `VideoEncoders.H265`, `OutputPixelFormats.YUV420`
(the rig's cameras are monochrome), and `gpu=0` for hardware-accelerated encoding. Per-camera parameters (index,
display rate, quantization, preset) come from the configuration.

The binding class exposes per-camera lifecycle granularity rather than a single `start()`:

- `start_face_camera()` / `start_body_camera()` begin frame acquisition without saving
- `save_face_camera_frames()` / `save_body_camera_frames()` begin saving frames to disk
- `stop()` stops both acquisition and saving for all cameras

This split exists because acquisition starts early in the session, before the operator-driven setup, while saving
starts only after the pre-start checkpoint completes. `stop()` disables frame saving per camera through
`run_shutdown_step`, waits `_CAMERA_DRAIN_DELAY_S = 2` seconds so the frames still in flight reach the saver
processes, and only then stops each camera (`mesoscope_vr/binding_classes.py`).

For VideoSystem mechanics, encoding configuration, and frame acquisition patterns, see `video:camera-interface`.

---

## Hardware subsystem: Zaber motors

The Mesoscope-VR system uses **three Zaber motor groups**, seven axes in total, for positioning the headbar, wheel,
and lickport:

| Group    | Daisy-chain      | Axes                                        | Purpose                             |
|----------|------------------|---------------------------------------------|-------------------------------------|
| HeadBar  | Z → Pitch → Roll | 3-axis (Z linear + Pitch + Roll rotational) | Animal head positioning             |
| Wheel    | X only           | 1-axis (X linear)                           | Running wheel longitudinal position |
| LickPort | Z → Y → X        | 3-axis (Z, Y, X linear)                     | Lickport spatial positioning        |

Each group connects to its own USB serial port. The binding class binds each axis by its daisy-chain index, so the
hardware cabling MUST match the order above, which the `ZaberMotors` constructor assumes
(`mesoscope_vr/binding_classes.py`).

### ZaberMotors binding class

`ZaberMotors` (in `mesoscope_vr/binding_classes.py`) composes three `ZaberConnection` instances and holds the
seven per-axis `ZaberAxis` handles as private attributes, exposing them only through the pose methods below.

Construction signature: `ZaberMotors(zaber_configuration: MesoscopeVRAssets, zaber_positions: ZaberPositions |
None)`.

This binding class takes no `DataLogger`, because Zaber motor state is per-session position data captured by the
`/mesoscope-vr-snapshots` skill rather than a real-time logged event stream.

The constructor connects to all three ports and retrieves the per-axis handles by daisy-chain index. It warns when no
previous-runtime positions were supplied, in which case the motors fall back to the mounting and parking positions
stored in each motor controller's non-volatile memory.

Method surface:

| Method                         | Role                                                                                         |
|--------------------------------|----------------------------------------------------------------------------------------------|
| `prepare_motors()`             | Unparks, homes all seven axes in parallel, waits for idle, then parks again                  |
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

For motor mechanics (park/unpark safety, position management, configuration tooling, the `ZaberConnection` /
`_ZaberDevice` / `ZaberAxis` hierarchy, MCP discovery), see `experiment:zaber-interface`.

---

## Hardware subsystem: Mesoscope acquisition

The Mesoscope-VR system images through a ScanImage-controlled 2P-RAM Mesoscope hosted on the ScanImagePC. The
acquisition runtime does not drive ScanImage directly: it publishes commands over MQTT to the `runAcquisition` MATLAB
function, which configures the online motion-estimation reference, acquires the high-definition reference z-stack,
and arms and runs frame acquisition. The Mesoscope frame stream is observed on the VRPC side as the SENSOR board's
mesoscope-frame TTL rather than over MQTT.

### MesoscopeAcquisition dataclass

`MesoscopeAcquisition` (in `mesoscope_vr/system.py`) is the `acquisition` field of `MesoscopeSystemConfiguration`. It
parameterizes the reference motion estimator and the high-definition reference z-stack that the ScanImagePC generates at
the start of each runtime.

For the full per-field documentation (types, defaults, units, and `__post_init__` validation), see
[`references/configuration-fields.md`](references/configuration-fields.md) under the "MesoscopeAcquisition" section.

### MesoscopeDriver out-of-band device driver

`MesoscopeDriver` (in `mesoscope_vr/mesoscope_driver.py`) encapsulates all MQTT communication with the `runAcquisition`
MATLAB function on the ScanImagePC. Its construction signature, method surface, command-acknowledgement semantics, MQTT
topic namespace, status states, per-command payload contents, the `runAcquisition` MATLAB counterpart, and the
pre-flight bridge check that confirms that counterpart is running are documented in
[`references/mesoscope-driver.md`](references/mesoscope-driver.md).

For when the orchestrator invokes these methods within the runtime state machine, see `/mesoscope-vr-runtime`.

### Operational note

Every mesoscope hardware action is either commanded over MQTT to the experimenter-launched `runAcquisition` MATLAB
loop (`MesoscopeDriver.await_alive` in `mesoscope_vr/mesoscope_driver.py`) or gated behind an operator prompt
(`setup_zaber_motors`, `reset_zaber_motors`, and `setup_mesoscope` in `mesoscope_vr/acquisition_components.py`).
Calibration and direct hardware control are experimenter-operated through the maintenance runtime GUI (the
`MaintenanceControlUI` block of `maintenance_logic` in `mesoscope_vr/data_acquisition.py`). You MUST NOT drive
mesoscope, valve, or motor hardware yourself. Direct the experimenter to the maintenance runtime instead.

---

## Auxiliary sections and workflows

`_MesoscopeFileSystem` with the `MesoscopeData` path resolver it feeds, `MesoscopeGoogleSheets`, the Unity-VR portion
of `MesoscopeVRAssets`, and `MesoscopeVideoTracking` are documented field by field in
[`references/configuration-fields.md`](references/configuration-fields.md).

[`references/modification-workflows.md`](references/modification-workflows.md) documents authoring a configuration on a
new host, verifying, dumping, and restoring a camera's GenICam configuration, and reindexing or re-porting hardware. It
also documents recalibrating a module, adding a module to an existing or to a new microcontroller board, adding a
camera, and adding a Zaber motor group.

---

## Lifecycle orchestrator handoff

`MesoscopeVRSystem` also constructs the `VRTaskDriver` (experiment sessions only) and the `MesoscopeDriver` (every
session, connected only for experiments).

This skill owns the **hardware composition**: which binding classes exist, and how they are constructed from
configuration. `/mesoscope-vr-runtime` owns the **temporal composition**: when binding classes start and stop, how state
transitions happen, and how the training logic uses them. A change to the binding classes (new wrapper, new construction
parameter) is in scope here. A change to the runtime state machine or the CLI is in scope there.

---

## Maintenance contract

Each file of this skill carries its own update trigger:

- **SKILL.md** (this file). Update when a hardware subsystem is added or removed, or when the binding-class
  structure changes.
- **[`references/configuration-fields.md`](references/configuration-fields.md)**. Update whenever a configuration
  dataclass field is added, removed, renamed, or has its type or units changed.
- **[`references/mesoscope-driver.md`](references/mesoscope-driver.md)**. Update whenever the driver's method
  surface, a topic, a status state, a command payload, or the bridge-check surface changes.
- **[`references/modification-workflows.md`](references/modification-workflows.md)**. Update whenever a workflow's
  steps change.

Stale field documentation is more damaging than missing documentation, because an agent acting on it writes malformed
YAML that either fails schema validation or validates and parameterizes the system incorrectly. When in doubt, re-read
`mesoscope_vr/system.py` and reconcile the references file against it.

---

## Related skills

The `video:` and `communication:` entries below resolve through the ataraxis marketplace. Every other entry resolves
inside the sollertia marketplace.

| Skill                                     | Relationship                                                                                    |
|-------------------------------------------|-------------------------------------------------------------------------------------------------|
| `experiment:external-tool-bindings`       | Owns the binding convention the `video_tracking` section instantiates                           |
| `/mesoscope-vr-cli-reference`             | Owns the `sle mesoscope configure system` and `configure experiment` command surface            |
| `experiment:acquisition-system-design`    | The platform-general pattern this system implements. Required reading.                          |
| `experiment:library-extension`            | The seam catalog a new acquisition system fills, of which this system is the worked instance.   |
| `experiment:microcontroller-interface`    | The slmc + sle wrapper layer the microcontroller binding class composes.                        |
| `/mesoscope-vr-runtime`                   | Mesoscope-VR runtime behavior (state machine, training modes, CLI).                             |
| `/mesoscope-vr-session-schema`            | Field schemas of the per-session descriptors this skill delegates.                              |
| `/mesoscope-vr-module-parsing`            | Owns the forgery parser a new microcontroller module needs to reach processed output.           |
| `/mesoscope-vr-experiment-schema`         | Field schema of the experiment configuration file this skill's authoring command creates.       |
| `/mesoscope-vr-snapshots`                 | Per-session Zaber position snapshots consumed by `ZaberMotors.restore_position()`.              |
| `experiment:zaber-interface`              | Zaber motor mechanics consumed by `ZaberMotors`.                                                |
| `experiment:vr-driver-interface`          | The Unity VR task driver (`VRTaskDriver`) configured by `assets.vr_task`.                       |
| `video:camera-interface`                  | VideoSystem mechanics consumed by `VideoSystems`.                                               |
| `communication:microcontroller-interface` | MicroControllerInterface mechanics consumed by `MicroControllerInterfaces`.                     |
| `experiment:acquisition-system-setup`     | Source of camera indices, microcontroller ports, Zaber ports via hardware discovery.            |
| `experiment:google-sheets-processing`     | Reads and writes the `MesoscopeGoogleSheets` sheets (`surgery_sheet_id`, `water_log_sheet_id`). |
| `assets:working-directory`                | Required prerequisite for configuration authoring.                                              |
| `assets:task-templates`                   | Authors the task templates the experiment configuration command instantiates.                   |
| `assets:project-hierarchy`                | The on-disk hierarchy within which `MesoscopeData` resolves session paths.                      |
| `assets:session-hardware-state`           | Generic owner of the `hardware_state.yaml` snapshot whose fields gate each parser.              |
| `experiment:data-management`              | Transfer and removal workflows that consume the resolved storage destinations.                  |
| `forging:server-configuration`            | Compute-server SSH and SLURM access for remote slf batches, not storage transfer.               |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Prerequisites:
- [ ] assets:working-directory has been run on this host and the sle mcp server is connected
- [ ] describe_system_configuration_schema_tool was called and used as the source of truth for field names
- [ ] references/configuration-fields.md was consulted for field semantics

Authoring:
- [ ] Camera indices and microcontroller ports sourced from experiment:acquisition-system-setup, not guessed
- [ ] Zaber motor ports sourced from experiment:zaber-interface discovery, not guessed
- [ ] write_system_configuration_tool carried the complete record and succeeded without schema errors
- [ ] read_system_configuration_tool returned the expected configuration after the write
- [ ] validate_system_configuration_tool reported all mounts healthy
- [ ] video_tracking.conda_environment confirmed by hand, since it is the only video_tracking field outside the report
- [ ] Did not call write_server_configuration_tool, handing off to forging:server-configuration where needed

Modifying:
- [ ] Hardware-subsystem modifications followed the cross-repo workflow (slmc + sle)
- [ ] Configuration field naming follows <device>_<parameter>_<unit>
- [ ] references/configuration-fields.md updated to reflect new or changed fields
- [ ] sollertia-experiment version bumped on any dataclass schema change
- [ ] Binding class or MesoscopeDriver extended for new wrappers, cameras, motor groups, or acquisition parameters
- [ ] All affected deployments had their YAML regenerated, and this skill's inventory tables updated
- [ ] No hardware was driven by the agent, with calibration and positioning left to the experimenter
```
