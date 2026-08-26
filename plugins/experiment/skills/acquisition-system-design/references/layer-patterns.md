# Layer patterns and cross-layer contracts

Detailed authoring patterns for the three layers of a Sollertia acquisition system, plus the
cross-layer contracts that hold the architecture together. Loaded on demand from
`acquisition-system-design`'s SKILL.md.

---

## Layer 1: System configuration

The top-level system configuration is a single `@dataclass` that inherits from `ataraxis_data_structures.YamlConfig` and
composes one nested dataclass per concern (per hardware subsystem, plus auxiliary host-state concerns like filesystem
paths and external service IDs).

### Class definition pattern

```python
from dataclasses import dataclass, field
from sollertia_experiment.cross_system import SystemConfiguration

@dataclass
class <System>SystemConfiguration(SystemConfiguration):
    """Defines the hardware and software asset configuration for the <System> data acquisition system."""

    name: str = "<system_name>"
    """The descriptive name of the data acquisition system."""

    filesystem: <System>FileSystem = field(default_factory=<System>FileSystem)
    """Stores the filesystem configuration."""

    cameras: <System>Cameras = field(default_factory=<System>Cameras)
    """Stores the video cameras configuration."""

    microcontrollers: <System>MicroControllers = field(default_factory=<System>MicroControllers)
    """Stores the microcontrollers configuration."""

    # Additional per-subsystem sections as needed: external_assets, sheets, etc.
```

**Rules:**
- The class is a plain `@dataclass` (NOT `slots=True`). `YamlConfig` serializes through `fields()`
  and `getattr`, so slots would work, but the base class defines no `__slots__` of its own and the
  top-level configuration is a single long-lived instance, so slots buy nothing here.
- Every nested section uses `field(default_factory=<Section>)` so each section's defaults apply when
  the field is absent from the YAML.
- Field docstrings (triple-quoted strings on the line after each field) describe the section's
  purpose. They document the dataclass source only, as `YamlConfig` writes no comments into the
  generated YAML.
- The `name` field is a free-form human-readable system label. It SHOULD be the short system token
  rather than the dataclass name, because the token is what the operator reads in the YAML.

### Optional `__post_init__` for normalization and validation

Use `__post_init__` to normalize on load whenever a field's natural YAML shape differs from the shape
the binding class consumes. A calibration lookup table is the recurring case, because a mapping reads
well in YAML while the wrapper wants a tuple of pairs:

```python
def __post_init__(self) -> None:
    """Normalizes the calibration table to a tuple representation and validates its shape."""
    table = self.<section>.<calibration_field>
    if not isinstance(table, tuple):
        self.<section>.<calibration_field> = tuple(table.items())
    # ... shape validation, raising through console.error ...
```

**Rules:**
- Normalization runs one way only, from the YAML shape to the in-memory shape. The `save()` override
  below performs the reverse conversion.
- Validation that catches user error belongs here, meaning wrong field types and missing required
  relationships between fields. Use `console.error(message=..., error=TypeError)` from
  `ataraxis_base_utilities`, so the failure mode matches the rest of the platform.
- Do NOT let `__post_init__` reach an MCP tool, the filesystem, or any other non-local resource. It
  runs on every YAML load and must stay fast.

### Optional `save()` override for YAML roundtrip hooks

Override `save()` whenever the YAML representation should differ from the in-memory representation:

```python
def save(self, path: Path) -> None:
    """Saves the instance to disk as a .yaml file, restoring the in-memory shape afterwards."""
    original_value = self.<section>.<calibration_field>
    try:
        if isinstance(original_value, tuple):
            self.<section>.<calibration_field> = dict(original_value)
        self.to_yaml(file_path=path)
    finally:
        self.<section>.<calibration_field> = original_value
```

**Rules:**
- Always restore the original in-memory state in a `finally` block. Otherwise an in-flight save
  followed by another access to the dataclass instance returns the wrong type.
- This pattern is earned only when the YAML shape differs from the in-memory shape, which most fields
  never need.

