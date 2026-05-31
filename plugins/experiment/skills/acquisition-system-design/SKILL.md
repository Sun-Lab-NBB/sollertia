---
name: acquisition-system-design
description: >-
  Documents the platform-general design pattern for a Sollertia data acquisition system: a top-level
  YAML system configuration composed of per-hardware-lane calibration dataclasses, per-lane binding
  classes that compose the lower-level interface instances and run their lifecycle, and a runtime
  orchestrator that owns the master start/stop. Use when designing a new acquisition system from
  scratch, adding a new hardware lane to an existing system, or auditing an existing system's
  configuration/binding layer for pattern compliance.
user-invocable: false
---

# Acquisition system design

Documents the platform-general design pattern for a Sollertia data acquisition system at the
configuration and binding-class layer. An acquisition system is the host-PC stack that composes
lower-level hardware interfaces (microcontrollers, cameras, motors, etc.) into a runnable platform
that produces session data.

This skill is a **pattern skill** — it documents the conventions and contracts that all Sollertia
acquisition systems share, but does not document any single system's specific composition. For
concrete instances, see the per-system skills (currently `experiment:mesoscope-vr`).

---

## Scope

**Covers:**
- Three-layer architecture (System Configuration YAML → Calibration dataclasses + Binding classes →
  Lifecycle orchestrator)
- Top-level system configuration pattern (`@dataclass` + `YamlConfig`, composition, validation, YAML roundtrip)
- Per-lane calibration dataclass pattern (naming, field conventions, units in field names)
- Per-lane binding class pattern (constructor signature, lifecycle methods, idempotency, `__del__` semantics)
- Module-level helpers for configuration file lifecycle (`create_*`, `get_*_path`, `get_*_data`)
- Cross-layer contracts (field naming consistency, schema versioning)
- Lifecycle ordering rules (DataLogger first, binding classes second, reverse on shutdown)
- Workflows for adding a new hardware lane, building a new acquisition system, extending an existing lane

**Does not cover** (delegated):
- The per-firmware-module Python wrapper layer (`cross_system/module_interfaces.py`) and slmc
  firmware Modules — see `experiment:microcontroller-interface`.
- Concrete Mesoscope-VR composition (the binding-class instances and their YAML field surface) — see
  `experiment:mesoscope-vr`.
- The platform-general runtime-behavior pattern (state machine, runtime loop, event dispatch) — see
  `experiment:acquisition-system-runtime`.
- Concrete Mesoscope-VR runtime behavior (state machine, training modes, CLI) — see
  `experiment:mesoscope-vr-runtime`.
- The Unity VR task driver lane — see `experiment:vr-driver-interface`.
- Low-level VideoSystem mechanics — see `ataraxis@video:camera-interface`.
- Low-level MicroControllerInterface mechanics — see `ataraxis@communication:microcontroller-interface`.
- Zaber motor interface mechanics — see `experiment:zaber-interface`.
- Per-session metadata, task templates, experiment configuration, server transfer — owned by the
  assets and forging plugins respectively.
- The `sollertia-shared-assets` enum/registry side of registering a new acquisition system (the
  `AcquisitionSystems` member, the dispatch registries, the per-system descriptor / hardware-state /
  experiment-config / raw-data dataclasses, and the experiment-config factory) — owned by the assets
  plugin's `/library-extension`.

---

## Three-layer architecture

A Sollertia acquisition system is composed of three layers, top-down:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Layer 1: System Configuration YAML                                             │
│  ─────────────────────────────────                                              │
│  <System>SystemConfiguration   ── persisted to disk as <system>_configuration.yaml │
│      ├── <System>FileSystem    ── filesystem paths (host-machine state)         │
│      ├── <System>Cameras       ── per-camera calibration                        │
│      ├── <System>MicroControllers ── per-microcontroller-module calibration     │
│      ├── <System>ExternalAssets ── third-party hardware (motors, MQTT, etc.)    │
│      └── auxiliary sections     ── e.g., external service IDs                   │
└────────────────────────────┬────────────────────────────────────────────────────┘
                             │ instantiates and parameterizes
┌────────────────────────────▼────────────────────────────────────────────────────┐
│  Layer 2: Per-hardware-lane binding classes                                     │
│  ─────────────────────────────────────────                                      │
│  VideoSystems(camera_configuration, data_logger, output_directory)              │
│      └── wraps N × VideoSystem instances + per-camera lifecycle                 │
│  MicroControllerInterfaces(microcontroller_configuration, data_logger)          │
│      ├── instantiates N × ModuleInterface subclasses from calibration fields    │
│      └── wraps N × MicroControllerInterface instances + per-controller lifecycle│
│  <Lane>Bindings(<lane>_configuration, ...)                                      │
│      └── one per hardware lane (motors, custom devices, etc.)                   │
└────────────────────────────┬────────────────────────────────────────────────────┘
                             │ composed and lifecycle-orchestrated by
