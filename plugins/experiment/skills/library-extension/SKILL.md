---
name: library-extension
description: >-
  Owns the extension path of sollertia-experiment and sollertia-micro-controllers: the acquisition-system
  configuration registry, the shared cross_system primitives a new system composes, the MCP tool-module and CLI
  group registration seams, the absence of a generic runtime base class, and the firmware module, controller
  target, and board family seams. Use when implementing a new acquisition system against the Mesoscope-VR worked
  example, adding a firmware module or controller board, or auditing which seams a half-built system still misses.
user-invocable: false
---

# Sollertia experiment library extension

Catalogues the seams a new acquisition system composes across sollertia-experiment and sollertia-micro-controllers, and
names the handoffs for the halves that other skills own.

You MUST read this entire skill before extending either library, then read
[references/sle-seams.md](references/sle-seams.md) or [references/slmc-seams.md](references/slmc-seams.md) for the
repository you are touching. You MUST run the verification checklist before reporting an extension complete.

---

## Scope

**Covers:**
- The acquisition-system configuration registry and the zero-code file lifecycle it unlocks
- The shared `cross_system` primitives a new system composes rather than rewrites
- The `interfaces/` seams: automatic MCP tool-module discovery and manual CLI group registration
- The absence of a generic runtime base class, and what a new system therefore writes from scratch
- The sollertia-micro-controllers seams: a new firmware module, a new controller target, a new board family, and
  the cross-repo constants that move together
- The ordered new-system workflow across four repositories, with its gating conditions
- The audit view: which seams a half-built system still misses, and what each omission looks like at runtime

**Does not cover:**
- The sollertia-shared-assets registry half, which is the `AcquisitionSystems` member, the three
  `AcquisitionSystems`-keyed dispatch registries and the `SYSTEM_SESSION_TYPES` association, the per-system record
  dataclasses, and the import-time contract checks. Owned by `assets:library-extension`
- The Mesoscope-VR worked example's concrete values. Owned by `mesoscope:mesoscope-vr`,
  `mesoscope:mesoscope-vr-runtime`, and `mesoscope:mesoscope-vr-snapshots`
- The static composition pattern for the configuration and binding layers, owned by `/acquisition-system-design`
- The runtime behavior pattern, owned by `/acquisition-system-runtime`
- The paired firmware Module and Interface conventions and the module catalog, owned by
  `/microcontroller-interface`
- Phase ordering across the whole build, owned by `/system-design-pipeline`
- The base ataraxis firmware and communication mechanics, owned by `microcontroller:firmware-module` and
  `communication:microcontroller-interface`

---

## What each repository owns

A new acquisition system touches four repositories. Each one owns a distinct half of the contract, and the two this
skill covers sit in the middle of that chain.

| Repository                                                                                 | What the new system contributes                                                                               | Owning skill                              |
|--------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| `sollertia-shared-assets` (`sollertia-shared-assets/src/sollertia_shared_assets/`, `slsa`) | An `AcquisitionSystems` member, per-system record dataclasses, and the dispatch registry entries              | `assets:library-extension`                |
| `sollertia-experiment` (`src/sollertia_experiment/`, `sle`)                                | A `SystemConfiguration` subclass, binding classes, a runtime controller, a CLI group, and a tool module       | This skill                                |
| `sollertia-micro-controllers` (`slmc/`)                                                    | Firmware modules and controller targets, only when the rig needs hardware the current firmware does not drive | This skill                                |
| `sollertia-virtual-reality`                                                                | The corridor scene and its task prefabs, when the system runs the VR task                                     | `unity:task-prefabs`, `unity:scene-setup` |

The `AcquisitionSystems` enum in `enums.py` currently holds one member, so Mesoscope-VR is the only registered
acquisition system. Read it as the worked example through `mesoscope:mesoscope-vr` and
`mesoscope:mesoscope-vr-runtime`, then mirror the shape and leave the values behind.

---

## The configuration registry

Every acquisition system stores its host-machine configuration in one YAML file, and the whole lifecycle of that file
is shared. A system subclasses `SystemConfiguration`, composes one nested dataclass per concern, and calls
`register_system_configuration()` at module scope. Both names live in `cross_system/system_configuration.py`. That
call populates the `_SYSTEM_CONFIGURATION_CLASSES` registry in the same module and is the only system-specific wiring
the file lifecycle requires.

