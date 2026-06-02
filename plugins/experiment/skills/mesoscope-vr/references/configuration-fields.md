# Mesoscope-VR configuration fields

State snapshot of every field in `MesoscopeSystemConfiguration` and its nested calibration
dataclasses, as defined in `sollertia_experiment/mesoscope_vr/system.py`.

This file is the authoritative per-field reference for the Mesoscope-VR YAML configuration. See
[`../SKILL.md`](../SKILL.md) for the system overview, binding-class composition, and modification
workflows. Update this file whenever any field is added, removed, renamed, or has its
type/units/default changed.

---

## Top-level: MesoscopeSystemConfiguration

| Field              | Type                        | Default                      | Purpose                                                              |
|--------------------|-----------------------------|------------------------------|----------------------------------------------------------------------|
| `name`             | `str`                       | `"mesoscope"`                | Human-readable system label                                          |
| `filesystem`       | `MesoscopeFileSystem`       | `field(default_factory=...)` | Filesystem paths (see below)                                         |
| `sheets`           | `MesoscopeGoogleSheets`     | `field(default_factory=...)` | Google Sheets identifiers (see below)                                |
| `cameras`          | `MesoscopeCameras`          | `field(default_factory=...)` | Camera configuration (see below)                                     |
| `microcontrollers` | `MesoscopeMicroControllers` | `field(default_factory=...)` | Microcontroller configuration (see below)                            |
| `assets`           | `MesoscopeVRAssets`         | `field(default_factory=...)` | Zaber motor ports + nested Unity MQTT task configuration (see below) |

### Non-default behaviors

- **`__post_init__`** normalizes `microcontrollers.valve_calibration_data` from the YAML-natural
  `dict` shape to the in-memory `tuple[tuple[int | float, int | float], ...]` shape and validates
  that every entry is a 2-tuple of numeric values. Malformed entries raise `TypeError`.
- **`save(path)`** temporarily converts the valve calibration tuple back to a `dict` before YAML
  serialization, then restores the tuple form. This preserves the mapping layout in the persisted
  YAML files.

---

## MesoscopeFileSystem

Captures filesystem layout — two fields. Both default to empty paths; the user MUST set them
per-host.

| Field                 | Type              | Default                             | Purpose                                                                                                                                                                                                                                                                                                                                                                                    |
|-----------------------|-------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `mesoscope_directory` | `Path`            | `Path()`                            | Absolute path to the local-filesystem-mounted directory where mesoscope-acquired data is aggregated during acquisition by the PC that manages the mesoscope DAQ                                                                                                                                                                                                                            |
| `storage_directories` | `dict[str, Path]` | `{"NAS": Path(), "Server": Path()}` | Maps each long-term storage destination name to its local-filesystem-mounted project-root path. Seeded with the `MesoscopeStorageDestination` members `"NAS"` and `"Server"`; any number of destinations may be configured under arbitrary names. An empty path means the destination is not configured and is skipped during transfer/removal; mapping order defines pull-back preference |

The local **data root** (the directory under which projects are stored on this machine) is NOT in
this section — it is the platform-shared data root, resolved with `get_data_root()` and set with
`slsa configure data-root`.

**Mount checks:** `mesoscope_directory` and every configured `storage_directories` path are
validated by `check_system_mounts_tool` at every session start. A missing or unwritable path
aborts the runtime.

---

## MesoscopeGoogleSheets

Captures Google Sheets identifiers — two `str` fields. Both default to empty strings; the user
MUST populate them per-host.

| Field                | Type  | Default | Purpose                                                                                     |
|----------------------|-------|---------|---------------------------------------------------------------------------------------------|
| `surgery_sheet_id`   | `str` | `""`    | Identifier of the Google Sheet that stores information about surgical interventions         |
| `water_log_sheet_id` | `str` | `""`    | Identifier of the Google Sheet that stores information about water restriction and handling |

Sheet IDs are the long alphanumeric segments in Google Sheets URLs
(e.g., `1AbC...XyZ`).

---

## MesoscopeCameras

