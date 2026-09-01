# Mesoscope-VR configuration fields

State snapshot of every field in `MesoscopeSystemConfiguration` and its nested calibration dataclasses, as defined in
`mesoscope_vr/system.py`.

See [`../SKILL.md`](../SKILL.md) for the system overview and binding-class composition, and
[`modification-workflows.md`](modification-workflows.md) for the modification workflows. Update this file whenever any
field is added, removed, renamed, or has its type/units/default changed.

---

## Top-level: MesoscopeSystemConfiguration

| Field              | Type                        | Default                      | Purpose                                                                       |
|--------------------|-----------------------------|------------------------------|-------------------------------------------------------------------------------|
| `name`             | `str`                       | `"mesoscope"`                | Human-readable system label                                                   |
| `filesystem`       | `MesoscopeFileSystem`       | `field(default_factory=...)` | Filesystem paths (see below)                                                  |
| `sheets`           | `MesoscopeGoogleSheets`     | `field(default_factory=...)` | Google Sheets identifiers (see below)                                         |
| `cameras`          | `MesoscopeCameras`          | `field(default_factory=...)` | Camera configuration (see below)                                              |
| `microcontrollers` | `MesoscopeMicroControllers` | `field(default_factory=...)` | Microcontroller configuration (see below)                                     |
| `acquisition`      | `MesoscopeAcquisition`      | `field(default_factory=...)` | Mesoscope motion-estimation and z-stack acquisition configuration (see below) |
| `assets`           | `MesoscopeVRAssets`         | `field(default_factory=...)` | Zaber motor ports + nested Unity MQTT task configuration (see below)          |
| `video_tracking`   | `MesoscopeVideoTracking`    | `field(default_factory=...)` | DeepLabCut face-camera eye-tracking inference configuration (see below)       |

### Non-default behaviors

- **`__post_init__`** normalizes `microcontrollers.valve_calibration_data` from the YAML-natural `dict` shape to the
  in-memory `tuple[tuple[int | float, int | float], ...]` shape and validates that every entry is a 2-tuple of numeric
  values. Malformed entries raise `TypeError`.
- **`save(path)`** temporarily converts the valve calibration tuple back to a `dict` before YAML serialization, then
  restores the tuple form. This preserves the mapping layout in the persisted YAML files.

---

## MesoscopeFileSystem

Captures the filesystem layout in two fields. Both default to empty paths, but only `mesoscope_directory` MUST be set
per host, because leaving it unset raises `ValueError` when the session filesystem layout is resolved. Individual
`storage_directories` entries are optional.

| Field                 | Type              | Default                             | Purpose                                                                                                                                                                                                                                                                                                                                                                                        |
|-----------------------|-------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `mesoscope_directory` | `Path`            | `Path()`                            | Absolute path to the local-filesystem-mounted directory where mesoscope-acquired data is aggregated during acquisition by the PC that manages the mesoscope DAQ                                                                                                                                                                                                                                |
| `storage_directories` | `dict[str, Path]` | `{"NAS": Path(), "Server": Path()}` | Maps each long-term storage destination name to its local-filesystem-mounted project-root path. Seeded with the `MesoscopeStorageDestination` members `"NAS"` and `"Server"`, and any number of destinations may be configured under arbitrary names. An empty path means the destination is not configured and is skipped during transfer/removal. Mapping order defines pull-back preference |

The local **data root**, the directory under which projects are stored on this machine, is platform-shared rather than a
field of this section. Resolve it with `get_data_root()` and set it with `slsa configure data-root`.

**Mount checks:** `check_system_mounts_tool` is an agent-invoked MCP tool that returns a diagnostic report covering
every path the configuration declares, keyed as `data_root`, `mesoscope_directory`, one `storage_directory:<name>` entry
per declared destination, `face_camera_configuration`, `body_camera_configuration`, and `dlc_project`. The directories
are reported `ok` when they exist and are writable, and the three optional input files when they exist and are readable.
An unset storage root or input file is reported as `{"configured": False, "ok": True}`, because configuring those is
optional, while an unset `mesoscope_directory` is reported as `{"configured": False, "ok": False}`.
`validate_system_configuration_tool` builds its own `paths` report from the same source. Call one of them yourself as a
pre-flight check. The acquisition runtime performs no on-disk existence check of its own. `MesoscopeData.__init__`
rejects an unset `mesoscope_directory` with `ValueError` before any path is resolved, while unset storage roots are
recorded under `unconfigured_destinations` and only produce a preprocessing warning about the skipped backup.