Three shared helpers then work without further per-system code, all in `cross_system/system_configuration.py`.
`create_system_configuration_file` writes a default-constructed instance and afterwards unlinks every other
`*_system_configuration.yaml` in the directory. `get_system_configuration_path` raises `FileNotFoundError` unless
exactly one file matches its glob. `get_system_configuration_data` maps the resolved filename back to a registered
class and returns the loaded instance. The filename derives from the enum value through
`_system_configuration_filename`, so a system named `<system>` gets `<system>_system_configuration.yaml`.

`get_system_configuration_data` returns the shared base type. A system therefore adds one thin typed accessor that
narrows the return value and rejects a host belonging to another system, following the pattern of
`get_system_configuration()` in `mesoscope_vr/system.py`. Override `SystemConfiguration.save()` only when the on-disk
YAML layout must differ from the in-memory one. `/acquisition-system-design` owns the composition conventions for the
dataclass tree itself.

---

## Shared primitives a new system composes

The `__all__` list of `cross_system/__init__.py` exports 40 names, and a new system reuses them rather than
authoring equivalents. Six post-acquisition primitives live in `cross_system/data_preprocessing.py` and are all
system-agnostic: `assemble_session_logs`, `rename_session_videos`, `snapshot_surgery_data`, `push_session_data`,
`delete_session_directories`, and `migrate_session_directory`. Four take a `SessionData` and the other two take
resolved paths, so none of them needs a per-system branch. `/data-management` owns their individual contracts, and
[references/sle-seams.md](references/sle-seams.md) lists them beside the seam each one occupies.

One constant and one shared type decide whether those primitives find anything to work on. The behavior `DataLogger`
of every acquisition
system is named `"behavior"`, because `BEHAVIOR_LOGGER_NAME` derives the `behavior_data_log` directory that
`assemble_session_logs` looks for (`cross_system/data_preprocessing.py`). Long-term storage targets reach the
shared utilities as a `StorageDestinations` collection of `StorageDestination` records, and each system resolves the
paths from its own configuration (`cross_system/data_preprocessing.py`).

The hardware bindings are reusable in the same way. The eight `ModuleInterface` subclasses in
`cross_system/module_interfaces.py` are hardware-family bindings rather than system-specific code, so any system
driving the same modules composes them unchanged. `MesoscopeFrameTTLInterface` in the same module is an input-only
role specialization of the generic firmware `TTLModule`, and its name reads as system-specific while its binding is
not. The `ZaberAxis`, `ZaberDevice`, and `ZaberConnection` hierarchy (`cross_system/zaber_bindings.py`), the
`SurgeryLog` and `WaterLog` sheet readers (`cross_system/google_sheet_tools.py`), the terminal prompt helpers
(`cross_system/terminal_prompts.py`), the `run_shutdown_step` isolator (`cross_system/shutdown_tools.py`), and the
`get_version_data` and `get_project_experiments` helpers (`cross_system/project_tools.py`) are all system-neutral.

---

## The interfaces seams

The `interfaces/` package registers MCP tools and CLI commands through two mechanisms that behave differently, and
the asymmetry between them is the seam agents most often miss.

MCP registration is **discovered**. `_register_tool_modules()` globs `*_tools.py` inside `interfaces/` in `sorted()`
order and imports every match, so `@mcp.tool()` decorators register purely as an import side effect
(`interfaces/mcp_server.py`). A new system drops `interfaces/<system>_tools.py` into that directory, imports `mcp`
from `.mcp_instance` (`interfaces/mcp_instance.py`), and edits nothing else. The `_tools.py` suffix is
load-bearing, and the call uses `glob` rather than `rglob`, so the module must sit directly in `interfaces/`.

CLI registration is **hand-edited**. `_register_subcommands()` carries one explicit import and one `add_command` call
per group, and a third group requires editing that function (`interfaces/entry_points.py`). The group module
itself is new work: it declares its own `CONTEXT_SETTINGS`, because the constant is duplicated per module rather than
imported, following the pattern of `CONTEXT_SETTINGS` and the `get` group in `interfaces/get.py`.

Three further pieces of the package are reused unchanged. `run_server` handles both transports
(`interfaces/mcp_server.py`). The `write_yaml_validated` and `read_yaml` helpers take any `YamlConfig`
subclass as their `validator_cls`, while `serialize` accepts any value, `describe_dataclass` any dataclass type, and
`probe_writable` a directory path (`interfaces/mcp_instance.py`). The hardware-agnostic
discovery surface serves every system, meaning the `get` group in `interfaces/get.py` and the seven tools in
`interfaces/get_tools.py`. The private helpers of a system's own tool module are re-implemented rather than imported,
because they are typed against one system's configuration.

