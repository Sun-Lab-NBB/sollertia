# sollertia-experiment extension seams

Carries the per-seam detail behind the extension seam table in [SKILL.md](../SKILL.md), seam by seam, in the order the
table numbers them. Paths are relative to `src/sollertia_experiment/` unless the citation names another repository.

---

## What is already shared

The `__all__` list of `cross_system/__init__.py` exports 40 names, and every one of them is available to any
acquisition system. Compose them rather than authoring equivalents.

| Module                    | Exported names                                                                                                                                                                                                                 |
|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `system_configuration.py` | `SystemConfiguration`, `register_system_configuration`, `create_system_configuration_file`, `get_system_configuration_path`, `get_system_configuration_data`                                                                   |
| `data_preprocessing.py`   | `BEHAVIOR_LOGGER_NAME`, `StorageDestination`, `StorageDestinations`, `assemble_session_logs`, `rename_session_videos`, `snapshot_surgery_data`, `push_session_data`, `delete_session_directories`, `migrate_session_directory` |
| `module_interfaces.py`    | `EncoderInterface`, `LickInterface`, `TorqueInterface`, `MesoscopeFrameTTLInterface`, `BrakeInterface`, `WaterValveInterface`, `ScreenInterface`, `GasPuffValveInterface`                                                      |
| `zaber_bindings.py`       | `ZaberAxis`, `ZaberConnection`, `CRCCalculator`, `discover_zaber_devices`, `get_zaber_devices_info`, `get_zaber_device_settings`, `set_zaber_device_setting`, `validate_zaber_device_configuration`                            |
| `google_sheet_tools.py`   | `SurgeryLog`, `WaterLog`                                                                                                                                                                                                       |
| `terminal_prompts.py`     | `wait_for_enter`, `request_confirmation`, `request_required_confirmation`, `request_text`, `request_selection`                                                                                                                 |
| `shutdown_tools.py`       | `run_shutdown_step`                                                                                                                                                                                                            |
| `project_tools.py`        | `get_version_data`, `get_project_experiments`                                                                                                                                                                                  |

Three Zaber classes stay out of that list deliberately. `ZaberDevice`, `ZaberDeviceSettings`, and
`ZaberValidationResult` are importable only by their full `cross_system/zaber_bindings.py` module path, and a device
is reached through `ZaberConnection.get_device()` in that same module.

---

## The configuration registry

Seams 1 through 5. The registration call is the only system-specific wiring, and everything downstream of it is shared.

### Seam 1, the registry itself

`_SYSTEM_CONFIGURATION_CLASSES` in `cross_system/system_configuration.py` maps each registered acquisition system to
its configuration class and starts empty. Each system's configuration module calls
`register_system_configuration(system, configuration_class)` at module scope, which resolves the first argument
through `AcquisitionSystems(str(system))` and stores the class. The call runs at import time, so the module that
defines the subclass has to be imported before any lifecycle helper resolves that system.

### Seam 2, the base class

`SystemConfiguration` in `cross_system/system_configuration.py` is a `YamlConfig` dataclass that supplies the common
registry type and a default `save()`. A system subclasses it once and composes its per-subsystem configuration
sections as nested dataclasses. `/acquisition-system-design` owns the composition conventions.

### Seam 3, the save hook

`SystemConfiguration.save(path)` in `cross_system/system_configuration.py` delegates to
`self.to_yaml(file_path=path)`. Override it only when the on-disk YAML layout has to differ from the in-memory one,
such as when a tuple is persisted as a mapping for a stable file layout.

### Seam 4, the file lifecycle

Three helpers need no per-system code, and all of them live in `cross_system/system_configuration.py` beside the
three seams above. The filename derives from the enum value through `_system_configuration_filename` as
`f"{system}_system_configuration.yaml"`.