### Configuration-file lifecycle (shared cross-system registry)

Every system shares the create, resolve, and load steps of the configuration-file lifecycle. Those
steps live in `cross_system/system_configuration.py`, driven by a registry that maps each
`AcquisitionSystems` member to its `SystemConfiguration` subclass. The experiment package owns this
registry because the `SystemConfiguration` classes are defined there, while `sollertia-shared-assets`
owns the keyed-by-system registries for experiment configuration, hardware state, and raw data.

A host-machine belongs to exactly one acquisition system at a time, so its working directory holds
exactly one `<system>_system_configuration.yaml`. The shared discovery reads whichever single file is
present, so the getters resolve the local system from disk and take no `system` argument.

The shared module exposes:

```python
class SystemConfiguration(YamlConfig):
    """Base class for every system's configuration, supplying the registry type and a default save()."""

def register_system_configuration(system, configuration_class: type[SystemConfiguration]) -> None: ...
def create_system_configuration_file(system: AcquisitionSystems | str) -> None: ...  # any registered system
def get_system_configuration_path() -> Path: ...                                     # errors unless one match
def get_system_configuration_data() -> SystemConfiguration: ...                       # class resolved via the registry
```

Each system's package contributes the two things below. Path resolution, file discovery, and creation
all come from the shared helpers:

1. **Register** its `SystemConfiguration` subclass at import time:
   `register_system_configuration(AcquisitionSystems.<SYSTEM>, <System>SystemConfiguration)`.
2. **Expose a thin typed wrapper** that narrows the return type and validates the machine's system:

```python
def get_system_configuration() -> <System>SystemConfiguration:
    data = get_system_configuration_data()
    if not isinstance(data, <System>SystemConfiguration):
        console.error(...)   # this PC belongs to a different acquisition system
    return data
```

It may also expose a no-arg `create_system_configuration_file()` that defaults to its own system, and
re-export the shared `get_system_configuration_path`, for CLI/MCP convenience.

**Rules:**
- `register_system_configuration` is the only system-specific wiring the file lifecycle requires, and
  everything else is shared (`cross_system/system_configuration.py`).
- The shared `create_system_configuration_file` writes a **default-constructed** instance of the
  registered class first, and only then unlinks every other `*_system_configuration.yaml` in the same
  directory. That order means a failed write leaves the host with its previous system identity rather
  than none (`cross_system/system_configuration.py`).
- The just-written file is identified by inode through `existing.samefile(configuration_path)` rather
  than by name, because a case-insensitive filesystem keeps a differently-cased directory entry for it
  and a name comparison would delete the file the call just created. That guard sits in the cleanup
  loop of `create_system_configuration_file` (`cross_system/system_configuration.py`).
- The canonical path is `<working_directory>/configuration/<system>_system_configuration.yaml`, where
  `get_working_directory()` comes from `sollertia_shared_assets` and `_system_configuration_filename`
  derives the filename from the enum value as `f"{system}_system_configuration.yaml"`
  (`cross_system/system_configuration.py`).
- `get_system_configuration_path` globs that directory and raises `FileNotFoundError` unless exactly
  one file matches (`cross_system/system_configuration.py`).
- The per-system typed wrapper asserts the concrete `<System>SystemConfiguration` type, and the
  cross-system getters return the shared `SystemConfiguration` base.
- `SystemConfiguration`'s default `save()` calls `to_yaml`. A system overrides it when the on-disk YAML
  shape must differ from the in-memory shape, following the `__post_init__` and `save()` pattern above.

---

## Layer 2a: Per-subsystem configuration dataclasses

For each hardware subsystem (cameras, microcontrollers, motors, external assets), the system configuration
embeds a per-subsystem configuration dataclass. Each dataclass captures the **parameters** of the
subsystem's hardware that the binding class uses to instantiate per-device wrappers (physical
calibration values for some fields, plain operating settings for others).

### Class definition pattern

