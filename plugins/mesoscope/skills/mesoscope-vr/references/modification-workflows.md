# Mesoscope-VR modification workflows

Step-by-step workflows for changing the Mesoscope-VR hardware inventory, its calibration values, and the configuration
fields that parameterize them. See [`../SKILL.md`](../SKILL.md) for the system composition these workflows mutate and
[`configuration-fields.md`](configuration-fields.md) for the per-field reference.

---

## Configuration authoring workflow

### Step 1: Verify prerequisites

- The `sle mcp` server is connected. If not, run `experiment:experiment-mcp-environment-setup`.
- The host working directory is set. If not, run `assets:working-directory`.

### Step 2: Determine whether to create or modify

Call `read_system_configuration_tool`. It returns either `{"file_path", "data"}` or `{"error"}`, never an empty or
not-found shape. A `data` key means you are modifying an existing configuration. An `error` key means you are creating
from scratch, and is typically the `FileNotFoundError` raised when the working directory's configuration folder holds no
`*_system_configuration.yaml` file.

### Step 3: Inspect the schema

Call `describe_system_configuration_schema_tool()`. It returns
`{"schema": describe_dataclass(MesoscopeSystemConfiguration)}`, a recursive field, type, default, and required
description, and it never errors. Use that response together with [`configuration-fields.md`](configuration-fields.md)
as the source of truth. You MUST NOT guess field names.

### Step 4: Gather values from the user (creation case)

For a new host, the user-supplied values are:

- **Camera indices**: source them from `experiment:acquisition-system-setup`, which uses `video:camera-setup` for
  hardware discovery. You MUST NOT call camera discovery tools from this skill.
- **Microcontroller ports**: source them from `experiment:acquisition-system-setup`, which uses
  `communication:microcontroller-setup`. The user must confirm which physical Teensy plays the actor, sensor, or encoder
  role.
- **Zaber motor ports**: source them from `experiment:zaber-interface` discovery (`get_zaber_devices_tool`).
- **Filesystem paths**: `mesoscope_directory`, the VRPC-side mount of the ScanImagePC data share, and the
  `storage_directories` destination paths. Ask the user for absolute paths. The platform data root is set separately via
  `slsa configure data-root` (`assets:working-directory`).
- **Google Sheet IDs**: ask the user for the sheet IDs, the long alphanumeric segment in the URL. Both are optional, but
  configuring either one makes the Google service-account credentials mandatory, so confirm they are set via
  `slsa configure credentials` (`assets:working-directory`) or preprocessing aborts with `FileNotFoundError`.
- **Calibration data**: the defaults in [`configuration-fields.md`](configuration-fields.md) are reasonable starting
  points. Override them only when the user has freshly measured values.
- **Acquisition geometry**: `z_step_um`, `z_range_um`, `z_exclusion_um`, `acquisition_order`, `registration_channel`,
  `field_curvature_correction`, `frames_per_reference_plane`, and `zstack_scale_factor` describe the microscope, so
  confirm them with the user. `MesoscopeAcquisition.__post_init__` rejects a non-positive or mis-ordered value with
  `ValueError`.
- **Face-camera tracking values**: `video_tracking.conda_environment` and `video_tracking.dlc_project_path`, asked of
  the user when the host runs DeepLabCut pose inference during preprocessing. Leaving both unset keeps inference
  disabled. The remaining video-tracking fields (`shuffle`, `crop`, `batch_size`, `chunks`, `compile_model`) carry
  working defaults, so override them only for a different trained model or a different GPU.

### Step 5: Write the configuration

```text
write_system_configuration_tool(
    configuration_payload={ ... full nested dict ... },
    overwrite=True,
)
```

The tool validates the payload by round-tripping it through the dataclass, then writes the YAML in place at the target
path.

### Step 6: Verify

Call `read_system_configuration_tool` and confirm the returned configuration matches what you wrote. Then call
`validate_system_configuration_tool` to confirm filesystem mounts and hardware-port assumptions hold.

Both `validate_system_configuration_tool` and `check_system_mounts_tool` report `video_tracking.dlc_project_path` under
the `dlc_project` key. They check that the file exists and is readable, and report `{"configured": false, "ok": true}`
when the path is unset, so a wrong path fails at pre-flight rather than at the end of preprocessing. Only
`video_tracking.conda_environment` sits outside that report, so confirm the environment name by hand.

### Step 7: Hand off for server configuration

If the user is also setting up remote storage transfer, hand off to the forging plugin's `forging:server-configuration`
skill. This skill does not write the server configuration file.

---

## Camera GenICam configuration: verify, dump, restore