| Helper                             | Behavior                                                                                                                                              | Failure                                                                                                                         |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `create_system_configuration_file` | Writes a default-constructed instance into the working directory's configuration folder, then unlinks every other `*_system_configuration.yaml` there | `ValueError` naming the requested system and listing the registered ones, or `"none"`                                           |
| `get_system_configuration_path`    | Globs `*_system_configuration.yaml` in that folder and returns the single match                                                                       | `FileNotFoundError` unless exactly one file matches, listing what it found and pointing at the system's own `configure` command |
| `get_system_configuration_data`    | Resolves the single file, matches its name against each registered system's canonical filename, and returns `from_yaml`                               | `ValueError` naming the file and listing the registered configuration filenames                                                 |

Two properties of `create_system_configuration_file` decide how a host stays consistent. The deletion loop runs
**after** the write succeeds, so a failed write leaves the host with its previous acquisition system identity rather
than none. The just-written file is identified by `existing.samefile(configuration_path)`, an inode comparison,
because a case-insensitive filesystem keeps a differently-cased directory entry that neither a name comparison nor a
resolved-path comparison recognizes.

### Seam 5, the typed accessor

`get_system_configuration_data` returns the shared base type, so a system adds one thin wrapper that calls it,
`isinstance`-checks the result against its own subclass, and raises `TypeError` when the host belongs to another
system. Follow the pattern of `get_system_configuration()` in `mesoscope_vr/system.py`.

---

## Preprocessing and storage

Seams 6 through 9. `/data-management` owns the operator-facing lifecycle, and the contracts below are the seam view.

`StorageDestination(name, session_path)` and `StorageDestinations(destinations=())` are the system-agnostic interface
the shared utilities operate on (`cross_system/data_preprocessing.py`). A system resolves its destinations from its
own configuration and hands over the resolved paths, and the utilities never read a configuration themselves.

`BEHAVIOR_LOGGER_NAME` is `"behavior"`, and `_LOG_DIRECTORY_NAME` derives `behavior_data_log` from it
(`cross_system/data_preprocessing.py`). A system that names its behavior `DataLogger` anything else produces a
directory that `assemble_session_logs` never finds. Every primitive in the table below lives in that same module.

| Primitive                    | Contract                                                                                                                                                                                                                                                                                                                                                   |
|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `assemble_session_logs`      | No-ops when the log directory is absent or empty, raises `RuntimeError` when the directory holds both `.npy` entries and `.npz` archives, archives in place, then renames the log directory onto `behavior_data`, removing a stale one with a WARNING so an interrupted run resumes                                                                        |
| `rename_session_videos`      | Resolves the camera manifest from `behavior_data/` or the un-archived log directory, returns early when it is absent, and renames each video to `<session_name>_<source.name>.mp4`. It needs no static source-ID map                                                                                                                                       |
| `snapshot_surgery_data`      | Writes the surgery record to the session's raw-data path and returns the open `SurgeryLog`. The caller owns it and closes it in a `try/finally` to release the SSL socket                                                                                                                                                                                  |
| `push_session_data`          | Empty destinations produce a WARNING and an early return that keeps the local copy. Otherwise it computes an xxHash3-128 directory checksum, fans out `transfer_directory(remove_source=False)` across a process pool, propagates exceptions through `future.result()`, and then deletes the **entire local session directory**, `processed_data` included |
| `delete_session_directories` | Destructive and irreversible. When confirmation is requested it warns, calls `request_confirmation(default=False)`, and returns `False` on abort                                                                                                                                                                                                           |
| `migrate_session_directory`  | Pulls the session with `verify_integrity=False`, copies the pulled `raw_data/session_data.yaml` back to the source project's host path, recreating the `raw_data` directory that preprocessing removed, sets `project_name`, saves, and returns a freshly reloaded `SessionData`                                                                           |

---

## Hardware bindings

Seams 10 through 12.

The eight `ModuleInterface` subclasses in `cross_system/module_interfaces.py` bind hardware families rather than
acquisition systems, so any system driving the same firmware modules composes them unchanged. The base
`ModuleInterface` auto-logs every message sent or received, `data_codes` select which received events additionally
reach `process_received_data()`, and `error_codes` map event codes that raise `RuntimeError` and abort the runtime.
`/microcontroller-interface` owns the full catalog of the eight wrappers and their firmware counterparts.