```python
from dataclasses import dataclass

@dataclass(slots=True)
class <System><Subsystem>:
    """Stores the <subsystem> configuration of the <System> data acquisition system."""

    <device>_<parameter>_<unit>: <type> = <default>
    """One-line description of what the field parameterizes, with units explicit where applicable."""
```

**Rules:**
- Use `@dataclass(slots=True)`. Slots reduce memory and catch attribute typos.
- Class name follows the `<System><Subsystem>` convention, so a system's camera section, board section,
  and external-asset section all carry the same system prefix.
- Each field has an explicit default, because the YAML loader falls back to defaults when a field is
  absent.
- Each field has a triple-quoted docstring immediately after it. It documents the field in source,
  and the generated YAML carries values only.

### Field naming convention

Field names encode three pieces of information separated by underscores:

```text
<device-or-module>_<parameter>_<unit>
```

| Component            | Examples                                       | Notes                                     |
|----------------------|------------------------------------------------|-------------------------------------------|
| `<device-or-module>` | a camera role, a sensor role, an actuator role | The device's role, not the hardware brand |
| `<parameter>`        | `index`, `threshold`, `delta_threshold`, `ppr` | The semantic name of the parameter        |
| `<unit>`             | `adc`, `us`, `ms`, `cm`, `g_cm`, `pulse`       | Included when the unit is non-obvious     |

**Examples:**
- `<role>_camera_index: int` carries no unit, because an index is an integer position.
- `<sensor>_threshold_adc: int` makes the ADC units explicit, because a threshold could be in volts.
- `<device>_polling_delay_us: int` makes the microseconds explicit.
- `<part>_diameter_cm: float` makes the centimeters explicit.

**Antipattern:** ambiguous parameter names. A bare `<sensor>_threshold: int` leaves the reader guessing
between ADC units, millivolts, and a count. Always add the unit suffix when the unit is non-obvious.

### Field type conventions

| Type                      | When to use                                                                      |
|---------------------------|----------------------------------------------------------------------------------|
| `int`                     | Counts, ADC units, frame rates, encoder pulse counts, durations in microseconds  |
| `float`                   | Physical measurements, conversion ratios, calibration coefficients               |
| `str`                     | Serial ports, hostnames, and a path consumed as a string rather than as a `Path` |
| `bool`                    | Behavior flags, such as a per-direction reporting switch                         |
| `Path`                    | Filesystem paths consumed directly as `Path` objects                             |
| `tuple[tuple[T, T], ...]` | Lookup tables, such as a calibration curve of duration against yield             |
| `<EnumType>`              | Enum values for encoder presets, pixel formats, and similar closed vocabularies  |

**Antipattern:** Using `Any` or untyped fields. Every field has an explicit, narrow type.

### Defaults convention

Defaults represent **factory defaults that work for the most common deployment**, usually the values
that produced a working system on the reference rig. They are calibrated starting points the user
overrides per host rather than magic numbers.

- A numeric default SHOULD be the value measured on the reference rig, so a fresh YAML describes a
  system that already ran.
- `Path()`, the empty path, is the default for any filesystem field. It reads as not configured, and the
  deployment sets it to enable the feature that consumes it. The on-demand mount report marks an unset
  long-term-storage root as not configured and reports it as ok. An optional read-only path, such as a
  stored camera configuration or an external tool's project path, is reported the same way. An unset
  path reads as not configured with an ok status, and a set path is checked for existence and
  readability rather than writability.
- A serial-port default SHOULD be a representative USB device path for the reference platform, such as
  a `/dev/tty*` entry on Linux or a `COMx` entry on Windows. The value is OS-specific and every
  deployment is expected to override it.

---

## Layer 2b: Per-subsystem binding classes

For each hardware subsystem, the acquisition system has one binding class that composes the subsystem's
per-device wrappers and orchestrates their lifecycle.

### Class definition pattern