Declaring a camera's `*_configuration_path` records where its expected GenICam node configuration lives, so an agent
does not have to be handed the path on every operation. The path is declarative, and applying it stays a deliberate
operator step rather than something the acquisition runtime does at start-up. The workflow is:

- **Verify** the live camera against its stored config: call `verify_camera_configuration_tool` on `sle mcp`. It reads
  the configured paths, dumps each camera's live GenICam configuration, and returns a per-camera diff carrying `match`,
  `identity_match`, `camera_model`, `camera_serial_number`, `value_mismatches`, `nodes_only_in_stored`, and
  `nodes_only_in_live`. `match` is True only when the identity matches, no value mismatches exist, and no stored node is
  missing from the live camera. Extra live nodes do not break the match. A camera with no path set is reported as
  `{"configured": false}`.
- **Dump** the current configuration to the stored path (e.g. after tuning nodes): use `video:camera-setup`'s
  `dump_genicam_config_tool` (axvs MCP) with `camera_index` set to the camera's configured index and `output_file` set
  to the path declared in the system configuration.
- **Restore** a known-good configuration onto a camera: use `video:camera-setup`'s `load_genicam_config_tool` with
  `camera_index` set to the camera's configured index and `config_file` set to the declared path.

Source the path from `read_system_configuration_tool` (`cameras.<role>_camera_configuration_path`) so the dump/restore
targets the declared file. For the GenICam node mechanics themselves, hand off to `video:camera-setup`.

---

## Reindex / re-port hardware

| Change                                  | Section to mutate                                                      |
|-----------------------------------------|------------------------------------------------------------------------|
| Camera reindexed                        | `cameras.face_camera_index` / `body_camera_index`                      |
| Teensy replaced / re-flashed            | `microcontrollers.actor_port` / `sensor_port` / `encoder_port`         |
| Zaber motor group reconnected           | `assets.headbar_port` / `wheel_port` / `lickport_port`                 |
| Unity broker relocated                  | `assets.vr_task.ip` / `assets.vr_task.port`                            |
| Storage volume remounted                | `filesystem.storage_directories` / `filesystem.mesoscope_directory`    |
| Google Sheet rotated                    | `sheets.<sheet>_id`                                                    |
| DeepLabCut model or environment changed | `video_tracking.dlc_project_path` / `conda_environment`                |
| Imaging geometry or z-stack retuned     | `acquisition.*` (delivered to the ScanImagePC in each command payload) |

For each: read the configuration, mutate the relevant field, write back via `write_system_configuration_tool`. No code
changes needed.

---

## Recalibrate a module

| Change                        | Section to mutate                                                              |
|-------------------------------|--------------------------------------------------------------------------------|
| Valve recalibrated            | `microcontrollers.valve_calibration_data`                                      |
| Wheel diameter changed        | `microcontrollers.wheel_diameter_cm`                                           |
| Encoder PPR changed           | `microcontrollers.wheel_encoder_ppr`                                           |
| Lick threshold adjusted       | `microcontrollers.lick_threshold_adc` (and signal/delta if needed)             |
| Torque calibration updated    | `microcontrollers.torque_*` fields                                             |
| Brake strength bounds changed | `microcontrollers.minimum_brake_strength_g_cm` / `maximum_brake_strength_g_cm` |

For each: read, mutate, write. The binding class consumes the calibration at session start, when it instantiates the
wrappers, so a session already running keeps the values it started with.

---

## Add a new module to an existing microcontroller

This crosses repositories. Follow the "Workflow: adding a paired Module + Interface" section of
`experiment:microcontroller-interface` first to author the firmware Module and its Python wrapper, and see
`experiment:library-extension` for the full cross-repo seam map. Then in this skill:

1. **Add configuration fields** to `MesoscopeMicroControllers` for the new module's runtime parameters. Follow the
   `<device>_<parameter>_<unit>` naming convention. Update [`configuration-fields.md`](configuration-fields.md).
2. **Extend `MicroControllerInterfaces.__init__`** to instantiate the new wrapper inside the appropriate board's wrapper
   block (ACTOR, SENSOR, or ENCODER) using the new configuration fields, then add it to that board's `module_interfaces`
   tuple.
3. **Extend `MicroControllerInterfaces.start()`** when the new wrapper needs `initialize_local_assets()`, which every
   wrapper backed by a `SharedMemoryArray` needs, or a runtime parameter push through `set_parameters()`.
