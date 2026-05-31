# Layer patterns and cross-layer contracts

Detailed authoring patterns for the three layers of a Sollertia acquisition system, plus the
cross-layer contracts that hold the architecture together. Loaded on demand from
`acquisition-system-design`'s SKILL.md.

---

## Layer 1: System configuration

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
- Always restore the original in-memory state in a `finally` block. Otherwise, an in-flight save
  followed by another access to the dataclass instance returns the wrong type.
- This pattern is needed only when the YAML "natural" shape differs from the in-memory shape. Most
  fields don't need it.

### Module-level helpers

Every system configuration module SHOULD expose three module-level helper functions for
configuration file lifecycle:

```python
def create_system_configuration_file(system: AcquisitionSystems | str = ...) -> None:
    """Creates the .YAML configuration file for the data acquisition system."""

def get_system_configuration_path() -> Path:
    """Returns the expected path to the local system configuration YAML."""

def get_system_configuration() -> <System>SystemConfiguration:
    """Loads the local configuration file and verifies it belongs to this acquisition system."""
```

**Rules:**
- `create_system_configuration_file` removes any pre-existing `*_system_configuration.yaml` in the
  target directory so exactly one file remains. It writes the defaults; the user edits afterward.
- `get_system_configuration_path` returns the canonical path under `get_working_directory()` (from
  `sollertia_shared_assets`). The system configuration ALWAYS lives in
  `<working_directory>/configuration/<system>_system_configuration.yaml`.
- `get_system_configuration` raises a clear error (via `console.error`) if the file does not exist,
  pointing the user at the system's `configure` CLI command (`sle mesoscope configure` for Mesoscope-VR).
- Only `create_system_configuration_file` accepts the `AcquisitionSystems` enum (defaulting to the
  package's supported system) and rejects any other value; the two getters take no arguments.
  `get_system_configuration` additionally verifies that the loaded file belongs to this acquisition
  system.

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

```text
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

**Antipattern:** ambiguous parameter names. `lick_threshold: int` is ambiguous (is it ADC units?
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

**Antipattern:** Using `Any` or untyped fields. Every field has an explicit, narrow type.

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

Lanes that wrap a third-party SDK connection (e.g., the Zaber motor lane) may expose `connect` (in
`__init__`) / `disconnect` plus position methods (`restore_position`, `park` / `unpark`) in place of
`start` / `stop`.

---

## Layer 3: Lifecycle orchestrator

The lifecycle orchestrator is one class (typically named `<System>System` or `<System>VRSystem`)
per acquisition system that composes the Layer-2 binding classes and owns the master start/stop.

This skill documents the *contract* the orchestrator must honor; the specific state machine and
runtime modes for any given system are documented in that system's behavior/runtime skill.

### Construction order

When the orchestrator builds its binding classes, the order MUST be:

```text
DataLogger(...)              ── instantiated first (owns the output directory each controller registers against)
    ↓
MQTTCommunication() ──── if the system uses MQTT (e.g., for Unity-VR communication)
    ↓
MicroControllerInterfaces(...)
    ↓
VideoSystems(...)
    ↓
<other lane bindings>... (any other lanes — motors, custom devices)
    ↓
DataLogger.start(), then .start() on each binding class, in the same order
```

Reasons for this order:
- The DataLogger MUST be instantiated before any controller is constructed, because each
  `MicroControllerInterface.__init__` writes a manifest entry to the DataLogger's output directory.
  Its communication process is then started first (before any binding class) inside the orchestrator's
  `start()`.
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
```

The reverse-order rule is non-negotiable: each binding class's `stop()` may write final messages to
the DataLogger, so the DataLogger must outlive every consumer. After shutdown, the downstream
preprocessing pipeline calls `assemble_log_archives()` (from `ataraxis_data_structures`) to
consolidate the per-source log entries into the final session output; this runs as a separate
preprocessing step, not inside the orchestrator's `stop()`.

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