```python
class <System><Subsystem>Bindings:
    """Interfaces with the <subsystem> devices used in the <system> data acquisition system."""

    def __init__(
        self,
        data_logger: DataLogger,
        <subsystem>_configuration: <System><Subsystem>,
        # optional: output_directory, previous_state, etc.
    ) -> None:
        # 1. Initialize the _started lifecycle flag
        self._started: bool = False

        # 2. Cache the configuration (sometimes needed for runtime parameter updates)
        self._configuration: <System><Subsystem> = <subsystem>_configuration

        # 3. Instantiate per-device wrappers using configuration fields
        self.<device_a>: <DeviceA>Interface = <DeviceA>Interface(
            <parameter>=<subsystem>_configuration.<field>,
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
        """Brings the subsystem online (the exact bring-up sequence is subsystem-type-specific)."""
        if self._started:
            return
        # Marks the subsystem started before the first bring-up step, so a failure partway through
        # still routes through stop(). Each low-level controller's own stop() self-guards.
        self._started = True
        # Bring-up sequence varies by subsystem type. See references/subsystem-types.md.

    def stop(self) -> None:
        """Stops all communication processes and releases all reserved resources."""
        if not self._started:
            return
        # Stop each underlying controller, isolating every step through run_shutdown_step. Stopping a
        # controller also resets the wrapped hardware.
        # Clears the flag only after every step, so a failure leaves the instance stoppable on retry.
        self._started = False
```

The skeleton above shows a microcontroller-style binding. It wraps `module_interfaces` and uses the
`_started`, `start()`, and `stop()` lifecycle. Camera and third-party-SDK subsystems share the
constructor and ownership rules below and differ in their method surface, documented in
`subsystem-types.md`.

**Rules:**
- **Constructor takes** `data_logger` first, when the subsystem logs to DataLogger, then the
  per-subsystem configuration dataclass, then any optional supplementary inputs such as an output
  directory or a previous-session state snapshot. Order matters, because `data_logger` is the
  most-shared dependency. An SDK-connection subsystem such as the Zaber motors may omit `data_logger`.
- **Per-device wrappers are public attributes.** A wrapper the lifecycle orchestrator may access
  directly, to issue a command at runtime, is public. An internal-only wrapper takes a leading
  underscore.
- **Underlying low-level controllers are private.** The binding class owns their lifecycle, and
  consumers reach the hardware through the public per-device wrappers.
- **`__del__` calls `stop()`** to guarantee cleanup on garbage collection. Always call `stop()`
  explicitly and treat `__del__` as a safety net only.
- **`_started` idempotency.** `start()` and `stop()` are no-ops when already in the target state,
  because the lifecycle orchestrator may double-call them during error recovery.
- **Flag ordering.** `stop()` clears the flag after its last step, so a failed tear-down stays
  retryable. A flag guarding a bring-up that walks several devices is raised before the first step, so
  a partial bring-up still tears down. A per-device flag guarding a single device whose own tear-down
  self-guards is raised after that device's bring-up returns.
- **Teardown isolation.** Each step of a multi-device tear-down runs inside `run_shutdown_step`, which
  catches the failure and echoes an ERROR so later steps still run
  (`cross_system/shutdown_tools.py`). An SDK-connection subsystem isolates inside its connection class
  instead, so its binding class calls `disconnect()` bare (`cross_system/zaber_bindings.py`).

### Bring-up sequence (subsystem-type-specific)

The bring-up sequence and method surface are specific to each subsystem type. The full per-type
sequences are documented in `subsystem-types.md`:

- **Microcontroller subsystems**: start each controller → `initialize_local_assets()` on every
  `SharedMemoryArray`-backed wrapper → push runtime parameters via `set_parameters()`.
- **Camera subsystems**: a `start_<role>_camera()` (acquisition) → `save_<role>_camera_frames()`
  (saving) split.
- **Third-party-SDK subsystems**: connect in `__init__`, then expose `disconnect` plus position
  methods for their lifecycle.

The one invariant across types: if any bring-up step fails, the subsystem's runtime state is unsafe
and bring-up MUST raise rather than silently leaving the subsystem half-started.

### Lifecycle method conventions