See [MesoscopeData path resolution](#mesoscopedata-path-resolution) below for the class that turns this section into
resolved per-session paths.

---

## MesoscopeData path resolution

`MesoscopeData(system_configuration, session_data)` (`mesoscope_vr/system.py`) turns the `filesystem` section into the
resolved paths a session reads and writes. It raises `ValueError` when `mesoscope_directory` is left unset, before any
path is resolved, because every Mesoscope-VR session type resolves part of its layout under the Mesoscope acquisition
mount. It then anchors an `AnimalData` on the platform data root and rebinds that anchor through `.for_root()` onto the
Mesoscope mount and onto each configured storage root.

| Attribute                   | Type                  | Content                                                                      |
|-----------------------------|-----------------------|------------------------------------------------------------------------------|
| `vrpc_data`                 | `_VRPCPersistentData` | The per-animal VRPC `persistent_data` layout for the session's type          |
| `scanimagepc_data`          | `_ScanImagePCData`    | The ScanImagePC layout under the Mesoscope acquisition mount                 |
| `destinations`              | `StorageDestinations` | One `StorageDestination` per configured storage root, in configuration order |
| `unconfigured_destinations` | `tuple[str, ...]`     | Names of the storage roots left unset, which preprocessing warns about       |

`_VRPCPersistentData(session_type, persistent_data_path)` (`mesoscope_vr/system.py`) derives `zaber_positions.yaml`,
`mesoscope_positions.yaml`, `window_screenshot.png`, and a per-session-type `*_descriptor.yaml` under the animal's VRPC
persistent directory, creating the directory when it is absent. An unrecognized session type raises `ValueError`.

`_ScanImagePCData(session, mesoscope_root_path, persistent_data_path)` (`mesoscope_vr/system.py`) derives the animal's
persistent `MotionEstimator.me` and `fov.roi`, the session-specific directory `<root>/<session>`, and the shared
acquisition directory `<root>/mesoscope_data` that every session writes into during its runtime.

For the on-disk hierarchy these resolved paths address, see `assets:project-hierarchy`. For the transfer, verification,
and removal workflows that consume `destinations`, see `experiment:data-management`. For the per-session snapshot files
under `vrpc_data`, see `/mesoscope-vr-snapshots`.

---

## MesoscopeGoogleSheets

Captures Google Sheets identifiers in two `str` fields. Both are optional and default to empty strings. An unset
identifier skips that exchange with a warning, and leaving both unset disables the Google Sheets integration entirely.
Configuring **either** identifier makes the Google service-account credentials mandatory: preprocessing resolves them
before either exchange and aborts with `FileNotFoundError` when they are missing or unconfigured. Set them with
`slsa configure credentials` (`assets:working-directory`).

| Field                | Type  | Default | Purpose                                                                                     |
|----------------------|-------|---------|---------------------------------------------------------------------------------------------|
| `surgery_sheet_id`   | `str` | `""`    | Identifier of the Google Sheet that stores information about surgical interventions         |
| `water_log_sheet_id` | `str` | `""`    | Identifier of the Google Sheet that stores information about water restriction and handling |

Sheet IDs are the long alphanumeric segments in Google Sheets URLs (e.g., `1AbC...XyZ`).

---

## MesoscopeCameras

Captures per-camera configuration. The Mesoscope-VR system uses two cameras, face and body. See the
[Cameras section in SKILL.md](../SKILL.md#hardware-subsystem-cameras) for their roles.

| Field                            | Type                  | Default                       | Purpose                                                                 |
|----------------------------------|-----------------------|-------------------------------|-------------------------------------------------------------------------|
| `face_camera_index`              | `int`                 | `0`                           | Index of the face camera in the list of Harvester-managed cameras       |
| `face_camera_display_frame_rate` | `int`                 | `25`                          | Live preview FPS for the face camera (independent of save rate)         |
| `face_camera_quantization`       | `int`                 | `20`                          | H.265 quantization parameter (lower = higher quality, larger file size) |
| `face_camera_preset`             | `EncoderSpeedPresets` | `EncoderSpeedPresets.SLOWEST` | H.265 encoder speed preset (slower = better compression)                |
| `face_camera_configuration_path` | `Path`                | `Path()` (unset)              | Optional GenICam config .yaml: face camera's expected node config       |
| `body_camera_index`              | `int`                 | `1`                           | Index of the body camera in the list of Harvester-managed cameras       |
| `body_camera_display_frame_rate` | `int`                 | `25`                          | Live preview FPS for the body camera                                    |
| `body_camera_quantization`       | `int`                 | `20`                          | H.265 quantization parameter for the body camera                        |
| `body_camera_preset`             | `EncoderSpeedPresets` | `EncoderSpeedPresets.SLOWEST` | H.265 encoder speed preset for the body camera                          |
| `body_camera_configuration_path` | `Path`                | `Path()` (unset)              | Optional GenICam config .yaml: body camera's expected node config       |

**Source of values:**
- Camera indices come from `experiment:acquisition-system-setup` discovery (`video:camera-setup`'s `list_cameras_tool`,
  ataraxis marketplace). Do NOT guess.
- Display frame rates, quantization, and presets are deployment defaults that have produced good results on the
  reference rig. Override only with measured / preferred values.
- Configuration paths are **optional**. Set them only for cameras whose GenICam node configuration is captured to a YAML
  (the standard practice for GenTL/GenICam cameras). By convention these files live in the working-directory
  `configuration/` folder, next to `*_system_configuration.yaml` (e.g. `face_camera_configuration.yaml`). See
  [`modification-workflows.md`](modification-workflows.md) for the verify / dump / restore workflow. The file is an
  `ataraxis-video-system` `GenicamConfiguration` YAML.

---

## MesoscopeMicroControllers

Captures port assignments + per-module calibration for the three Teensy 4.1 boards (ACTOR, SENSOR, ENCODER). See
[Microcontrollers section in SKILL.md](../SKILL.md#hardware-subsystem-microcontrollers) for board roles.

The `MesoscopeMicroControllers` dataclass in `mesoscope_vr/system.py` holds 28 fields. Three name the board ports and
one sets the keepalive interval. The remaining 24 parameterize seven of the eight module wrappers running on the three
boards: brake strength, wheel geometry, lick thresholds, torque calibration, encoder reporting, screen pulse duration,
sensor polling delay, and the valve calibration table. `GasPuffValveInterface` takes no configuration.

### Port and keepalive

| Field                   | Type  | Default          | Purpose                                                                              |
|-------------------------|-------|------------------|--------------------------------------------------------------------------------------|
| `actor_port`            | `str` | `"/dev/ttyACM0"` | USB port used by the ACTOR microcontroller                                           |
| `sensor_port`           | `str` | `"/dev/ttyACM1"` | USB port used by the SENSOR microcontroller                                          |
| `encoder_port`          | `str` | `"/dev/ttyACM2"` | USB port used by the ENCODER microcontroller                                         |
| `keepalive_interval_ms` | `int` | `500`            | Interval (ms) at which controllers expect and send keepalive messages during runtime |

Default ports use the Linux device-path form (`/dev/ttyACM*`). The value is OS-specific, taking the `COMx` form on
Windows, and is set per host from discovery.

Ports come from `experiment:acquisition-system-setup` discovery (`communication:microcontroller-setup`'s
`list_microcontrollers_tool`). The user must confirm which physical Teensy plays the ACTOR / SENSOR / ENCODER role and
assign ports accordingly.

### Brake calibration (consumes `BrakeInterface`)

| Field                         | Type    | Default     | Purpose                                                                      |
|-------------------------------|---------|-------------|------------------------------------------------------------------------------|
| `minimum_brake_strength_g_cm` | `float` | `43.2047`   | Torque (gram centimeter) applied by the brake at minimum operational voltage |
| `maximum_brake_strength_g_cm` | `float` | `1152.1246` | Torque (gram centimeter) applied by the brake at maximum operational voltage |

`BrakeInterface(minimum_brake_strength, maximum_brake_strength)` converts these from g·cm to N·cm internally.

### Wheel calibration (consumes `EncoderInterface`)

| Field                                 | Type    | Default   | Purpose                                                            |
|---------------------------------------|---------|-----------|--------------------------------------------------------------------|
| `wheel_diameter_cm`                   | `float` | `15.0333` | Diameter of the running wheel (centimeters)                        |
| `wheel_encoder_ppr`                   | `int`   | `8192`    | Pulses-Per-Revolution resolution of the wheel's quadrature encoder |
| `wheel_encoder_report_cw`             | `bool`  | `False`   | Whether to report clockwise rotation                               |
| `wheel_encoder_report_ccw`            | `bool`  | `True`    | Whether to report counter-clockwise rotation                       |
| `wheel_encoder_delta_threshold_pulse` | `int`   | `15`      | Minimum pulse-count delta for reporting a rotation event           |
| `wheel_encoder_polling_delay_us`      | `int`   | `500`     | Delay (microseconds) between consecutive encoder state readouts    |

`EncoderInterface(encoder_ppr, wheel_diameter, polling_frequency)` consumes these, and
`set_parameters(report_ccw, report_cw, delta_threshold)` is sent at session start. The centimeters-per-Unity-unit
conversion lives in the active `TaskTemplate` (`vr_environment.cm_per_unity_unit`) rather than in this section, and the
runtime applies it at experiment start through `EncoderInterface.set_unity_scale()`.

### Lick sensor calibration (consumes `LickInterface`)

| Field                       | Type  | Default | Purpose                                                                              |
|-----------------------------|-------|---------|--------------------------------------------------------------------------------------|
| `lick_threshold_adc`        | `int` | `600`   | Voltage (3.3V 12-bit ADC units) interpreted as the tongue contacting the lick sensor |
| `lick_signal_threshold_adc` | `int` | `300`   | Minimum voltage (ADC units) reported to the PC. Below this, signal is pulled to 0    |
| `lick_delta_threshold_adc`  | `int` | `300`   | Minimum delta between consecutive readouts to report a change                        |
| `lick_averaging_pool_size`  | `int` | `2`     | Number of readouts averaged together for the final lick sensor reading               |

`LickInterface(lick_threshold, polling_frequency)` consumes `lick_threshold_adc`, and
`set_parameters(signal_threshold, delta_threshold, average_pool_size)` is sent at session start.

### Torque sensor calibration (consumes `TorqueInterface`)

| Field                         | Type    | Default    | Purpose                                                           |
|-------------------------------|---------|------------|-------------------------------------------------------------------|
| `torque_baseline_voltage_adc` | `int`   | `2048`     | ADC voltage corresponding to 0 torque (post-AD620-amplifier)      |
| `torque_maximum_voltage_adc`  | `int`   | `3443`     | ADC voltage corresponding to maximum detectable torque            |
| `torque_sensor_capacity_g_cm` | `float` | `720.0779` | Maximum torque detectable by the sensor (gram centimeter)         |
| `torque_report_cw`            | `bool`  | `True`     | Whether to report clockwise torque                                |
| `torque_report_ccw`           | `bool`  | `True`     | Whether to report counter-clockwise torque                        |
| `torque_signal_threshold_adc` | `int`   | `150`      | Minimum voltage reported to PC. Below this, signal is pulled to 0 |
| `torque_delta_threshold_adc`  | `int`   | `100`      | Minimum delta between consecutive readouts to report a change     |
| `torque_averaging_pool_size`  | `int`   | `4`        | Number of readouts averaged together                              |

`TorqueInterface(baseline_voltage, maximum_voltage, sensor_capacity, polling_frequency)` consumes the calibration
fields, and `set_parameters(report_ccw, report_cw, signal_threshold, delta_threshold, averaging_pool_size)` is sent at
session start.

### Screen trigger (consumes `ScreenInterface`)

| Field                              | Type  | Default | Purpose                                                              |
|------------------------------------|-------|---------|----------------------------------------------------------------------|
| `screen_trigger_pulse_duration_ms` | `int` | `500`   | Duration (milliseconds) of the TTL pulse used to toggle screen power |

`ScreenInterface.set_parameters(pulse_duration)` consumes this at session start (converted to microseconds before
sending).

### Mesoscope frame TTL (consumes `MesoscopeFrameTTLInterface`)

| Field                                 | Type  | Default | Purpose                                                                        |
|---------------------------------------|-------|---------|--------------------------------------------------------------------------------|
| `mesoscope_frame_averaging_pool_size` | `int` | `0`     | Number of digital readouts averaged when determining mesoscope frame TTL state |

`MesoscopeFrameTTLInterface.set_parameters(averaging_pool_size)` consumes this at session start.

### Generic sensor polling

| Field                     | Type  | Default | Purpose                                                                     |
|---------------------------|-------|---------|-----------------------------------------------------------------------------|
| `sensor_polling_delay_ms` | `int` | `1`     | Delay (milliseconds) between consecutive readouts of any non-encoder sensor |

Converted to microseconds and passed as `polling_frequency` to `MesoscopeFrameTTLInterface`, `LickInterface`, and
`TorqueInterface` constructors.

### Valve calibration (consumes `WaterValveInterface`)

| Field                    | Type                                                                                | Default                                                        | Purpose                                                                   |
|--------------------------|-------------------------------------------------------------------------------------|----------------------------------------------------------------|---------------------------------------------------------------------------|
| `valve_calibration_data` | `dict[int \| float, int \| float] \| tuple[tuple[int \| float, int \| float], ...]` | `((15000, 1.10), (30000, 3.0), (45000, 6.25), (60000, 10.90))` | Maps valve open durations (microseconds) → dispensed volume (microliters) |

`WaterValveInterface(valve_calibration_data)` consumes the tuple form, which the dataclass's `__post_init__` normalizes
from `dict` on YAML load. `WaterValveInterface` fits a power-law model (`a * pulse_duration ** b`) to this calibration
data using `scipy.optimize.curve_fit`.

**Recalibration** drives the valve hardware, repeatedly opening the valve to measure the dispensed volume, so the
experimenter performs it on the rig. Direct the user to the maintenance runtime (`sle mesoscope maintain`, the
`maintenance_logic` hardware-maintenance GUI documented in `/mesoscope-vr-runtime`) to gather new calibration points,
then update `valve_calibration_data` with the resulting measurements. Replace the entire tuple. You MUST NOT mix old and
new measurements, and you MUST NOT invoke `WaterValveInterface.calibrate_valve()` or drive the valve yourself.

---

## MesoscopeAcquisition

Captures the online motion-estimation and z-stack acquisition configuration delivered to the ScanImagePC. See
[`mesoscope-driver.md`](mesoscope-driver.md) for the `MesoscopeDriver` MQTT contract that carries these parameters to
the `runAcquisition` MATLAB function. Eight fields.

| Field                        | Type                        | Default        | Purpose                                                                                                                                                                                                        |
|------------------------------|-----------------------------|----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `z_step_um`                  | `int`                       | `20`           | Spacing, in micrometers, between consecutive target imaging planes in the acquired z-stack                                                                                                                     |
| `z_range_um`                 | `tuple[int, int]`           | `(1050, 1050)` | The `[minimum, maximum]` z-plane range to image, in micrometers. Equal boundaries image a single plane at that depth, and distinct boundaries image the inclusive slice between them                           |
| `z_exclusion_um`             | `tuple[int, int]`           | `(0, 0)`       | The `[minimum, maximum]` boundaries, in micrometers, of the non-imaged exclusion zone for two-plane imaging. Equal boundaries disable two-plane imaging, and distinct boundaries must fall within `z_range_um` |
| `acquisition_order`          | `MesoscopeAcquisitionOrder` | `INTERLEAVED`  | Order in which the target planes are acquired when building the reference and high-definition z-stacks                                                                                                         |
| `registration_channel`       | `int`                       | `1`            | Acquisition channel used for online motion registration and the high-definition reference z-stack                                                                                                              |
| `field_curvature_correction` | `bool`                      | `False`        | Whether ScanImage field curvature correction is enabled during acquisition (microscope-dependent)                                                                                                              |
| `frames_per_reference_plane` | `int`                       | `20`           | Number of frames acquired and averaged at each reference plane. Larger values improve motion characterization at the cost of longer processing and higher acquisition-machine load                             |
| `zstack_scale_factor`        | `float`                     | `2.0`          | Factor by which each ROI's X and Y resolution is scaled when acquiring the high-definition reference z-stack. The scaling preserves the original ROI aspect ratios                                             |

### MesoscopeAcquisitionOrder enum

`acquisition_order` is a `MesoscopeAcquisitionOrder` (`StrEnum`) with two members:

| Member        | Value           | Meaning                                                                                           |
|---------------|-----------------|---------------------------------------------------------------------------------------------------|
| `INTERLEAVED` | `"interleaved"` | Iterate over the target planes once per acquired volume, one frame at each plane (Z1, Z2, Z1, Z2) |
| `SMOOTH`      | `"smooth"`      | Acquire all averaged frames at one target plane before advancing to the next (Z1, Z1, Z2, Z2)     |

### `__post_init__` validation

`MesoscopeAcquisition.__post_init__` raises `ValueError` (via `console.error`) when:

- `z_step_um`, `registration_channel`, `frames_per_reference_plane`, or `zstack_scale_factor` is not positive (`<= 0`).
- `z_range_um[0]` is not positive, or the boundaries are not ordered as `(minimum, maximum)`
  (`z_range_um[0] > z_range_um[1]`).
- `z_exclusion_um` boundaries are not ordered as `(minimum, maximum)`.
- a configured (unequal) `z_exclusion_um` zone does not fall within the `z_range_um` boundaries.

**Source of values:** These are deployment defaults tuned for the reference rig and microscope. Override them with the
imaging geometry and estimator settings appropriate for the specific Mesoscope. This section is the single source of
truth for the acquisition geometry, because the parameters travel to the ScanImagePC inside each command payload that
consumes them.

---

## MesoscopeVRAssets

Captures the Virtual Reality task assets in four fields: the three Zaber motor ports plus a nested `vr_task`
configuration.

### Zaber motor ports

| Field           | Type  | Default          | Purpose                                                                                   |
|-----------------|-------|------------------|-------------------------------------------------------------------------------------------|
| `headbar_port`  | `str` | `"/dev/ttyUSB0"` | USB port for the HeadBar Zaber motor group (3-axis: Z, Pitch, Roll, in daisy-chain order) |
| `lickport_port` | `str` | `"/dev/ttyUSB1"` | USB port for the LickPort Zaber motor group (3-axis: Z, Y, X, in daisy-chain order)       |
| `wheel_port`    | `str` | `"/dev/ttyUSB2"` | USB port for the Wheel Zaber motor group (1-axis: X)                                      |

Default ports use the Linux device-path form (`/dev/ttyUSB*`). The value is OS-specific, taking the `COMx` form on
Windows, and is set per host from discovery.

Ports come from `experiment:zaber-interface` discovery (`get_zaber_devices_tool`). The daisy-chain order is
hardware-cabled and MUST match the order the binding class assumes.

### Unity VR task (`vr_task`)

`vr_task` is a nested `VRTaskConfiguration` (from `vr_task/configuration.py`). It stores only the MQTT broker discovery
fields. Mesoscope-VR nests it under `MesoscopeSystemConfiguration.assets.vr_task` (the `vr_task` field of
`MesoscopeVRAssets` in `mesoscope_vr/system.py`), which is the platform-general VR task seam that
`experiment:vr-driver-interface` documents.

| Field          | Type  | Default       | Purpose                                                            |
|----------------|-------|---------------|--------------------------------------------------------------------|
| `vr_task.ip`   | `str` | `"127.0.0.1"` | IP address of the MQTT broker used to reach the Unity game engine  |
| `vr_task.port` | `int` | `1883`        | Port number of the MQTT broker used to reach the Unity game engine |

The Mesoscope-VR runtime publishes VR-environment commands over MQTT. The broker is typically co-hosted on the
acquisition PC (localhost) but may be relocated to a separate machine for multi-PC rigs. The geometric VR parameters
(cue catalog, corridor geometry, cm-per-Unity-unit) live in the matching `TaskTemplate` YAML, which the runtime resolves
at experiment start. See `experiment:vr-driver-interface`.

`MesoscopeDriver` reuses these same two fields as its own broker discovery (the `MesoscopeDriver` construction in
`MesoscopeVRSystem.__init__` of `mesoscope_vr/system_controller.py`, and `MesoscopeDriver.__init__` in
`mesoscope_vr/mesoscope_driver.py`), so the Unity task and the ScanImagePC share one broker. See
[`mesoscope-driver.md`](mesoscope-driver.md) for the topic namespace that keeps the two surfaces apart.

Scene activation and Play Mode are driven over the editor MCP Bridge on the fixed loopback endpoint `127.0.0.1:8090`,
which `unity:unity-mcp-environment-setup` owns, so this section carries no bridge field.

---

## MesoscopeVideoTracking

Captures the DeepLabCut pose-inference configuration that analyzes the face-camera video during experiment-session
preprocessing. Seven fields, plus the `conda run` subprocess boundary, the placement inside the preprocessing pipeline,
and the transfer-abort failure mode.

| Field               | Type   | Default  | Purpose                                                                                                                                                                      |
|---------------------|--------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `conda_environment` | `str`  | `""`     | Name of the conda environment that provides the `slvt` command and its DeepLabCut installation. An empty string disables face-camera inference                               |
| `dlc_project_path`  | `Path` | `Path()` | Absolute path to the DeepLabCut project's `config.yaml` whose trained model analyzes the face-camera video. An empty path disables face-camera inference                     |
| `shuffle`           | `int`  | `1`      | Shuffle index of the trained DeepLabCut model to run                                                                                                                         |
| `crop`              | `str`  | `""`     | The `x1,x2,y1,y2` pixel rectangle to analyze instead of the full frame, matching the region the model was trained on. An empty string analyzes the project's configured crop |
| `batch_size`        | `int`  | `32`     | Number of frames the pose model processes per forward pass, sized for the acquisition rig's GPU                                                                              |
| `chunks`            | `int`  | `1`      | Number of contiguous frame-range pieces the face-camera video is split into for concurrent analysis. A value of one analyzes the video as a single unbroken frame range      |
| `compile_model`     | `bool` | `True`   | Whether the pose model is compiled with `torch.compile`. Enabled by default because the rig's GPU amortizes the one-time warm-up cost over the long face-camera video        |

**Opt-in gate:** face-camera inference runs only when the host configures both `conda_environment` and
`dlc_project_path`. An empty string or an unset path disables it, matching the empty-value idiom the other sections use.
Inference is also skipped, with a warning, when the expected face-camera video is absent.

**Source of values:** `conda_environment` and `dlc_project_path` name the host's DeepLabCut install and trained project,
so ask the user for both. `shuffle`, `crop`, `batch_size`, `chunks`, and `compile_model` are deployment defaults tuned
for the reference rig's GPU. Override them for a different trained model or a different GPU.

**Pre-flight coverage:** `dlc_project_path` is covered by the mount checks the
[MesoscopeFileSystem](#mesoscopefilesystem) section describes. The `conda_environment` name sits outside that report, so
confirm it separately.

### Face-tracking subprocess

`_launch_face_tracking` (in `mesoscope_vr/data_preprocessing.py`) applies that gate and launches the subprocess.

The pose model runs in a separate process. `sollertia-video-tracking` (slvt) pins `deeplabcut[gui]==3.0.1`, which
constrains it to Python 3.12 and the numpy 1.x series, while sollertia-experiment runs Python 3.14 and numpy 2.5.2. The
acquisition process therefore reaches slvt across a `conda run` boundary:

```text
conda run -n <conda_environment> slvt infer --config-path <dlc_project_path>
  --videos <session>/raw_data/camera_data/<session>_face_camera.mp4
  --shuffle <shuffle> --device cuda --gpus 0 --batch-size <batch_size> --chunks <chunks>
  --compile-model on|off --no-progress [--crop <crop>]
```

slvt ships no MCP server and no plugin, so the `slvt` CLI is its only agent-facing surface and this binding is
documented on the `sollertia-experiment` side.

Preprocessing launches inference asynchronously right after `rename_session_videos`, and only for
`SessionTypes.MESOSCOPE_EXPERIMENT` sessions, so it overlaps the CPU-bound and disk-bound stages on the rig's
otherwise-idle GPU. `_join_face_tracking` waits for it immediately before `push_session_data`. A successful run writes
the DeepLabCut `.h5` file and its companion pickles beside the face-camera video in `raw_data/camera_data/`, which is
the `slvt infer` default when `--output` is omitted. They are therefore covered by the raw-data checksum and shipped to
long-term storage as raw data, where the forging plugin's video pipeline consumes them.

A non-zero exit status or zero written `.h5` prediction files raises `RuntimeError`, aborts the transfer to long-term
storage, and retains the local session copy for a manual retry. The transient log lives at
`<tmp>/slvt_infer_<session_name>.log`, is removed on success, and is retained on failure, with its last 2000 characters
echoed into the error.

---

## Field-naming convention

Field naming follows the "Field naming convention" rule of `experiment:acquisition-system-design`, in its
`references/layer-patterns.md`. The `<unit>` suffixes in use across the Mesoscope-VR schema are `adc`, `us`, `ms`, `um`,
`cm`, `g_cm`, and `pulse`.

---

## Schema versioning

Schema changes follow the "Contract 2: Schema versioning" rule of `experiment:acquisition-system-design`, in its
`references/layer-patterns.md`. The package whose version a Mesoscope-VR schema change MUST bump is
`sollertia-experiment`.