---

## There is no generic runtime base class

The platform exposes no runtime base class to subclass, so each system writes its own controller against the seams
above, as the "Extending the Platform" section of `sollertia-experiment/README.md` states. Budget for that work
explicitly, because it is the largest single item in a new system's build and no scaffolding shortens it.

`/acquisition-system-runtime` owns the contract that controller must satisfy, which is the two state axes, the
per-mode logic functions, the per-cycle loop, the typed-event dispatch, and the teardown ordering.
`/acquisition-system-design` owns the binding-class contract the controller composes. This skill owns only the fact
that the seams are the whole inheritance story.

---

## The sollertia-micro-controllers seams

Firmware work is needed only when the rig drives hardware that the current firmware modules do not cover. Three
distinct seams exist, and each carries a different obligation on the sollertia-experiment side.

| Seam                  | Firmware work                                                                                                                   | sollertia-experiment mirror                                                                   |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| New firmware module   | A new `slmc/src/<name>_module.h`, wired into the target selection block of `slmc/src/main.cpp`                                  | Required. A matching `ModuleInterface` carrying the same type, id, codes, and parameter order |
| New controller target | A `platformio.ini` environment, an `#elif` branch, and the `static_assert` message of the `#else` branch in `slmc/src/main.cpp` | Required. A `MicroControllerInterface(controller_id=...)` mirror                              |
| New board family      | A second non-`env:` template mirroring `[teensy41_base]` in `slmc/platformio.ini`                                               | Only when controller IDs, event codes, or the baud rate change                                |

Teensy 4.1 is the only board family for which the firmware currently builds, because every `[env:teensy41_*]`
environment inherits `board` and `monitor_speed` from that one template (`slmc/platformio.ini`). The firmware repository
ships no `library.json`, so its version lives in two places that MUST move together, `PROJECT_NUMBER` in `slmc/Doxyfile`
and `release` in `slmc/docs/source/conf.py`. [references/slmc-seams.md](references/slmc-seams.md) carries the step
lists, and `/microcontroller-interface` owns the paired-module conventions and the module catalog.

---

## Extension seam table

Column `Kind` reads `automatic` when a new system needs no edit to an existing file, and `manual` when it does.
Column `Counterpart` names the paired obligation in another repository.