| Method                       | When                                                                    |
|------------------------------|-------------------------------------------------------------------------|
| `start()`                    | Microcontroller type. Idempotent. Brings every managed controller online. |
| `stop()`                     | Microcontroller and camera types. Idempotent. Tears the subsystem down. An SDK-connection subsystem uses `disconnect()` instead. |
| `start_<role>_camera()`      | Camera type. When per-device bring-up granularity is useful.            |
| `save_<role>_camera_frames()`| Camera type. When saving is distinct from acquiring, as it is for cameras. |
| `restore_position()`         | Optional. When the subsystem carries cross-session state, as motors do. |

Subsystems that wrap a third-party SDK connection (e.g., the Zaber motor subsystem) open the SDK
connection inside `__init__` and expose `disconnect` plus position methods (`restore_position`,
`park_motors` / `unpark_motors`) for their lifecycle.

---

## Layer 3: Lifecycle orchestrator

The lifecycle orchestrator is one class (typically named `<System>System` or `<System>VRSystem`)
per acquisition system that composes the Layer-2 binding classes and owns the master start/stop.

This skill documents the *contract* the orchestrator must honor. That system's runtime skill documents
its specific state machine and runtime modes.

### Construction and bring-up order

**Construction (instantiation)** is fixed by one hard build dependency. The DataLogger must exist
before any controller:

```text
DataLogger(...)              ── instantiated first (owns the output directory each controller registers against)
    ↓
<Boards>Bindings(...)
    ↓
<Cameras>Bindings(...)
    ↓
<other subsystem bindings>... (motors, custom devices)
    ↓
VRTaskDriver(...)            ── after the hardware bindings, only for the session types that run the task
```

Each `MicroControllerInterface.__init__` writes a manifest entry into the DataLogger's output directory, a directory
created by `DataLogger.__init__`, so the DataLogger *instance* MUST precede every controller. The DataLogger's logger
process starts later, during the orchestrator's bring-up, after every controller is already constructed. The remaining
instantiation order follows the subsystem list, and the *bring-up* staging below governs runtime behavior.

**Bring-up (start)** is staged by the runtime state machine and governed by two principles:

1. **Dependency ordering. Start an asset only after the assets it depends on are running.**
   *Worked example:* a display-driving board is started before the interactive VR setup.
   See `mesoscope:mesoscope-vr-runtime`.

2. **Keep as few assets running as possible until interactive setup is done.** An interactive setup
   phase is lengthy and operator-driven, so starting only what each step strictly needs keeps the main
   PC's resources free.
   *Worked example:* resource-heavy assets stay off until setup completes.
   See `mesoscope:mesoscope-vr-runtime`.

Which asset starts in which runtime state is the runtime skill's domain. See `/acquisition-system-runtime` and the
per-system runtime skill. The design contract here is only that the orchestrator honors construction and runtime
dependencies, and defers resource-heavy assets until interactive setup needs them.

### Shutdown order

Shutdown applies the **same dependency principle in reverse**. Stop each asset before the assets that
record from or depend on it, and stop the DataLogger last.

- **A producer is stopped before its recorder.** Otherwise the recorder stops while the source is
  still emitting. *Worked example:* an instrument is stopped before the board that timestamps its
  pulses. See `mesoscope:mesoscope-vr-runtime`.
- **The DataLogger is stopped last.** Every binding class's `stop()` may write final messages to it,
  and it records the data streams from all sources, so it MUST outlive every consumer.
- **Every teardown step is isolated.** `run_shutdown_step` catches a failing step and echoes an ERROR
  so the remaining steps still run (`cross_system/shutdown_tools.py`).

After shutdown, the downstream preprocessing pipeline calls `assemble_log_archives()` from
`ataraxis_data_structures` to consolidate the per-source log entries into the final session output.
That call is a separate preprocessing step rather than part of the orchestrator's `stop()`
(`assemble_session_logs` in `cross_system/data_preprocessing.py`).

### Keepalive