Captures per-camera configuration. The Mesoscope-VR system uses two cameras (face, body) — see
[Cameras section in SKILL.md](../SKILL.md#hardware-subsystem-cameras) for their roles.

| Field                            | Type                  | Default                       | Purpose                                                                 |
|----------------------------------|-----------------------|-------------------------------|-------------------------------------------------------------------------|
| `face_camera_index`              | `int`                 | `0`                           | Index of the face camera in the list of Harvester-managed cameras       |
| `face_camera_display_frame_rate` | `int`                 | `25`                          | Live preview FPS for the face camera (independent of save rate)         |
| `face_camera_quantization`       | `int`                 | `20`                          | H.265 quantization parameter (lower = higher quality, larger file size) |
| `face_camera_preset`             | `EncoderSpeedPresets` | `EncoderSpeedPresets.SLOWEST` | H.265 encoder speed preset (slower = better compression)                |
| `body_camera_index`              | `int`                 | `1`                           | Index of the body camera in the list of Harvester-managed cameras       |
| `body_camera_display_frame_rate` | `int`                 | `25`                          | Live preview FPS for the body camera                                    |
| `body_camera_quantization`       | `int`                 | `20`                          | H.265 quantization parameter for the body camera                        |
| `body_camera_preset`             | `EncoderSpeedPresets` | `EncoderSpeedPresets.SLOWEST` | H.265 encoder speed preset for the body camera                          |

**Source of values:**
- Camera indices come from `experiment:acquisition-system-setup` discovery
  (`ataraxis@video:camera-setup`'s `list_cameras` tool). Do NOT guess.
- Display frame rates, quantization, and presets are deployment defaults that have produced good
  results on the reference rig. Override only with measured / preferred values.

---

## MesoscopeMicroControllers

Captures port assignments + per-module calibration for the three Teensy 4.1 boards (ACTOR, SENSOR,
ENCODER). See [Microcontrollers section in SKILL.md](../SKILL.md#hardware-subsystem-microcontrollers)
for board roles.

### Port and keepalive

| Field                   | Type   | Default            | Purpose                                                                                |
|-------------------------|--------|--------------------|----------------------------------------------------------------------------------------|
| `actor_port`            | `str`  | `"/dev/ttyACM0"`   | USB port used by the ACTOR microcontroller                                             |
| `sensor_port`           | `str`  | `"/dev/ttyACM1"`   | USB port used by the SENSOR microcontroller                                            |
| `encoder_port`          | `str`  | `"/dev/ttyACM2"`   | USB port used by the ENCODER microcontroller                                           |
| `keepalive_interval_ms` | `int`  | `500`              | Interval (ms) at which controllers expect and send keepalive messages during runtime   |

Default ports use the Linux device-path form (`/dev/ttyACM*`); the value is OS-specific (e.g. `COMx` on
Windows) and is set per host from discovery.

Ports come from `experiment:acquisition-system-setup` discovery
(`ataraxis@communication:microcontroller-setup`'s `list_microcontrollers` tool). The user must
confirm which physical Teensy plays the ACTOR / SENSOR / ENCODER role and assign ports accordingly.

### Brake calibration (consumes `BrakeInterface`)

| Field                          | Type    | Default      | Purpose                                                                       |
|--------------------------------|---------|--------------|-------------------------------------------------------------------------------|
| `minimum_brake_strength_g_cm`  | `float` | `43.2047`    | Torque (gram centimeter) applied by the brake at minimum operational voltage  |
| `maximum_brake_strength_g_cm`  | `float` | `1152.1246`  | Torque (gram centimeter) applied by the brake at maximum operational voltage  |

`BrakeInterface(minimum_brake_strength, maximum_brake_strength)` converts these from g·cm to N·cm
internally.

### Wheel calibration (consumes `EncoderInterface`)

| Field                                 | Type    | Default   | Purpose                                                            |
|---------------------------------------|---------|-----------|--------------------------------------------------------------------|
| `wheel_diameter_cm`                   | `float` | `15.0333` | Diameter of the running wheel (centimeters)                        |
| `wheel_encoder_ppr`                   | `int`   | `8192`    | Pulses-Per-Revolution resolution of the wheel's quadrature encoder |
| `wheel_encoder_report_cw`             | `bool`  | `False`   | Whether to report clockwise rotation                               |
| `wheel_encoder_report_ccw`            | `bool`  | `True`    | Whether to report counter-clockwise rotation                       |
| `wheel_encoder_delta_threshold_pulse` | `int`   | `15`      | Minimum pulse-count delta for reporting a rotation event           |
| `wheel_encoder_polling_delay_us`      | `int`   | `500`     | Delay (microseconds) between consecutive encoder state readouts    |

`EncoderInterface(encoder_ppr, wheel_diameter, polling_frequency)` consumes these;
`set_parameters(report_ccw, report_cw, delta_threshold)` is sent at session start. The
centimeters-per-Unity-unit conversion is NOT a configuration field — it is read from the active
`TaskTemplate` (`vr_environment.cm_per_unity_unit`) and applied at experiment start via
`EncoderInterface.set_unity_scale()`.

### Lick sensor calibration (consumes `LickInterface`)

| Field                       | Type  | Default | Purpose                                                                              |
|-----------------------------|-------|---------|--------------------------------------------------------------------------------------|
| `lick_threshold_adc`        | `int` | `600`   | Voltage (3.3V 12-bit ADC units) interpreted as the tongue contacting the lick sensor |
| `lick_signal_threshold_adc` | `int` | `300`   | Minimum voltage (ADC units) reported to the PC; below this, signal is pulled to 0    |
| `lick_delta_threshold_adc`  | `int` | `300`   | Minimum delta between consecutive readouts to report a change                        |
| `lick_averaging_pool_size`  | `int` | `2`     | Number of readouts averaged together for the final lick sensor reading               |

`LickInterface(lick_threshold, polling_frequency)` consumes `lick_threshold_adc`;
`set_parameters(signal_threshold, delta_threshold, average_pool_size)` is sent at session start.

### Torque sensor calibration (consumes `TorqueInterface`)

| Field                         | Type    | Default    | Purpose                                                           |
|-------------------------------|---------|------------|-------------------------------------------------------------------|
| `torque_baseline_voltage_adc` | `int`   | `2048`     | ADC voltage corresponding to 0 torque (post-AD620-amplifier)      |
| `torque_maximum_voltage_adc`  | `int`   | `3443`     | ADC voltage corresponding to maximum detectable torque            |
| `torque_sensor_capacity_g_cm` | `float` | `720.0779` | Maximum torque detectable by the sensor (gram centimeter)         |
| `torque_report_cw`            | `bool`  | `True`     | Whether to report clockwise torque                                |
| `torque_report_ccw`           | `bool`  | `True`     | Whether to report counter-clockwise torque                        |
| `torque_signal_threshold_adc` | `int`   | `150`      | Minimum voltage reported to PC; below this, signal is pulled to 0 |
| `torque_delta_threshold_adc`  | `int`   | `100`      | Minimum delta between consecutive readouts to report a change     |
| `torque_averaging_pool_size`  | `int`   | `4`        | Number of readouts averaged together                              |

`TorqueInterface(baseline_voltage, maximum_voltage, sensor_capacity, polling_frequency)` consumes
the calibration fields; `set_parameters(report_ccw, report_cw, signal_threshold, delta_threshold,
averaging_pool_size)` is sent at session start.

### Screen trigger (consumes `ScreenInterface`)

| Field                              | Type   | Default | Purpose                                                                |
|------------------------------------|--------|---------|------------------------------------------------------------------------|
| `screen_trigger_pulse_duration_ms` | `int`  | `500`   | Duration (milliseconds) of the TTL pulse used to toggle screen power   |

`ScreenInterface.set_parameters(pulse_duration)` consumes this at session start (converted to
microseconds before sending).

### Mesoscope frame TTL (consumes `MesoscopeFrameTTLInterface`)

| Field                                  | Type   | Default | Purpose                                                                          |
|----------------------------------------|--------|---------|----------------------------------------------------------------------------------|
| `mesoscope_frame_averaging_pool_size`  | `int`  | `0`     | Number of digital readouts averaged when determining mesoscope frame TTL state   |

`MesoscopeFrameTTLInterface.set_parameters(averaging_pool_size)` consumes this.

### Generic sensor polling

| Field                     | Type  | Default | Purpose                                                                     |
|---------------------------|-------|---------|-----------------------------------------------------------------------------|
| `sensor_polling_delay_ms` | `int` | `1`     | Delay (milliseconds) between consecutive readouts of any non-encoder sensor |

Converted to microseconds and passed as `polling_frequency` to `MesoscopeFrameTTLInterface`,
`LickInterface`, and `TorqueInterface` constructors.

### Valve calibration (consumes `WaterValveInterface`)

| Field                    | Type                                                                                | Default                                                        | Purpose                                                                   |
|--------------------------|-------------------------------------------------------------------------------------|----------------------------------------------------------------|---------------------------------------------------------------------------|
| `valve_calibration_data` | `dict[int \| float, int \| float] \| tuple[tuple[int \| float, int \| float], ...]` | `((15000, 1.10), (30000, 3.0), (45000, 6.25), (60000, 10.90))` | Maps valve open durations (microseconds) → dispensed volume (microliters) |

`WaterValveInterface(valve_calibration_data)` consumes the tuple form; the dataclass's `__post_init__`
normalizes from `dict` on YAML load. `WaterValveInterface` fits a power-law model
(`a * pulse_duration ** b`) to this calibration data using `scipy.optimize.curve_fit`.

**Recalibration**: Use the `experiment:mesoscope-vr-runtime` skill (or `WaterValveInterface.calibrate_valve()`
directly) to gather new calibration points. Replace the entire tuple; do NOT mix old and new
measurements.

---

## MesoscopeVRAssets

Captures the Virtual Reality task assets — the three Zaber motor ports plus a nested `vr_task`
configuration. Four fields.

### Zaber motor ports

| Field           | Type  | Default          | Purpose                                                                                   |
|-----------------|-------|------------------|-------------------------------------------------------------------------------------------|
| `headbar_port`  | `str` | `"/dev/ttyUSB0"` | USB port for the HeadBar Zaber motor group (3-axis: Z, Pitch, Roll, in daisy-chain order) |
| `lickport_port` | `str` | `"/dev/ttyUSB1"` | USB port for the LickPort Zaber motor group (3-axis: Z, Y, X, in daisy-chain order)       |
| `wheel_port`    | `str` | `"/dev/ttyUSB2"` | USB port for the Wheel Zaber motor group (1-axis: X)                                      |

Default ports use the Linux device-path form (`/dev/ttyUSB*`); the value is OS-specific (e.g. `COMx` on
Windows) and is set per host from discovery.

Ports come from `experiment:zaber-interface` discovery (`get_zaber_devices_tool`). The
daisy-chain order is hardware-cabled and MUST match the order the binding class assumes.

### Unity VR task (`vr_task`)

`vr_task` is a nested `VRTaskConfiguration` (from `sollertia_experiment/vr_task/configuration.py`).
It stores only the MQTT broker discovery fields used to reach Unity.

| Field          | Type  | Default       | Purpose                                                            |
|----------------|-------|---------------|--------------------------------------------------------------------|
| `vr_task.ip`   | `str` | `"127.0.0.1"` | IP address of the MQTT broker used to reach the Unity game engine  |
| `vr_task.port` | `int` | `1883`        | Port number of the MQTT broker used to reach the Unity game engine |

The Mesoscope-VR runtime publishes VR-environment commands over MQTT. The broker is typically
co-hosted on the acquisition PC (localhost) but may be relocated to a separate machine for
multi-PC rigs. The geometric VR parameters (cue catalog, corridor geometry, cm-per-Unity-unit) are
NOT stored here — they are resolved at experiment start from the matching `TaskTemplate` YAML. See
`experiment:vr-driver-interface`.

---

## Field-naming convention recap

All configuration fields follow `<device-or-module>_<parameter>_<unit>` per
`experiment:acquisition-system-design`'s [Configuration field naming
convention](../../acquisition-system-design/SKILL.md#field-naming-convention):

| Component            | Examples                                                    |
|----------------------|-------------------------------------------------------------|
| `<device-or-module>` | `face_camera`, `lick`, `torque`, `wheel_encoder`, `headbar` |
| `<parameter>`        | `index`, `threshold`, `delta_threshold`, `ppr`, `port`      |
| `<unit>`             | `adc`, `us`, `ms`, `cm`, `g_cm`, `pulse`                    |

Adding a new field that violates this convention is a maintenance hazard — future agents auditing
configuration values will need extra context to interpret the value's units.

---

## Schema versioning rule

Any change to a field (add, remove, rename, type-change, unit-change) is a schema change and MUST
be paired with a `sollertia-experiment` version bump in `pyproject.toml`. Per
`experiment:acquisition-system-design`'s [Contract 2: Schema
versioning](../../acquisition-system-design/SKILL.md#contract-2-schema-versioning):

- **Add field**: bump minor version. Older YAML files load with the new field at default.
- **Remove field**: bump major version. Older YAML files load with the removed field silently ignored.
- **Rename field**: bump major version. Add a one-cycle deprecation migration in `__post_init__`.
- **Change type or units**: bump major version. Add validation in `__post_init__` that detects
  old shape and raises a clear migration error.