`initialize_local_assets` is a project-local convention rather than part of the upstream `ModuleInterface` ABC. It is
defined on `EncoderInterface`, `LickInterface`, `MesoscopeFrameTTLInterface`, `WaterValveInterface`, and
`GasPuffValveInterface` in `cross_system/module_interfaces.py`, and the system's own binding class calls it after
the microcontroller processes start. A new shared-memory interface that omits it raises `AttributeError` at that
call site.

The Zaber layer is a three-class hierarchy in `cross_system/zaber_bindings.py`, `ZaberConnection` for the port,
`ZaberDevice` for the controller, and `ZaberAxis` for the motor. Its persistent contract lives in the device's
non-volatile memory map `_ZaberSettings` in the same module. That map assigns `USER_DATA_0` to the label checksum,
`USER_DATA_1` to the shutdown flag, `USER_DATA_10` to the unsafe flag, and `USER_DATA_11`, `USER_DATA_12`, and
`USER_DATA_13` to the park, maintenance, and mount positions. A new system reuses that contract as written, and
`/zaber-interface` owns the operator-facing surface.

---

## Auxiliary shared assets

Seams 13 through 16.

`SurgeryLog` and `WaterLog` (`cross_system/google_sheet_tools.py`) assume the platform-wide sheet schema, so a new
system supplies only a credentials path, the sheet identifiers, and the animal and session identity. Both build their
own service from a service-account file scoped to the Sheets API, and both expose a `close()` that the caller
invokes. `/google-sheets-processing` owns the schema contract and the auth model.

`wait_for_enter`, `request_confirmation`, `request_required_confirmation`, `request_text`, and `request_selection`
(`cross_system/terminal_prompts.py`) are the only operator-interaction path a runtime uses.
`request_required_confirmation` has no default and re-prompts until an explicit yes or no, so it is the one a
high-stakes action uses.

`run_shutdown_step(description, step)` runs one teardown callable and catches `(Exception, KeyboardInterrupt)`,
echoing an ERROR so the later steps still run (`cross_system/shutdown_tools.py`). Wrap every step of a multi-asset
shutdown in it, and pass `description` as a gerund phrase.

`get_version_data()` returns the Python and sollertia-experiment versions, and
`get_project_experiments(project_directory)` returns the naturally sorted experiment names of a project
(`cross_system/project_tools.py`). Both are already system-neutral.

---

## The interfaces package

Seams 17 through 26. MCP registration is discovered, and CLI registration is hand-edited. Nothing in
`interfaces/__init__.py` needs editing, because its `__all__` list is empty.

### Automatic

| Seam                               | Mechanism                                                                                                                                                                             |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| MCP tool-module registration       | `_register_tool_modules()` globs `*_tools.py` in `interfaces/` in `sorted()` order and imports each match, so the `@mcp.tool()` decorators self-register (`interfaces/mcp_server.py`) |
| Shared server instance             | `mcp = MCPServer(name="sollertia-experiment")` is constructed once at import, and a tool module imports it from `.mcp_instance` (`interfaces/mcp_instance.py`)                        |
| YAML plumbing                      | `serialize`, `describe_dataclass`, `write_yaml_validated`, `read_yaml`, and `probe_writable` accept any `YamlConfig` subclass (`interfaces/mcp_instance.py`)                          |
| Transports                         | `run_server` handles `stdio` and `streamable-http`, and new tools inherit both (`interfaces/mcp_server.py`)                                                                           |
| Warning filter and QT env preamble | The `warnings.warn` override and the `QT_LOGGING_RULES` `setdefault` run once at the top of `interfaces/entry_points.py`, and every group and every spawned subprocess inherits them  |
| Hardware-agnostic discovery        | The six `sle get` commands and the seven agnostic tools serve every system unchanged (the `get` group in `interfaces/get.py` and the tool functions in `interfaces/get_tools.py`)     |