┌────────────────────────────▼────────────────────────────────────────────────────┐
│  Layer 3: Lifecycle orchestrator (one per acquisition system)                   │
│  ─────────────────────────────────────────────────────────                      │
│  <System>System / <System>Runtime / data_acquisition module                     │
│      ├── owns DataLogger and MQTTCommunication (where applicable)               │
│      ├── constructs each Layer-2 binding class in the correct order             │
│      ├── coordinates start / stop / state transitions                           │
│      └── exposes the public runtime surface to CLI / GUI consumers              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

Each layer's conventions are documented below. The pattern is the same for every Sollertia
acquisition system; what differs between systems is the *specific* set of hardware lanes, the
*specific* calibration fields per lane, and the *specific* runtime states.

---

## Layer 1: System Configuration

The top-level system configuration is a single `@dataclass` that inherits from
`ataraxis_data_structures.YamlConfig` and composes one nested dataclass per concern (per hardware
lane, plus auxiliary host-state concerns like filesystem paths and external service IDs).

### Class definition pattern

```python
from dataclasses import dataclass, field
from ataraxis_data_structures import YamlConfig

@dataclass
class <System>SystemConfiguration(YamlConfig):
    """Defines the hardware and software asset configuration for the <System> data acquisition system."""

    name: str = "<system_name>"
    """The descriptive name of the data acquisition system."""

    filesystem: <System>FileSystem = field(default_factory=<System>FileSystem)
    """Stores the filesystem configuration."""

    cameras: <System>Cameras = field(default_factory=<System>Cameras)
    """Stores the video cameras configuration."""

    microcontrollers: <System>MicroControllers = field(default_factory=<System>MicroControllers)
    """Stores the microcontrollers configuration."""

    # Additional per-lane sections as needed: external_assets, sheets, etc.
```

**Rules:**
- The class is a plain `@dataclass` (NOT `slots=True`) because `YamlConfig`'s reflection-based
  serialization is incompatible with slots.
- Every nested section uses `field(default_factory=<Section>)` so each section's defaults apply when
  the field is absent from the YAML.
- Field docstrings (triple-quoted strings on the line after each field) describe the section's
  purpose. `YamlConfig` extracts these as YAML comments on write.
- The `name` field is a free-form human-readable system label and SHOULD differ from the dataclass
  name (e.g., `"mesoscope"` not `"mesoscope_vr_system"`).

### Optional `__post_init__` for normalization and validation

When the YAML's natural shape for a field differs from the in-memory shape (e.g., a calibration
table is naturally a YAML mapping but the binding class consumes it as a tuple), use
`__post_init__` to normalize on load:

```python
def __post_init__(self) -> None:
    """Normalizes the valve calibration data to a tuple representation and validates its shape."""
    if not isinstance(self.microcontrollers.valve_calibration_data, tuple):
        self.microcontrollers.valve_calibration_data = tuple(
            (open_time, volume)
            for open_time, volume in self.microcontrollers.valve_calibration_data.items()
        )
    # ... shape validation ...
```

**Rules:**
- Normalization is one-directional: YAML shape → in-memory shape. The reverse conversion happens in
  the `save()` override (see below).
- Validation that catches user error (wrong field types, missing required relationships between
  fields) belongs here. Use `console.error(message=..., error=TypeError)` from
  `ataraxis_base_utilities` so the failure mode is consistent with the rest of the platform.
- Do NOT make `__post_init__` call out to MCP tools, the filesystem, or any non-local resource. It
  runs on every YAML load and must be fast.

### Optional `save()` override for YAML roundtrip hooks

When the YAML representation should differ from the in-memory representation, override `save()`:

```python
def save(self, path: Path) -> None:
    """Saves the instance to disk as a .yaml file.

    The valve_calibration_data tuple is temporarily converted to a dict for serialization so that
    existing .yaml files retain their mapping layout for the calibration table, then restored.
    """
    original_value = self.microcontrollers.valve_calibration_data
    try:
        if isinstance(original_value, tuple):
            self.microcontrollers.valve_calibration_data = dict(original_value)
        self.to_yaml(file_path=path)
    finally:
        self.microcontrollers.valve_calibration_data = original_value
```