AXCI handles microcontroller keepalive. A `MicroControllerInterface` constructed with a non-zero
`keepalive_interval` sends keepalive messages to its controller at that interval, and raises a
`RuntimeError` when the controller misses its deadline. The firmware Kernel reports the miss as the
`kKeepAliveTimeout` status code, value 10, and performs an emergency reset that returns all managed
hardware to its default state.

The orchestrator and the binding classes are responsible for **configuration** only. Each
`MicroControllerInterface` receives its `keepalive_interval` at construction, read from the
configuration section's keepalive-interval field, where a value of `0` disables keepalive. AXCI owns
detection and aborting. See `/microcontroller-interface` for the per-interface keepalive surface.

### Cross-subsystem synchronization

The orchestrator wires every cross-subsystem signal, meaning every case where one subsystem's state
feeds another subsystem's startup. Two examples:

- A subsystem emits TTL into a microcontroller input, such as a camera or an instrument frame-clock.
  The orchestrator brings the receiving microcontroller's monitoring online before the emitter starts,
  so the receiver timestamps the pulses from the first one.
- The Unity initial state depends on a motor position. The orchestrator reads that position from the
  motor binding class and passes it to the Unity MQTT setup.

Cross-subsystem signaling lives in the orchestrator rather than in an individual binding class. Each
binding class stays oblivious to the other subsystems, and the orchestrator is the only place that
knows the full hardware composition.

---

## Cross-layer contracts

Three contracts must hold for the architecture to function:

### Contract 1: Configuration field naming agreement

Each per-subsystem configuration dataclass field that feeds a wrapper constructor MUST match the wrapper's
keyword-argument name conceptually (allowing for unit-suffix differences):

| Dataclass field pattern       | Wrapper constructor kwarg | Note                                                 |
|-------------------------------|---------------------------|------------------------------------------------------|
| `<device>_<parameter>_<unit>` | `<device>_<parameter>`    | Unit suffix dropped at the API boundary              |
| `<role>_<parameter>`          | `<device>_<parameter>`    | A rename to the wrapper's own device noun is allowed |
| `<parameter>`                 | `<parameter>`             | Exact match, the default                             |

The wrapper constructors these fields feed live in `cross_system/module_interfaces.py`, so their
argument names are platform material rather than per-system material. When a wrapper changes its
constructor signature, the corresponding dataclass field's name SHOULD change in the same change set
to maintain conceptual agreement. Drift here is allowed and causes confusion during debugging.

### Contract 2: Schema versioning

Any change to a configuration dataclass field (add, remove, rename, type-change) is a **schema change**
and MUST be paired with a version bump on the package that owns the dataclass. Older YAML files
written against the old schema MUST fail to load (with a clear error) rather than silently producing
a misconfigured system.

`YamlConfig`'s default behavior is permissive. Extra fields in the YAML are ignored, and missing fields
fall back to defaults. Both are dangerous for configuration, so the pattern is:

- Add a field: bump the consumer library's minor version. Older YAML files load with the new field
  at its default, which is acceptable when the default is safe and needs an explicit error otherwise.
- Remove a field: bump the consumer library's major version. Older YAML files load with the
  removed field silently ignored (acceptable since it has no effect).
- Rename a field: bump the consumer library's major version. Treat as remove-and-add. Add a
  one-cycle deprecation step in `__post_init__` that detects the old name in the loaded dict and
  migrates it.
- Change a field's type or units: bump the consumer library's major version. Add validation in
  `__post_init__` that detects the old shape and raises a clear migration error.

### Contract 3: Lifecycle ordering

Every binding class assumes:
- The DataLogger instance exists before `__init__` is called.
- The constructor's `data_logger` argument is the DataLogger instance to log to.
- The DataLogger is started before `start()` is called.
- `start()` is called before any per-device wrapper method that issues commands or reads data.
- `stop()` is called before DataLogger is stopped.

The orchestrator MUST honor this. Deviating from the order produces difficult-to-debug failures
(missing manifest entries, dropped initial parameters, hung communication processes).