4. **Populate the module's hardware-state field in every session-type branch.**
   `MesoscopeVRSystem._generate_hardware_state_snapshot` (`mesoscope_vr/system_controller.py`) builds a
   `MesoscopeHardwareState` three times by hand, once per session type, in a mesoscope-experiment branch, a
   `SessionTypes.LICK_TRAINING` branch, and a `SessionTypes.RUN_TRAINING` branch, each listing its fields explicitly.
   Set the new field in every branch whose session type drives the module, and leave it at its `None` default in the
   branches that do not, because `None` is how the snapshot records a module the runtime never used. A field populated
   in only one branch stays `None` in the others, the forgery eligibility check reads that `None` as "module not
   used", and the module's data is silently absent from those sessions with no error raised. Adding the field itself
   follows the "Adding a hardware-state field for a new module" workflow of `/mesoscope-vr-session-schema`.
5. **Bump the `sollertia-experiment` version** in `pyproject.toml`.
6. **Regenerate the system configuration YAML** on every deployment so the new fields appear.
7. **Update this skill.** When the new module changes the boards' module inventory, update the table at the top of
   [Hardware subsystem: microcontrollers](../SKILL.md#hardware-subsystem-microcontrollers).
8. **Hand off to the processing side.** A module the runtime drives produces no processed output until a parser exists
   for it. Follow the "Adding a new module parser" workflow of `/mesoscope-vr-module-parsing` to write the
   `parse_<module>` entry point, register its `_ModuleSpecification` under the module's `(module_type, module_id)` key
   in `_MODULE_REGISTRY`, and register the entry point in `_MICROCONTROLLER_PARSER_REGISTRY`. The specification's
   `required_fields` and `usage_flags` name the hardware-state fields set in step 4, so the two edits must agree, or
   `check_eligibility` skips the module for every session and no output feather is ever written.

---

## Add a new module that needs a new microcontroller board

Follow the "Workflow: adding a new controller board" section of `experiment:microcontroller-interface`, in its
`references/board-allocation.md`, to add the new target macro to slmc's `main.cpp` and its PlatformIO environment to
`platformio.ini`. Then in this skill:

1. **Add a new port field** to `MesoscopeMicroControllers` (e.g., `<role>_port`).
2. **Add a new construction block** in `MicroControllerInterfaces.__init__` for the new board, using the new port, a
   `buffer_size`, the shared `keepalive_interval_ms`, and a fresh `controller_id`. The values 101, 152, and 203 are
   taken. `MicroControllerInterface` accepts a `np.uint8` id between 1 and 255, and
   `experiment:microcontroller-interface` owns the allocation convention that narrows that range.
3. **Update this skill's hardware-subsystem table** to list the new board, its controller ID, role, and modules.
4. **Bump the `sollertia-experiment` version** and regenerate YAMLs.

---

## Add a new camera

1. **Add per-camera fields** to `MesoscopeCameras` following the existing `<role>_camera_<setting>` pattern, including
   an optional `<role>_camera_configuration_path: Path = Path()` for the camera's GenICam configuration YAML. Update
   [`configuration-fields.md`](configuration-fields.md).
2. **Extend `VideoSystems`** to instantiate a new `VideoSystem` with the new camera's parameters and a fresh
   `system_id`. The values 51 and 62 are taken, and the new value must not collide with any other DataLogger source id
   in use on the rig.
3. **Add per-camera lifecycle methods** (`start_<role>_camera`, `save_<role>_camera_frames`), a `_<role>_camera_started`
   flag, and the matching `run_shutdown_step` calls in `stop()`.
4. **Bump the `sollertia-experiment` version** and regenerate YAMLs.
5. **Update this skill's hardware-subsystem table.**

---

## Add a new Zaber motor group

1. **Add a `<group>_port` field** to `MesoscopeVRAssets`.
2. **Extend `ZaberMotors`** to instantiate a new `ZaberConnection` for the new port and pull out the per-axis
   `ZaberAxis` handles in the documented daisy-chain order.
3. **Extend the ten methods that enumerate axes explicitly** to cover the new group's axes: `restore_position()`,
   `prepare_motors()`, `park_position()`, `maintenance_position()`, `mount_position()`, `unmount_position()`,
   `generate_position_snapshot()`, `unpark_motors()`, `park_motors()`, and `wait_until_idle()`. Also extend
   `disconnect()` and the `is_connected` property to cover the new `ZaberConnection`.
4. **Update the `ZaberPositions` dataclass** in `mesoscope_vr/system.py` to capture the new group's per-axis positions,
   and reconcile `/mesoscope-vr-snapshots`, which owns that record's schema.
5. **Bump the `sollertia-experiment` version** and regenerate YAMLs.
6. **Update this skill's hardware-subsystem table.**

For motor-side mechanics (checksum validation, parking, position storage), see `experiment:zaber-interface`.
