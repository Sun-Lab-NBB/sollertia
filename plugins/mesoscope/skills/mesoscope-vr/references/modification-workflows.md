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
from scratch, and is typically the `FileNotFoundError` raised when the working directory's configuration folder holds
no `*_system_configuration.yaml` file.

### Step 3: Inspect the schema

Call `describe_system_configuration_schema_tool()`. The schema response gives canonical field
names, types, defaults, and units. Use this **and**
[`configuration-fields.md`](configuration-fields.md) as the source of truth —
do not guess field names.

### Step 4: Gather values from the user (creation case)

For a new host, the user-supplied values are:

- **Camera indices** — must come from `experiment:acquisition-system-setup`, which uses
  `video:camera-setup` for hardware discovery. Do NOT call camera discovery tools from this skill.
- **Microcontroller ports** — must come from `experiment:acquisition-system-setup`, which uses
  `communication:microcontroller-setup`. The user must confirm which physical Teensy plays
  the actor / sensor / encoder role.
- **Zaber motor ports** — must come from `experiment:zaber-interface` discovery
  (`get_zaber_devices_tool`).
- **Filesystem paths** — `mesoscope_directory` and the `storage_directories` destination paths.
  Ask the user for absolute paths. The platform data root is set separately via
  `slsa configure data-root` (`assets:working-directory`).
- **Google Sheet IDs** — ask the user for the sheet IDs (the long alphanumeric segment in the URL). Both are
  optional, but configuring either one makes the Google service-account credentials mandatory, so confirm they are set
  via `slsa configure credentials` (`assets:working-directory`) or preprocessing aborts with `FileNotFoundError`.
- **Calibration data** — defaults in
  [`configuration-fields.md`](configuration-fields.md) are reasonable
  starting points. Only override if the user has freshly measured values.
- **Face-camera tracking values**: `video_tracking.conda_environment` and
  `video_tracking.dlc_project_path`, asked of the user when the host runs DeepLabCut eye-tracking
  inference during preprocessing. Leaving both unset keeps inference disabled. The remaining
  video-tracking fields (`shuffle`, `crop`, `batch_size`, `chunks`, `compile_model`) carry working
  defaults, so override them only for a different trained model or a different GPU.

### Step 5: Write the configuration

```text
write_system_configuration_tool(
    configuration_payload={ ... full nested dict ... },
    overwrite=True,
)
```

The tool validates the payload by round-tripping it through the dataclass, then writes the YAML in
place at the target path.

### Step 6: Verify

Call `read_system_configuration_tool` and confirm the returned configuration matches what you
wrote. Then call `validate_system_configuration_tool` to confirm filesystem mounts and
hardware-port assumptions hold.

Both `validate_system_configuration_tool` and `check_system_mounts_tool` report
`video_tracking.dlc_project_path` under the `dlc_project` key, checking that the file exists and is readable and
reporting `{"configured": false, "ok": true}` when the path is unset, so a wrong path fails at pre-flight rather than
at the end of preprocessing. Only `video_tracking.conda_environment` sits outside that report, so confirm the
environment name by hand.

### Step 7: Hand off for server configuration

If the user is also setting up remote storage transfer, hand off to the forging plugin's
`forging:server-configuration` skill. This skill does not write the server configuration file.

---

## Camera GenICam configuration: verify, dump, restore

Declaring a camera's `*_configuration_path` records *where* its expected GenICam node configuration
lives, so agents do not have to be handed the path on every operation. The path is declarative only
— the acquisition runtime does not auto-apply it. The workflow is:

- **Verify** the live camera against its stored config: call `verify_camera_configuration_tool` (this
  skill's MCP server). It reads the configured paths, dumps each camera's live GenICam configuration,
  and returns a per-camera diff (`match`, identity match, `value_mismatches`, nodes present in only
  one side). Cameras with no path set are reported as `{"configured": false}`.
- **Dump** the current configuration to the stored path (e.g. after tuning nodes): use `video:camera-setup`'s
  `dump_genicam_config_tool` (axvs MCP) with `camera_index` set to the camera's configured index and `output_file` set
  to the path declared in the system configuration.
- **Restore** a known-good configuration onto a camera: use `video:camera-setup`'s `load_genicam_config_tool` with
  `camera_index` set to the camera's configured index and `config_file` set to the declared path.

Source the path from `read_system_configuration_tool` (`cameras.<role>_camera_configuration_path`)
so the dump/restore targets the declared file. For the GenICam node mechanics themselves, hand off
to `video:camera-setup`.

---

## Reindex / re-port hardware

| Change                                  | Section to mutate                                                   |
|-----------------------------------------|---------------------------------------------------------------------|
| Camera reindexed                        | `cameras.face_camera_index` / `body_camera_index`                   |
| Teensy replaced / re-flashed            | `microcontrollers.actor_port` / `sensor_port` / `encoder_port`      |
| Zaber motor group reconnected           | `assets.headbar_port` / `wheel_port` / `lickport_port`              |
| Unity broker relocated                  | `assets.vr_task.ip` / `assets.vr_task.port`                         |
| Storage volume remounted                | `filesystem.storage_directories` / `filesystem.mesoscope_directory` |
| Google Sheet rotated                    | `sheets.<sheet>_id`                                                 |
| DeepLabCut model or environment changed | `video_tracking.dlc_project_path` / `conda_environment`             |

For each: read the configuration, mutate the relevant field, write back via
`write_system_configuration_tool`. No code changes needed.

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

For each: read, mutate, write. The calibration is consumed at session start when the binding class
instantiates the wrappers; existing sessions are unaffected.

---

## Add a new module to an existing microcontroller

This crosses repositories. Follow the workflow in `experiment:microcontroller-interface`'s
"Adding a paired Module + Interface" section first to author the firmware Module + Python wrapper.
Then in this skill:

1. **Add configuration fields** to `MesoscopeMicroControllers` for the new module's runtime
   parameters. Follow the `<device>_<parameter>_<unit>` naming convention. Update
   [`configuration-fields.md`](configuration-fields.md).
2. **Extend `MicroControllerInterfaces`** to instantiate the new wrapper inside the appropriate
   board's wrapper block (ACTOR / SENSOR / ENCODER) using the new configuration fields. Add it to
   the board's `module_interfaces` tuple.
3. **Extend `MicroControllerInterfaces.start()`** if the new wrapper needs `initialize_local_assets()`
   (i.e., uses `SharedMemoryArray`) or per-module runtime parameter pushes (i.e., has a
   `set_parameters()` method).
4. **Bump the `sollertia-experiment` version** in `pyproject.toml`.
5. **Regenerate the system configuration YAML** on every deployment so the new fields appear.
6. **Update this skill** — if the new module changes the boards' module inventory in the table at
   the top of [Hardware subsystem: Microcontrollers](../SKILL.md#hardware-subsystem-microcontrollers),
   update that table.

---

## Add a new module that needs a new microcontroller board

Follow the workflow in `experiment:microcontroller-interface`'s "Adding a new controller board"
section to add the new target macro to slmc's `main.cpp` and its PlatformIO environment to
`platformio.ini`. Then in this skill:

1. **Add a new port field** to `MesoscopeMicroControllers` (e.g., `<role>_port`).
2. **Add a new construction block** in `MicroControllerInterfaces.__init__` for the new board,
   using the new port and a fresh `controller_id` not currently in use (101, 152, 203 are taken).
3. **Update this skill's hardware-subsystem table** to list the new board, its controller ID, role, and
   modules.
4. **Bump the `sollertia-experiment` version** and regenerate YAMLs.

---

## Add a new camera

1. **Add per-camera fields** to `MesoscopeCameras` following the existing `<role>_camera_<setting>`
   pattern, including an optional `<role>_camera_configuration_path: Path = Path()` for the camera's
   GenICam configuration YAML. Update [`configuration-fields.md`](configuration-fields.md).
2. **Extend `VideoSystems`** to instantiate a new `VideoSystem` with the new camera's parameters
   and a fresh `system_id` (51 and 62 are taken; pick a value that doesn't collide with the
   DataLogger source range used by other subsystems).
3. **Add per-camera lifecycle methods** (`start_<role>_camera`, `save_<role>_camera_frames`) and
   update `stop()` to include the new camera.
4. **Bump the `sollertia-experiment` version** and regenerate YAMLs.
5. **Update this skill's hardware-subsystem table.**

---

## Add a new Zaber motor group

1. **Add a `<group>_port` field** to `MesoscopeVRAssets`.
2. **Extend `ZaberMotors`** to instantiate a new `ZaberConnection` for the new port and pull out
   the per-axis `ZaberAxis` handles in the documented daisy-chain order.
3. **Extend all nine methods that enumerate axes explicitly** to cover the new group's axes: `restore_position()`,
   `prepare_motors()`, `park_position()`, `maintenance_position()`, `mount_position()`, `unmount_position()`,
   `generate_position_snapshot()`, `unpark_motors()`, and `park_motors()`.
4. **Update the `ZaberPositions` dataclass** in `sollertia_experiment/mesoscope_vr/system.py`
   to capture the new group's per-axis positions.
5. **Bump the `sollertia-experiment` version** and regenerate YAMLs.
6. **Update this skill's hardware-subsystem table.**

For motor-side mechanics (checksum validation, parking, position storage), see
`experiment:zaber-interface`.