Two conventions the new tool module follows. A dict-returning tool signals failure with a single `"error"` key, and a
string-returning tool signals failure with a leading `"Error: "` prefix. A destructive or hardware-mutating tool
gates on a tri-state `Literal["yes","no"] | None` confirmation argument rather than a boolean, so a falsy default
never reaches the mutation. The `confirm` parameter of `set_zaber_device_setting_tool`
(`interfaces/get_tools.py`) is the worked example of that gate.

### Manual

| Edit                                 | Detail                                                                                                                                                                                                                             |
|--------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Register the CLI group               | Add one import and one `sle_cli.add_command(cmd=<system>)` call inside `_register_subcommands` (`interfaces/entry_points.py`)                                                                                                      |
| Author the CLI group module          | A new `interfaces/<system>.py` declaring a Click group and its own `CONTEXT_SETTINGS = {"max_content_width": 120}`, which is duplicated per module rather than imported (`interfaces/get.py`)                                      |
| Author the tool module               | A new `interfaces/<system>_tools.py`. The suffix is load-bearing and the call uses `glob` rather than `rglob`, so the module sits directly in `interfaces/` (`_register_tool_modules()` in `interfaces/mcp_server.py`)             |
| Shared session-parameter object      | A `run`-style group re-implements the frozen dataclass and `click.make_pass_decorator` pattern, which is private to one system's module (`_SharedSessionParameters` and `_pass_shared_parameters` in `interfaces/mesoscope_vr.py`) |
| Filesystem and camera health reports | The report helpers are private and typed against one system's configuration, so a new system re-implements them in its own tool module (`_check_path` through `_verify_single_camera` in `interfaces/mesoscope_vr_tools.py`)       |
| Session-file constants               | The raw-data directory name and the canonical session filenames are private module constants (`_ZABER_POSITIONS_FILENAME` through `_MESOSCOPE_SYSTEM_CONFIGURATION_FILENAME` in `interfaces/mesoscope_vr_tools.py`)                |

---

## The VR task package

Seam 27. `vr_task/` is the acquisition-system-agnostic interface to the Unity task, and its `__all__` holds six names
(`vr_task/__init__.py`). A system nests a `VRTaskConfiguration` in its own configuration, constructs a `VRTaskDriver`
for the session types in `SESSION_TYPES_USING_VR_TASK`
(`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`), and dispatches the typed events the driver
returns against its own hardware.

The driver never touches hardware, so every actuator response is consumer code. `/vr-driver-interface` carries the
full consumer-obligation list. That list spans the broker configuration, the task template, the expected scene name,
a running Editor, and a scene carrying a `"Linear"` controller. It also spans the position source converted to Unity
units, the interaction-sensor mapping, per-trial hardware parameters joined by `trial_names`, display power sequencing
around `setup()`, an interactive terminal operator, and the shutdown-isolation helper. `unity:task-prefabs` and
`unity:scene-setup` own the Unity half.

---

## What is not a seam

Seam 28 is the one entry in the table that has nothing to compose. The platform exposes no generic runtime base
class, and each system implements its own controller against the seams above, as the "Extending the Platform"
section of `sollertia-experiment/README.md` states. Treat that as a budgeted authoring task rather than a missing
abstraction.

Nothing in `cross_system` drives microscope hardware. Calibration and hardware positioning stay experimenter-operated
through the maintenance runtime GUI, so no agent instruction moves a motor or calibrates a valve.

Two Zaber behaviors read as omissions and are safety contracts. `ZaberAxis.home()` refuses to home a parked motor and
never auto-unparks (`cross_system/zaber_bindings.py`). `ZaberConnection.connect()` releases its runtime
assets with `shutdown_devices=False` on failure and re-raises, deliberately declining to park, so an operator is able
to move the stage by hand and free a head-fixed animal (`cross_system/zaber_bindings.py`). Keep the reason
attached whenever either one is restated.