| #   | Seam                                              | Kind                                         | File and symbol                                                                                                                                                                                                                                               | Counterpart                                                                                                                                                                                            |
|-----|---------------------------------------------------|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Acquisition-system configuration registry         | manual, one call at import                   | `_SYSTEM_CONFIGURATION_CLASSES` and `register_system_configuration` in `cross_system/system_configuration.py`                                                                                                                                                 | slsa: the `AcquisitionSystems` member in `enums.py` (`assets:library-extension`)                                                                                                                       |
| 2   | `SystemConfiguration` base class                  | manual, one subclass                         | the `SystemConfiguration` class in `cross_system/system_configuration.py`                                                                                                                                                                                     | slsa: the `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`, and `SYSTEM_RAW_DATA_REGISTRY` entries in `registries.py`                                                                    |
| 3   | `SystemConfiguration.save()` hook                 | manual, optional override                    | `SystemConfiguration.save()` in `cross_system/system_configuration.py`                                                                                                                                                                                        | none                                                                                                                                                                                                   |
| 4   | Configuration file lifecycle                      | automatic                                    | `create_system_configuration_file`, `get_system_configuration_path`, `get_system_configuration_data`, and `_system_configuration_filename`, all in `cross_system/system_configuration.py`                                                                     | none. The filename derives from the enum value                                                                                                                                                         |
| 5   | Typed configuration accessor                      | manual, one thin wrapper                     | pattern of `get_system_configuration()` in `mesoscope_vr/system.py`                                                                                                                                                                                           | none                                                                                                                                                                                                   |
| 6   | `StorageDestination` and `StorageDestinations`    | automatic, consumed                          | `StorageDestination` and `StorageDestinations` in `cross_system/data_preprocessing.py`                                                                                                                                                                        | none                                                                                                                                                                                                   |
| 7   | The six preprocessing primitives                  | automatic, composed                          | `assemble_session_logs` through `migrate_session_directory` in `cross_system/data_preprocessing.py`                                                                                                                                                           | none                                                                                                                                                                                                   |
| 8   | The behavior logger name                          | manual, one constant to honor                | `BEHAVIOR_LOGGER_NAME = "behavior"` and `_LOG_DIRECTORY_NAME` in `cross_system/data_preprocessing.py`                                                                                                                                                         | none. Miss it and `assemble_session_logs` finds no `behavior_data_log/`                                                                                                                                |
| 9   | Camera-manifest video renaming                    | automatic                                    | `rename_session_videos` in `cross_system/data_preprocessing.py`                                                                                                                                                                                               | the acquisition code writes a `CameraManifest`, and no static source-ID map is needed                                                                                                                  |
| 10  | The eight `ModuleInterface` subclasses            | automatic, reused                            | the eight subclasses in `cross_system/module_interfaces.py`                                                                                                                                                                                                   | slmc: the matching firmware modules in the target blocks of `slmc/src/main.cpp`                                                                                                                        |
| 11  | `initialize_local_assets` convention              | manual, on any new shared-memory interface   | `initialize_local_assets` on the five shared-memory interfaces in `cross_system/module_interfaces.py`                                                                                                                                                         | none. It is a project-local convention rather than part of the upstream ABC. A new wrapper is also added to the `module_interfaces` import block and the `__all__` list of `cross_system/__init__.py`. |
| 12  | The Zaber tri-class hierarchy                     | automatic, reused                            | `ZaberAxis`, `ZaberDevice`, and `ZaberConnection`, with the `_ZaberSettings` map, in `cross_system/zaber_bindings.py`                                                                                                                                         | none                                                                                                                                                                                                   |
| 13  | Google Sheet logs                                 | automatic, reused                            | `SurgeryLog` and `WaterLog` in `cross_system/google_sheet_tools.py`                                                                                                                                                                                           | slsa: `CredentialsTypes.GOOGLE` in `enums.py`                                                                                                                                                          |
| 14  | Terminal prompts                                  | automatic, reused                            | the five prompt helpers in `cross_system/terminal_prompts.py`                                                                                                                                                                                                 | none                                                                                                                                                                                                   |
| 15  | Shutdown-step isolation                           | automatic, reused                            | `run_shutdown_step` in `cross_system/shutdown_tools.py`                                                                                                                                                                                                       | none                                                                                                                                                                                                   |
| 16  | Project and version discovery                     | automatic, reused                            | `get_version_data` and `get_project_experiments` in `cross_system/project_tools.py`                                                                                                                                                                           | none                                                                                                                                                                                                   |
| 17  | MCP tool-module registration                      | **automatic**                                | `_register_tool_modules()` in `interfaces/mcp_server.py`, create `interfaces/<system>_tools.py`                                                                                                                                                               | none. The `_tools.py` suffix is load-bearing and the module sits directly in `interfaces/`                                                                                                             |
| 18  | Shared MCP server instance                        | automatic, imported                          | `mcp = MCPServer(name="sollertia-experiment")` in `interfaces/mcp_instance.py`                                                                                                                                                                                | none                                                                                                                                                                                                   |
| 19  | YAML read, write, validate, describe helpers      | automatic, reused                            | `serialize`, `describe_dataclass`, `write_yaml_validated`, `read_yaml`, and `probe_writable` in `interfaces/mcp_instance.py`                                                                                                                                  | none                                                                                                                                                                                                   |
| 20  | MCP transports                                    | automatic                                    | `run_server` in `interfaces/mcp_server.py`                                                                                                                                                                                                                    | none                                                                                                                                                                                                   |
| 21  | CLI group registration                            | **manual**, one import and one `add_command` | `_register_subcommands` in `interfaces/entry_points.py`                                                                                                                                                                                                       | none. There is no glob discovery on the CLI side                                                                                                                                                       |
| 22  | CLI group module                                  | manual, new file                             | pattern of the `get` group in `interfaces/get.py` and the `mesoscope` group in `interfaces/mesoscope_vr.py`. `CONTEXT_SETTINGS` is duplicated per module                                                                                                      | none                                                                                                                                                                                                   |
| 23  | Shared session-parameter object for a `run` group | manual, re-implement                         | the pattern is private, `_SharedSessionParameters` and `_pass_shared_parameters` in `interfaces/mesoscope_vr.py`                                                                                                                                              | none                                                                                                                                                                                                   |
| 24  | Filesystem and camera health report helpers       | manual, re-implement                         | private and typed against one system's configuration, `_check_path` through `_verify_single_camera` in `interfaces/mesoscope_vr_tools.py`                                                                                                                     | none                                                                                                                                                                                                   |
| 25  | Session-file constants                            | manual, re-declare                           | private module constants, `_ZABER_POSITIONS_FILENAME` through `_MESOSCOPE_SYSTEM_CONFIGURATION_FILENAME` in `interfaces/mesoscope_vr_tools.py`                                                                                                                | none                                                                                                                                                                                                   |
| 26  | Hardware-agnostic discovery CLI and tools         | automatic, reused unchanged                  | the `get` command group in `interfaces/get.py` and the seven tools in `interfaces/get_tools.py`                                                                                                                                                               | none                                                                                                                                                                                                   |
| 26a | Sphinx API documentation entry                    | manual, one block                            | a `.. automodule:: sollertia_experiment.<system>` section in `docs/source/api.rst`, following its "Mesoscope-VR Acquisition System" section                                                                                                                   | none. Omit it and the new package renders no API documentation                                                                                                                                         |
| 27  | The VR task package                               | automatic, composed                          | the `__all__` list of `vr_task/__init__.py`, consumer obligations in `/vr-driver-interface`                                                                                                                                                                   | unity: a scene carrying a `"Linear"` controller and a running Editor                                                                                                                                   |
| 28  | The runtime controller                            | **manual, from scratch**                     | no base class exists, per the "Extending the Platform" section of `sollertia-experiment/README.md`                                                                                                                                                            | none                                                                                                                                                                                                   |
| 29  | slmc: a new firmware module                       | manual, nine steps                           | `slmc/src/<name>_module.h`, wired into the target block of `slmc/src/main.cpp` as an `#include`, an instantiation, and a `modules[]` entry                                                                                                                    | sle: a matching `ModuleInterface` carrying the same type, id, codes, and parameter order                                                                                                               |
| 30  | slmc: a new controller target                     | manual, three steps                          | the `[env:teensy41_actor]` pattern in `slmc/platformio.ini`, plus the `#elif` branch, `kControllerID`, `modules[]`, and `static_assert` of `slmc/src/main.cpp`                                                                                                | sle: a `MicroControllerInterface(controller_id=...)` mirror, pattern in `MicroControllerInterfaces.__init__` in `mesoscope_vr/binding_classes.py`                                                      |
| 31  | slmc: a new board family                          | manual, five steps                           | a second non-`env:` template mirroring `[teensy41_base]` in `slmc/platformio.ini`, plus one `[env:<board>_<target>]` per target                                                                                                                               | sle: only when controller IDs, event codes, or the baud rate change                                                                                                                                    |
| 32  | slmc: the eight cross-repo constants              | manual, moved together                       | `kKeepaliveInterval`, `kSerialBaudRate`, `kControllerID`, and the `(type, id)` pairs in `slmc/src/main.cpp`, the status codes, command codes, and parameter-struct layouts of the module headers, and `kDefaultCalibrationCount` in `slmc/src/valve_module.h` | sle mirrors listed per row in [references/slmc-seams.md](references/slmc-seams.md)                                                                                                                     |