**Rules:**
- Always restore the original in-memory state in a `finally` block. Otherwise an in-flight save
  followed by another access to the dataclass instance returns the wrong type.
- This pattern is needed only when the YAML "natural" shape differs from the in-memory shape. Most
  fields don't need it.

### Module-level helpers

Every system configuration module SHOULD expose three module-level helper functions for
configuration file lifecycle:

```python
def create_<system>_configuration_file(system: AcquisitionSystems | str = ...) -> None:
    """Creates the .YAML configuration file for the <system> data acquisition system."""

def get_<system>_configuration_path() -> Path:
    """Returns the expected path to the local <system> system configuration YAML."""

def get_<system>_configuration_data() -> <System>SystemConfiguration:
    """Resolves the local configuration file path and loads the configuration data."""
```

**Rules:**
- `create_*` removes any pre-existing `*_system_configuration.yaml` in the target directory so
  exactly one file remains. It writes the defaults; the user edits afterwards.
- `get_*_path` returns the canonical path under `get_working_directory()` (from
  `sollertia_shared_assets`). The system configuration ALWAYS lives in
  `<working_directory>/configuration/<system>_system_configuration.yaml`.
- `get_*_data` raises a clear error (via `console.error`) if the file does not exist, pointing the
  user at the `create_*` CLI command.
- All three functions accept the system enum from `sollertia_shared_assets.AcquisitionSystems` and
  reject any value other than the package's supported system.

---

## Layer 2a: Per-lane calibration dataclasses

For each hardware lane (cameras, microcontrollers, motors, external assets), the system configuration
embeds a per-lane calibration dataclass. Each dataclass captures the **physical parameters** of the
lane's hardware that the binding class uses to instantiate per-device wrappers.

### Class definition pattern

```python
from dataclasses import dataclass

@dataclass(slots=True)
class <System><Lane>:
    """Stores the <lane> configuration of the <System> data acquisition system."""

    <device>_<parameter>_<unit>: <type> = <default>
    """One-line description of what the field parameterizes, with units explicit where applicable."""
```

**Rules:**
- Use `@dataclass(slots=True)`. The per-lane dataclasses don't inherit from `YamlConfig`, so slots
  are safe and reduce memory.
- Class name follows the `<System><Lane>` convention: `MesoscopeCameras`, `MesoscopeMicroControllers`,
  `MesoscopeVRAssets`, etc.
- Each field has an explicit default — the YAML loader uses defaults when a field is absent.
- Each field has a triple-quoted docstring immediately after it. `YamlConfig` extracts these and
  writes them as YAML comments.

### Field naming convention

Field names encode three pieces of information separated by underscores:

```
<device-or-module>_<parameter>_<unit>
```

| Component            | Examples                                                        | Notes                                                              |
|----------------------|-----------------------------------------------------------------|--------------------------------------------------------------------|
| `<device-or-module>` | `face_camera`, `lick`, `torque`, `wheel_encoder`                | The device's role in the system, not the underlying hardware brand |
| `<parameter>`        | `index`, `threshold`, `delta_threshold`, `polling_delay`, `ppr` | The semantic name of the calibration value                         |
| `<unit>`             | `adc`, `us`, `ms`, `cm`, `g_cm`, `pulse`                        | Only included when the unit is non-obvious from the parameter name |