---

## Workflow: building a new acquisition system

Nine steps, each with a gate. A gate that does not hold blocks the next step.

### Step 1: Shared-assets contract

Hand off to `assets:library-extension` for the `AcquisitionSystems` member, the `<System>HardwareState`,
`<System>ExperimentConfiguration`, and `<System>RawData` classes, the four registry entries, and the
`SYSTEM_SESSION_TYPES` claim. A new session type the system runs is registered in the same handoff, because its
`SessionTypes` member and its `SYSTEM_SESSION_TYPES` claim both live there.
**Gate:** `python -c "import sollertia_shared_assets"` succeeds and the system appears in
`list_supported_acquisition_systems_tool`.

### Step 2: Configuration layer

Subclass `SystemConfiguration`, compose one nested dataclass per concern, call `register_system_configuration()` at
module scope, and add the typed `get_system_configuration()` accessor plus the zero-argument
`create_system_configuration_file()` wrapper. Design conventions come from `/acquisition-system-design`.
**Gate:** `create_system_configuration_file()` writes `<system>_system_configuration.yaml` and
`get_system_configuration_data()` loads it back.

### Step 3: Hardware interface layer

Reuse the eight `ModuleInterface` subclasses (`cross_system/module_interfaces.py`) and the Zaber hierarchy
(`cross_system/zaber_bindings.py`). The camera layer has no shared wrapper: a system composes the upstream
`ataraxis-video-system` `VideoSystem` into its own binding class, following `VideoSystems` in
`mesoscope_vr/binding_classes.py`, with conventions from `video:camera-interface`. Hand off to
`/microcontroller-interface` for the paired-module conventions, and to
[references/slmc-seams.md](references/slmc-seams.md) for the firmware seam, only when the rig needs hardware that none
of them covers.
**Gate:** every module the system needs has a wrapper with a matching firmware counterpart.

### Step 4: Binding classes and orchestrator

Follow `/acquisition-system-design` for the binding-class contract and `/acquisition-system-runtime` for the runtime
contract. There is no base class to subclass.
**Gate:** the orchestrator constructs, starts, and stops with the DataLogger outliving every consumer.

### Step 5: VR task wiring

Nest a `VRTaskConfiguration` in the system configuration and construct a `VRTaskDriver` for the session types in
`SESSION_TYPES_USING_VR_TASK` (`registries.py`). Satisfy the consumer obligations in `/vr-driver-interface`. Hand
off to `unity:task-prefabs`, `unity:task-scenes`, and `unity:scene-setup` for the Unity side.
**Gate:** `setup()` completes against a running Editor.

### Step 6: Preprocessing

Name the behavior logger `"behavior"`, resolve `StorageDestinations` from the system's own configuration, and compose
the six shared primitives, adding system-specific steps around them. Conventions come from `/data-management`. A
system's CLI and tool module call three session-lifecycle entry points, preprocess, purge, and migrate, which are
written from scratch in `<system>/data_preprocessing.py`. `cross_system` supplies only the six single-session primitives
they compose (`cross_system/data_preprocessing.py`).
**Gate:** a session preprocesses and pushes with no local copy left behind.

### Step 7: Interfaces

Author `interfaces/<system>.py` with its own `CONTEXT_SETTINGS` and command group, add the import and the
`add_command` call inside `_register_subcommands`, and author `interfaces/<system>_tools.py`, which registers itself.
Add a `.. automodule:: sollertia_experiment.<system>` block to `docs/source/api.rst`, following its
"Mesoscope-VR Acquisition System" section.
**Gate:** `sle <system> --help` prints and the new tools appear on `sle mcp`.

### Step 8: Agentic assets

Create `plugins/<system>/` with its `.claude-plugin/plugin.json`, add its entry to
`.claude-plugin/marketplace.json`, author its session-schema, experiment-schema, snapshot, instance, and runtime
skills mirroring the mesoscope plugin, and add a row to `/acquisition-system-setup`'s supported-systems table.
**Gate:** every per-system pointer in a generic skill resolves.

### Step 9: Downstream

Hand off to `forging:data-processing-design` and `forging:dataset-definition` for the sollertia-forgery registry
entries the system needs before its sessions are processed or forged.
**Gate:** the handoffs are listed in the pull request description.

---

## Guardrails and what nothing checks

sollertia-experiment runs no import-time contract check of its own. The only fail-fast guard in the chain is the three
assertions at the bottom of `registries.py`, `_assert_registry_coverage()`, `_assert_descriptor_contract()`, and
`_assert_experiment_configuration_contract()`, which cover the shared-assets half and which `assets:library-extension`
owns. Every omission below therefore surfaces at runtime, or silently, and each one is verified by hand.

| Omission                                                        | How it surfaces                                                                                                                                                                           |
|-----------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `register_system_configuration()` never called                  | `create_system_configuration_file` raises `ValueError` listing the registered systems, or `"none"` (`cross_system/system_configuration.py`)                                               |
| Two configuration files present on one host                     | `get_system_configuration_path` raises `FileNotFoundError`, because more than one file matches the glob (`cross_system/system_configuration.py`)                                          |
| Behavior logger named anything other than `"behavior"`          | `assemble_session_logs` finds no `behavior_data_log/` and silently no-ops (`cross_system/data_preprocessing.py`)                                                                          |
| No `CameraManifest` written during acquisition                  | `rename_session_videos` returns early and the videos keep their source-ID filenames (`cross_system/data_preprocessing.py`)                                                                |
| A runtime that never writes the system-configuration snapshot   | Nothing in sollertia-experiment checks it. The gap surfaces only when a caller compares the session directory against `SessionData.required_raw_assets()` (`session_data.py`)             |
| A runtime that never writes `hardware_state.yaml`               | Nothing checks it anywhere. `required_raw_assets()` does not list it, so the omission stays silent until a downstream consumer resolves `RawData.hardware_state_path` (`session_data.py`) |
| Tool module named without the `_tools.py` suffix, or nested     | The `*_tools.py` glob never imports it and the tools silently do not exist (`_register_tool_modules()` in `interfaces/mcp_server.py`)                                                     |
| CLI group not added to `_register_subcommands`                  | `sle <system>` is not a command, and nothing warns (`interfaces/entry_points.py`)                                                                                                         |
| A new shared-memory interface without `initialize_local_assets` | The binding class raises `AttributeError` at start (`MicroControllerInterfaces.start()` in `mesoscope_vr/binding_classes.py` shows the call site)                                         |
| Firmware and wrapper parameter structs disagree                 | Every field after the first mismatch is silently corrupted, because `PACKED_STRUCT` carries no padding                                                                                    |
| A new system missing from the supported-systems table           | `/pipeline` never routes to it at operate time, and nothing warns                                                                                                                         |