**Examples:**
- `face_camera_index: int` — index, no unit needed (it's an integer position)
- `lick_threshold_adc: int` — ADC units explicit because thresholds could be in volts or millivolts
- `wheel_encoder_polling_delay_us: int` — microseconds explicit
- `wheel_diameter_cm: float` — centimeters explicit

**Anti-pattern:** ambiguous parameter names. `lick_threshold: int` is ambiguous (is it ADC units?
millivolts? a count?). Always disambiguate via the unit suffix when the unit is non-obvious.

### Field type conventions

| Type                      | When to use                                                                                                    |
|---------------------------|----------------------------------------------------------------------------------------------------------------|
| `int`                     | Counts, ADC units, frame rates, encoder pulse counts, durations in microseconds                                |
| `float`                   | Physical measurements (cm, g·cm), conversion ratios, calibration coefficients                                  |
| `str`                     | Filesystem paths (use `Path` if the path is consumed as a Path; `str` if as a string), serial ports, hostnames |
| `bool`                    | Behavior flags (`report_ccw`, `report_cw`)                                                                     |
| `Path`                    | Filesystem paths consumed directly as `Path` objects                                                           |
| `tuple[tuple[T, T], ...]` | Lookup tables (e.g., valve calibration: pulse duration → volume)                                               |
| `<EnumType>`              | Enum values for encoder presets, pixel formats, etc.                                                           |

**Anti-pattern:** Using `Any` or untyped fields. Every field has an explicit, narrow type.

### Defaults convention

Defaults represent **factory defaults that work for the most common deployment** — usually values
that produced a working system in the reference rig. They are NOT magic numbers; they are calibrated
starting points the user overrides per-host.

- Numeric defaults SHOULD be the values measured on the reference rig (e.g.,
  `wheel_diameter_cm: float = 15.0333` is the actual diameter of the reference wheel).
- `Path()` (empty path) is the default for any filesystem field — the user MUST set it for each
  deployment; an empty path triggers a mount-check failure at load time.
- Serial-port defaults SHOULD be a representative USB device path for the reference platform
  (e.g. `/dev/ttyACM0`/`/dev/ttyUSB0` on Linux, `COMx` on Windows) — the value is OS-specific and is
  expected to be overridden per host.

---

## Layer 2b: Per-lane binding classes

For each hardware lane, the acquisition system has one binding class that composes the lane's
per-device wrappers and orchestrates their lifecycle.

### Class definition pattern

```python
class <System><Lane>Bindings:
    """Interfaces with the <lane> devices used in the <system> data acquisition system."""

    def __init__(
        self,
        data_logger: DataLogger,
        <lane>_configuration: <System><Lane>,
        # optional: output_directory, previous_state, etc.
    ) -> None:
        # 1. Initialize the _started lifecycle flag
        self._started: bool = False

        # 2. Cache the configuration (sometimes needed for runtime parameter updates)
        self._configuration: <System><Lane> = <lane>_configuration

        # 3. Instantiate per-device wrappers using configuration fields
        self.<device_a>: <DeviceA>Interface = <DeviceA>Interface(
            <calibration_param>=<lane>_configuration.<field>,
            ...
        )
        # ... more wrappers ...

        # 4. Wrap them in the underlying low-level controller(s)
        self._<controller>: <Controller> = <Controller>(
            data_logger=data_logger,
            module_interfaces=(self.<device_a>, ...),
            ...
        )

    def __del__(self) -> None:
        """Ensures that all communication processes are terminated when the instance is garbage-collected."""
        self.stop()

    def start(self) -> None:
        """Starts the communication processes and configures hardware with runtime parameters."""
        if self._started:
            return
        # Start each underlying controller
        # Call initialize_local_assets() on each wrapper that has SharedMemoryArray
        # Send runtime parameter updates (set_parameters calls)
        self._started = True

    def stop(self) -> None:
        """Stops all communication processes and releases all reserved resources."""
        if not self._started:
            return
        self._started = False
        # Stop each underlying controller (this also resets the wrapped hardware)
```

**Rules:**
- **Constructor takes** `data_logger` first (when the lane logs to DataLogger), then the per-lane
  calibration dataclass, then any optional supplementary inputs (output directory, previous-session
  state snapshot, etc.). Order matters: `data_logger` is the most-shared dependency.
- **Per-device wrappers are public attributes** (`self.brake`, `self.lick`, `self._face_camera`).
  The convention is public attributes for wrappers the lifecycle orchestrator may directly access
  (e.g., to issue commands at runtime), private (`_underscore`) for internal-only wrappers.
- **Underlying low-level controllers are private** (`self._actor`, `self._face_camera`'s
  `VideoSystem`). The binding class owns their lifecycle; consumers go through the public per-device
  wrappers.
- **`__del__` calls `stop()`** to guarantee cleanup on garbage collection. Never rely on this for
  correctness — always call `stop()` explicitly — but use `__del__` as a safety net.
- **`_started` idempotency**: `start()` and `stop()` are no-ops when already in the target state.
  The lifecycle orchestrator may double-call them during error recovery; both calls must be safe.

### Calling order inside `start()`

The exact order matters:

1. **Start each underlying controller** (`MicroControllerInterface.start()`, `VideoSystem.start()`,
   `ZaberConnection.connect()`). This spawns the controller's communication subprocess.
2. **Call `initialize_local_assets()` on each wrapper with SharedMemoryArray.** This connects the
   parent process to the shared memory the wrapper created in `__init__`. Without this, the
   wrapper's `lick_count` / `delivered_volume` / etc. properties return stale data.
3. **Send runtime parameter updates** via `wrapper.set_parameters(...)` using values from
   `self._configuration`. The wrappers were instantiated with calibration parameters in `__init__`;
   `set_parameters` pushes the runtime parameter struct (which mirrors the firmware's
   `CustomRuntimeParameters`) to the device. These two happen at different times and serve different
   purposes — see `experiment:microcontroller-interface` for the wrapper-side detail.

If any of these three steps fails, the wrapper's runtime state is unsafe and `start()` MUST raise
rather than silently leaving `_started=False`.

### Lifecycle method conventions

| Method                       | When                                                                                             |
|------------------------------|--------------------------------------------------------------------------------------------------|
| `start()`                    | Mandatory. Idempotent. Brings the lane online.                                                   |
| `stop()`                     | Mandatory. Idempotent. Tears the lane down.                                                      |
| `start_<sub_device>()`       | Optional. When per-device lifecycle granularity is useful (e.g., starting one camera at a time). |
| `save_<sub_device>_frames()` | Optional. When the lane has a "saving" distinct from "acquiring" (cameras).                      |
| `restore_position()`         | Optional. When the lane has cross-session state (motors).                                        |
| `is_started` property        | Optional. Read-only view of `_started`.                                                          |

---

## Layer 3: Lifecycle orchestrator

The lifecycle orchestrator is one class (typically named `<System>System` or `<System>VRSystem`)
per acquisition system that composes the Layer-2 binding classes and owns the master start/stop.

This skill documents the *contract* the orchestrator must honor; the specific state machine and
runtime modes for any given system are documented in that system's behavior/runtime skill.

### Construction order

When the orchestrator builds its binding classes, the order MUST be:

```text
DataLogger.start()
    ↓
MQTTCommunication() ──── if the system uses MQTT (e.g., for Unity-VR communication)
    ↓
<System>MicroControllerInterfaces(...)
    ↓
<System>VideoSystems(...)
    ↓
<System>Bindings... (any other lanes — motors, custom devices)
    ↓
.start() on each, in the same order
```

Reasons for this order:
- DataLogger MUST be running before any controller is constructed, because each
  `MicroControllerInterface.__init__` writes a manifest entry to the DataLogger's output directory.
- Microcontrollers SHOULD start before cameras because some camera triggers come from
  microcontroller TTL output; the camera receiving a trigger from an unstarted controller is a
  startup race.
- Zaber motors SHOULD initialize before or after the others depending on whether their position is
  consumed by a downstream calibration step (e.g., reading current position to compute a Unity
  initial state).

### Shutdown order

Reverse of construction:

```text
.stop() on each binding class, in reverse order
    ↓
DataLogger.stop()
    ↓
assemble_log_archives()  ── consolidates per-source archives into the final session output
```

The reverse-order rule is non-negotiable: each binding class's `stop()` may write final messages to
the DataLogger, so the DataLogger must outlive every consumer.

### Keepalive enforcement

For systems that use microcontroller keepalive, the orchestrator (or the
`MicroControllerInterfaces` binding class on its behalf) is responsible for:

- Setting the `keepalive_interval` on each `MicroControllerInterface` from the configuration's
  `keepalive_interval_ms` field.
- Monitoring for keepalive-timeout errors (`KEEPALIVE_TIMEOUT` event code 10 on the kernel channel)
  and aborting the session if any controller misses its deadline.

This skill's `experiment:microcontroller-interface` counterpart documents the per-wrapper
keepalive surface; the orchestrator-side enforcement is a system-design responsibility.

### Cross-lane synchronization

When two lanes must agree at runtime (e.g., camera frame capture triggered by a microcontroller TTL),
the orchestrator wires the cross-lane signals. Examples:

- Camera trigger pin connected to a microcontroller output → orchestrator's runtime logic ensures
  the camera is acquiring before issuing the microcontroller's start-trigger command.
- Motor position needed to compute Unity initial state → orchestrator reads the position from the
  motor binding class and passes it to the Unity MQTT setup.

Cross-lane signaling lives in the orchestrator, NOT in the individual binding classes. Each binding
class stays oblivious to the existence of other lanes; the orchestrator is the only place that knows
the full hardware composition.

---

## Cross-layer contracts

Three contracts must hold for the architecture to function:

### Contract 1: Calibration field naming agreement

Each per-lane calibration dataclass field that feeds a wrapper constructor MUST match the wrapper's
keyword-argument name conceptually (allowing for unit-suffix differences):

| Dataclass field               | Wrapper constructor kwarg | Note                                    |
|-------------------------------|---------------------------|-----------------------------------------|
| `lick_threshold_adc`          | `lick_threshold`          | Unit suffix dropped at the API boundary |
| `wheel_encoder_ppr`           | `encoder_ppr`             | Slight rename allowed                   |
| `minimum_brake_strength_g_cm` | `minimum_brake_strength`  | Unit suffix dropped at the API boundary |
| `valve_calibration_data`      | `valve_calibration_data`  | Exact match                             |

When a wrapper changes its constructor signature, the corresponding dataclass field's name SHOULD
be updated in the same change set to maintain conceptual agreement. Drift here is allowed but
causes confusion during debugging.

### Contract 2: Schema versioning

Any change to a calibration dataclass field (add, remove, rename, type-change) is a **schema change**
and MUST be paired with a version bump on the package that owns the dataclass. Older YAML files
written against the old schema MUST fail to load (with a clear error) rather than silently producing
a misconfigured system.

`YamlConfig`'s default behavior is permissive — extra fields in the YAML are ignored, missing fields
fall back to defaults. Both are dangerous for calibration. The pattern is:

- Add a field: bump the consumer library's minor version. Older YAML files load with the new field
  at its default (acceptable if the default is safe; explicit error if not).
- Remove a field: bump the consumer library's major version. Older YAML files load with the
  removed field silently ignored (acceptable since it has no effect).
- Rename a field: bump the consumer library's major version. Treat as remove-and-add. Add a
  one-cycle deprecation step in `__post_init__` that detects the old name in the loaded dict and
  migrates it.
- Change a field's type or units: bump the consumer library's major version. Add validation in
  `__post_init__` that detects the old shape and raises a clear migration error.

### Contract 3: Lifecycle ordering

Every binding class assumes:
- DataLogger is running before `__init__` is called.
- The constructor's `data_logger` argument is the DataLogger instance to log to.
- `start()` is called before any per-device wrapper method that issues commands or reads data.
- `stop()` is called before DataLogger is stopped.

The orchestrator MUST honor this. Deviating from the order produces difficult-to-debug failures
(missing manifest entries, dropped initial parameters, hung communication processes).

---

## Workflows

### Adding a new hardware lane to an existing system

When an acquisition system gains a new category of hardware (e.g., a system that previously had
only microcontrollers gains a camera), follow these steps:

1. **Decide whether the lane belongs in an existing dataclass section or needs its own.** Cameras
   and microcontrollers always get their own sections. Discrete-device categories (a single MQTT
   broker, a single power supply) can fold into `<System>ExternalAssets`.

2. **Author the calibration dataclass.** Follow the [Layer 2a pattern](#layer-2a-per-lane-calibration-dataclasses).
   Fields names follow `<device>_<parameter>_<unit>`; every field has a default; every field has a
   docstring.

3. **Add the new section to the system configuration.** Use `field(default_factory=...)`. Bump the
   consumer library's version per [Contract 2](#contract-2-schema-versioning).

4. **Author the binding class.** Follow the [Layer 2b pattern](#layer-2b-per-lane-binding-classes).
   Constructor takes the new calibration dataclass; lifecycle methods follow the conventions above.

5. **Wire the binding class into the lifecycle orchestrator.** Add the construction in the correct
   order (per [Construction order](#construction-order)) and the teardown in the reverse order.

6. **Update the per-system instance skill.** Add a section documenting the new lane's calibration
   surface and binding-class composition (for Mesoscope-VR, this is `experiment:mesoscope-vr`).

7. **Regenerate the system configuration YAML.** Use the system's configuration tooling (e.g., the
   `write_system_configuration_tool` MCP tool, or the `sle mesoscope configure` CLI for Mesoscope-VR)
   to write a new configuration file from the updated defaults. Older deployments need their YAML
   files re-generated against the new schema.

### Building a new acquisition system from scratch

1. **Define the system's hardware composition.** List every hardware lane (microcontrollers,
   cameras, motors, external devices) and the per-lane device count.

2. **Register the system on the `sollertia-shared-assets` side.** Adding the
   `sollertia_shared_assets.AcquisitionSystems` enum value is only the first of several coupled
   touches — the new system also needs its `<System>HardwareState`, `<System>ExperimentConfiguration`,
   and `<System>RawData` dataclasses, the matching `HARDWARE_STATE_REGISTRY`,
   `EXPERIMENT_CONFIGURATION_REGISTRY`, and `SYSTEM_RAW_DATA_REGISTRY` entries, and an experiment-config
   factory, all guarded by the import-time `_assert_registry_coverage()` parity check. Hand the entire
   slsa-side recipe off to the assets plugin's `/library-extension` ("Adding a new `AcquisitionSystems`
   member") and bump that package's version. Do not stop at the enum value — a half-wired registry
   fails the parity check and the package will not import.

3. **Author the per-lane calibration dataclasses.** One per lane, in the new system's package
   (typically `<system>/configuration.py`).

4. **Author the system configuration class.** Inherits `YamlConfig`, composes the calibration
   dataclasses, includes a `name` field and any auxiliary sections (filesystem, sheets, etc.).

5. **Author the module-level helpers.** `create_<system>_configuration_file`,
   `get_<system>_configuration_path`, `get_<system>_configuration_data`.

6. **Author the per-lane binding classes.** One per lane, in `<system>/binding_classes.py`.

7. **Author the lifecycle orchestrator.** Typically in `<system>/data_acquisition.py`.

8. **Wire the system into the package's CLI.** Add `<system>` commands to the `sle` entry points.

9. **(Optional but recommended) Author dedicated agentic assets for the new system.** A new
   acquisition system optionally benefits from its own per-system instance skill in this plugin,
   documenting the system's hardware lanes, calibration field surface, binding-class composition, and
   lifecycle. Follow the structure of `experiment:mesoscope-vr`. The system runs without it, but
   omitting it leaves the system driveable yet undocumented for agents (and the pattern skills above
   keep pointing at Mesoscope-VR as the sole worked instance).

10. **(Optional but recommended) Author a per-system runtime skill** when the system has non-trivial
    runtime modes / state machines / training behaviors. Follow the structure of
    `experiment:mesoscope-vr-runtime`.

### Extending an existing lane (adding a new device to an existing dataclass)

This is the most common change. Examples: adding a new camera to `MesoscopeCameras`, adding a new
microcontroller-driven sensor to `MesoscopeMicroControllers`.

1. **Add the new fields** to the existing calibration dataclass, following the field-naming
   convention.

2. **Extend the binding class** to instantiate the new device's wrapper and include it in the
   underlying controller's `module_interfaces` tuple (or equivalent).

3. **Update the system-specific instance skill** to document the new device's calibration field
   surface.

4. **Bump the consumer library's version** per [Contract 2](#contract-2-schema-versioning).

5. **Regenerate the system configuration YAML** on every deployment.

For microcontroller-module additions, also follow `experiment:microcontroller-interface`'s
"Adding a paired Module + Interface" workflow first — the firmware-and-wrapper pair must exist
before the binding class can compose it.

---

## Auxiliary sections beyond hardware lanes

The system configuration also captures host-machine state that isn't a hardware lane:

| Section type        | Typical contents                                                       |
|---------------------|------------------------------------------------------------------------|
| Filesystem section  | `root_directory`, `nas_directory`, `server_directory`, etc. (`Path` fields) |
| External services   | Google Sheets IDs, Slack webhook URLs, hosted endpoint URLs           |
| Network settings    | MQTT broker IP and port for cross-process / cross-machine communication |
| Software paths      | Paths to third-party acquisition software the system orchestrates     |

These follow the same dataclass pattern as hardware lanes but don't have binding classes — they're
consumed directly by orchestrator code or by per-session setup steps.

**Filesystem fields rule:** Every filesystem field SHOULD be checked at configuration load time via
a `check_system_mounts_tool` (or equivalent) that verifies the path exists and is writable. The
check is system-specific; the pattern is that the system configuration's MCP tooling exposes a
mount-check entry point.

---

## Mesoscope-VR as a worked example

The Mesoscope-VR acquisition system is the current consumer of every pattern in this skill:

- **System Configuration**: `MesoscopeSystemConfiguration` in
  `sollertia_experiment/mesoscope_vr/system.py`. Composes 5 sections (filesystem, sheets,
  cameras, microcontrollers, assets) plus a top-level `name` field. Implements
  `__post_init__` for valve calibration tuple normalization and `save()` for tuple→dict YAML
  roundtrip.
- **Calibration dataclasses**: `MesoscopeFileSystem`, `MesoscopeGoogleSheets`, `MesoscopeCameras`,
  `MesoscopeMicroControllers`, `MesoscopeVRAssets`. All use `slots=True` and follow the
  field-naming convention.
- **Binding classes**: `MicroControllerInterfaces`, `VideoSystems`, `ZaberMotors` in
  `sollertia_experiment/mesoscope_vr/binding_classes.py`. Each follows the constructor + `start` +
  `stop` + `__del__` pattern.
- **Lifecycle orchestrator**: `_MesoscopeVRSystem` in
  `sollertia_experiment/mesoscope_vr/data_acquisition.py`.

For the Mesoscope-VR-specific surface — actual field names, calibration values, binding-class
composition details, modification workflows — see `experiment:mesoscope-vr`. For Mesoscope-VR's
runtime states, training modes, and CLI, see `experiment:mesoscope-vr-runtime`.

---

## Maintenance contract

This skill documents durable design patterns. It is updated when:

- A new layer is added to the architecture (e.g., a fourth layer between the system configuration
  and the binding classes).
- A new lifecycle method becomes mandatory (e.g., a new asynchronous-asset initialization phase).
- A new naming or unit convention is adopted across the platform.
- A cross-layer contract changes (e.g., schema-versioning rules are tightened).

This skill is NOT updated when:

- A specific acquisition system gains a new lane or field — that's the per-system instance skill's
  domain.
- A specific binding class's internal implementation changes — the *pattern* it follows is what's
  documented here.

When you're unsure whether a change belongs here or in a per-system skill, ask: "Does this apply
to every Sollertia acquisition system, or only this one?" The pattern skill answers "every"; per-
system skills answer "only this one."

---

## Related skills

| Skill                                              | Relationship                                                                                                                        |
|----------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `experiment:microcontroller-interface`             | The per-module wrapper layer that binding classes compose. Authoritative for slmc/sle conventions.                                  |
| `experiment:zaber-interface`                       | Shared Zaber motor interface mechanics. Binding classes that include motors compose this.                                           |
| `experiment:mesoscope-vr`                          | The current Mesoscope-VR worked instance of this pattern.                                                                           |
| `experiment:acquisition-system-runtime`            | The runtime-behavior counterpart to this static-composition pattern.                                                                |
| `experiment:mesoscope-vr-runtime`                  | Mesoscope-VR-specific runtime behavior (state machine, training modes, CLI). Built on this pattern.                                 |
| `experiment:vr-driver-interface`                   | The Unity VR task driver lane an acquisition system composes for VR coupling.                                                       |
| `ataraxis@video:camera-interface`                  | Low-level VideoSystem mechanics. Camera binding classes compose VideoSystem instances.                                              |
| `ataraxis@communication:microcontroller-interface` | Low-level MicroControllerInterface mechanics. Microcontroller binding classes compose these.                                        |
| `experiment:acquisition-system-setup`              | Post-flash hardware discovery used to populate system configuration fields.                                                         |
| `experiment:pipeline`                              | End-to-end acquisition-system lifecycle orchestration context.                                                                      |
| `assets:library-extension`                         | Owns the `sollertia-shared-assets` enum/registry recipe for a new system; step 2 of the build-a-new-system workflow hands off here. |

---

## Verification checklist

```text
When designing a new system or extending an existing one:

System configuration:
- [ ] Top-level class is @dataclass (not slots) and inherits from YamlConfig
- [ ] Composes per-lane dataclasses via field(default_factory=...)
- [ ] Has a `name` field with a free-form human-readable label
- [ ] Implements __post_init__ if YAML shape differs from in-memory shape
- [ ] Overrides save() if YAML roundtrip needs translation
- [ ] Module-level helpers exist: create_*, get_*_path, get_*_data
- [ ] Helper functions reject AcquisitionSystems values other than the supported system

Calibration dataclasses:
- [ ] @dataclass(slots=True)
- [ ] Class named <System><Lane>
- [ ] Every field follows <device>_<parameter>_<unit> naming
- [ ] Every field has an explicit type annotation
- [ ] Every field has a sensible default
- [ ] Every field has a triple-quoted docstring describing purpose + units
- [ ] Filesystem fields default to Path() (empty); user MUST set them

Binding classes:
- [ ] Constructor takes data_logger first, then calibration dataclass, then optional args
- [ ] _started: bool = False initialized first
- [ ] Per-device wrappers instantiated as public attributes
- [ ] Underlying low-level controllers instantiated as private attributes
- [ ] __del__ calls stop()
- [ ] start() is idempotent and follows the controller → initialize_local_assets → set_parameters order
- [ ] stop() is idempotent and resets _started before tearing down
- [ ] Lifecycle methods cite the controller's start/stop order requirements

Cross-layer contract:
- [ ] Calibration field names align conceptually with wrapper constructor kwarg names
- [ ] Consumer library version bumped on any schema change
- [ ] Per-system instance skill updated with the new lane / field surface

Lifecycle orchestrator:
- [ ] Constructs DataLogger → MQTTCommunication → binding classes in the documented order
- [ ] Calls .start() on each in the same order
- [ ] Calls .stop() in reverse order
- [ ] DataLogger stops only after every binding class has stopped
- [ ] Keepalive enforcement and cross-lane signaling live in the orchestrator, not the binding classes
```