---

## Pitfalls

| Pitfall                                                           | Why it bites                                                                                                                                                   |
|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Assuming CLI registration is as automatic as MCP registration     | MCP tool modules are globbed, and CLI groups are two hand-written lines. The asymmetry is the single most missed seam                                          |
| Looking for a runtime base class to subclass                      | None exists. Each system writes its controller against the seams, so budget for that work                                                                      |
| Copying a private helper out of the worked example's tools module | The shared session-parameter object and the health-report helpers are private and typed against one system's configuration, so a new system re-implements them |
| Treating a role-specialized interface name as system coupling     | The eight `ModuleInterface` subclasses live in `cross_system` and bind hardware families, so any system driving the same modules reuses them                   |
| Changing a firmware constant on one side only                     | The eight constants in seam 32 move together, and a one-sided change produces a system that compiles and runs while its data is garbage                        |

---

## Related skills

The `microcontroller:`, `communication:`, and `automation:` entries below resolve through the ataraxis marketplace.
Every other entry resolves inside the sollertia marketplace.

| Skill                                     | Relationship                                                                                            |
|-------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `assets:library-extension`                | Owns the sollertia-shared-assets half. Step 1 hands off there and gates on its import check             |
| `assets:working-directory`                | Bootstraps the working directory, data root, credentials, and templates directory every seam consumes   |
| `assets:experiment-configuration`         | Owns authoring against a registered `<System>ExperimentConfiguration` once the extension lands          |
| `assets:task-templates`                   | Owns the `TaskTemplate` the VR task seam consumes                                                       |
| `assets:session-data`                     | Owns the session record the new system creates through `SessionData.create`                             |
| `/acquisition-system-design`              | Owns the configuration and binding-class design the configuration seam feeds                            |
| `/acquisition-system-runtime`             | Owns the runtime contract the controller written at seam 28 satisfies                                   |
| `/microcontroller-interface`              | Owns the paired Module and Interface conventions and the module catalog behind seams 10, 11, 29, and 30 |
| `/zaber-interface`                        | Owns the Zaber hierarchy reused at seam 12                                                              |
| `/vr-driver-interface`                    | Owns the VR task consumer obligations behind seam 27                                                    |
| `/google-sheets-processing`               | Owns the Sheets processors reused at seam 13                                                            |
| `/data-management`                        | Owns the six preprocessing primitives composed at seam 7                                                |
| `/acquisition-system-setup`               | Owns the supported-systems table a new system joins                                                     |
| `/system-design-pipeline`                 | Orchestrates the whole build and designates this skill as the sollertia-experiment and slmc phase owner |
| `/experiment-mcp-environment-setup`       | Owns the `sle mcp` server the new tool module joins                                                     |
| `mesoscope:mesoscope-vr`                  | The worked example's configuration and binding layer                                                    |
| `mesoscope:mesoscope-vr-runtime`          | The worked example's controller, CLI, and session lifecycle                                             |
| `mesoscope:mesoscope-vr-snapshots`        | The worked example's raw-data layout and per-session snapshots                                          |
| `forging:data-processing-design`          | Owns the per-system processing-stage design behind the sollertia-forgery registry entries               |
| `forging:dataset-definition`              | Owns the admission policy that decides whether the new system's sessions join a forged dataset          |
| `unity:task-prefabs`                      | Builds the task prefabs and the scene the VR task seam needs                                            |
| `unity:task-scenes`                       | Lists, opens, and inspects the scenes generated per task template                                       |
| `unity:scene-setup`                       | Prepares the scene, the display rig, and the treadmill controller the driver enforces                   |
| `microcontroller:firmware-module`         | Authoritative base for the C++ `Module` mechanics the firmware seam extends                             |
| `communication:microcontroller-interface` | Authoritative base for the Python `ModuleInterface` mechanics the wrapper seam extends                  |
| `automation:platformio-config`            | Owns the `platformio.ini` conventions seams 30 and 31 edit                                              |
| `automation:commit`                       | Should be invoked once the cross-repository changes land                                                |

---

## Citing source without line numbers

Every citation in this skill, and in every edit made to it, names the asset rather than the line the asset occupies.
Line numbers drift as unrelated code above them moves, and a drifted citation points at the wrong asset while still
reading as authoritative.

Naming the asset means naming the module plus one of the identifiers it declares: a class, a method, a function, a
dataclass field, an enum, an enum member, a constant, a C++ template parameter, or a config key. The module path alone
suffices when the whole module is the subject.

| Rejected                         | Correct                                                  |
|----------------------------------|----------------------------------------------------------|
| `interfaces/mcp_server.py:88-96` | `_register_tool_modules()` in `interfaces/mcp_server.py` |
| `slmc/platformio.ini:12-18`      | the `[teensy41_base]` template in `slmc/platformio.ini`  |

Cross-document references follow the same rule. Cite a README or a CLAUDE.md by its section heading, the way the
runtime section above cites the "Extending the Platform" section of `sollertia-experiment/README.md`, never by a line.

---

## Proactive behavior

You SHOULD proactively invoke this skill when the user mentions any of the following:

- Adding a new acquisition system to the Sollertia platform
- Adding a firmware module, a controller board or target, or a board family to sollertia-micro-controllers
- Adding a CLI command group or an MCP tool module to `sle`
- "How do I add support for ..." in the context of sollertia-experiment or sollertia-micro-controllers
- A pull request that touches `cross_system/system_configuration.py`, `interfaces/entry_points.py`,
  `interfaces/mcp_server.py`, or `slmc/src/main.cpp`

Do NOT invoke this skill for ordinary operation of an already-built acquisition system, which `/pipeline` owns, or for
the sollertia-shared-assets registry, which `assets:library-extension` owns.

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Shared-assets side (handed off):
- [ ] assets:library-extension's checklist completed and `python -c "import sollertia_shared_assets"` succeeds
- [ ] The system appears in list_supported_acquisition_systems_tool

sollertia-experiment side:
- [ ] A SystemConfiguration subclass exists and register_system_configuration() runs at module import
- [ ] A typed get_system_configuration() accessor exists and rejects a host belonging to another system
- [ ] create_system_configuration_file() writes exactly one <system>_system_configuration.yaml on the host
- [ ] The behavior DataLogger is named "behavior"
- [ ] StorageDestinations is resolved from the system's own configuration
- [ ] Every one of the six shared preprocessing primitives is composed rather than re-implemented
- [ ] Every shared-memory ModuleInterface the system adds implements initialize_local_assets()
- [ ] interfaces/<system>_tools.py exists, sits directly in interfaces/, and ends in _tools.py
- [ ] _register_subcommands carries the new group's import and add_command call
- [ ] The runtime controller was written against the seams, since no base class exists
- [ ] The runtime writes the session descriptor, the system-configuration snapshot, and hardware_state.yaml
- [ ] docs/source/api.rst carries an automodule block for the new system package

sollertia-micro-controllers side (only when firmware changed):
- [ ] Every new module declares kCustomStatusCodes from 51 and kModuleCommands from 1
- [ ] Every multi-stage command is non-blocking within the 500 ms keepalive
- [ ] Every new pin carries a LED_BUILTIN static_assert, and multi-pin modules assert pairwise distinctness
- [ ] main.cpp carries the include, the instantiation, and the modules[] entry for the right target
- [ ] A new target carries its platformio.ini env and its macro in the static_assert message
- [ ] All eight cross-repo constants moved together, verified row by row
- [ ] Doxyfile PROJECT_NUMBER and docs/source/conf.py release both match the intended release

Marketplace side:
- [ ] plugins/<system>/ exists with its .claude-plugin/plugin.json
- [ ] .claude-plugin/marketplace.json carries the new plugin entry
- [ ] The per-system schema, instance, and runtime skills exist and every generic-skill pointer resolves
- [ ] /acquisition-system-setup's supported-systems table carries a row for the new system
- [ ] Every cross-plugin link resolves to a skill on the authoritative roster
- [ ] Every source citation added to a skill names the asset or the section heading rather than a line

Downstream:
- [ ] forging registry entries handed off and listed in the pull request description
- [ ] sollertia-experiment version bumped and its sollertia-shared-assets pin updated
```
