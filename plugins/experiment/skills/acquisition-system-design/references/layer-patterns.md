# Layer patterns and cross-layer contracts

Detailed authoring patterns for the three layers of a Sollertia acquisition system, plus the
cross-layer contracts that hold the architecture together. Loaded on demand from
`acquisition-system-design`'s SKILL.md.

---

## Layer 1: System configuration

The top-level system configuration is a single `@dataclass` that inherits from
`ataraxis_data_structures.YamlConfig` and composes one nested dataclass per concern (per hardware
subsystem, plus auxiliary host-state concerns like filesystem paths and external service IDs).

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

### Configuration-file lifecycle (shared cross-system registry)

The configuration-file lifecycle — create, resolve, load — is shared across systems. It lives in a
cross-system module (`sollertia_experiment/cross_system/system_configuration.py`) driven by a registry
that maps each `AcquisitionSystems` member to its `SystemConfiguration` subclass. (The experiment
package owns this registry because the `SystemConfiguration` classes are defined there;
`sollertia-shared-assets` owns the keyed-by-system registries for experiment configuration, hardware
state, and raw data.)

A host-machine belongs to exactly one acquisition system at a time: its working directory holds exactly
one `<system>_system_configuration.yaml`. The shared discovery reads whichever single file is present,
so the getters resolve the local system from disk and take no `system` argument.

The shared module exposes:

```python
class SystemConfiguration(YamlConfig):
    """Base class for every system's configuration: the common registry type + a default save()."""

def register_system_configuration(system, configuration_class: type[SystemConfiguration]) -> None: ...
def create_system_configuration_file(system: AcquisitionSystems | str) -> None: ...  # registry-driven; any registered system
def get_system_configuration_path() -> Path: ...                                     # the single on-disk file; errors if != 1
def get_system_configuration_data() -> SystemConfiguration: ...                       # loads it, class resolved via the registry
```

Each system's package contributes two things; the path resolution, file discovery, and creation come
from the shared helpers:

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
- The shared `create_system_configuration_file` removes any pre-existing `*_system_configuration.yaml`
  so exactly one remains, then writes the registered class's defaults. The canonical path is
  `<working_directory>/configuration/<system>_system_configuration.yaml` (`get_working_directory()`
  from `sollertia_shared_assets`); the filename is derived from the system name, not stored separately.
- The per-system typed wrapper asserts the concrete `<System>SystemConfiguration` type; the
  cross-system getters return the shared `SystemConfiguration` base.
- `SystemConfiguration`'s default `save()` calls `to_yaml`; a system overrides it when the on-disk YAML
  shape must differ from the in-memory shape (the `__post_init__` / `save()` pattern above).

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
- Use `@dataclass(slots=True)`. The per-subsystem dataclasses don't inherit from `YamlConfig`, so slots
  are safe and reduce memory.
- Class name follows the `<System><Subsystem>` convention: `MesoscopeCameras`, `MesoscopeMicroControllers`,
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
| `<parameter>`        | `index`, `threshold`, `delta_threshold`, `polling_delay`, `ppr` | The semantic name of the parameter                                 |
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
        # Bring-up sequence varies by subsystem type — see references/subsystem-types.md.
        # (Microcontroller subsystems: start controllers → initialize_local_assets() → set_parameters().)
        self._started = True

    def stop(self) -> None:
        """Stops all communication processes and releases all reserved resources."""
        if not self._started:
            return
        self._started = False
        # Stop each underlying controller (this also resets the wrapped hardware)
```

The skeleton above shows a microcontroller-style binding (it wraps `module_interfaces` and uses the
`_started` / `start()` / `stop()` lifecycle); camera and third-party-SDK subsystems share the
constructor and ownership rules below but differ in their method surface (see
[subsystem-types.md](subsystem-types.md)).

**Rules:**
- **Constructor takes** `data_logger` first (when the subsystem logs to DataLogger), then the per-subsystem
  configuration dataclass, then any optional supplementary inputs (output directory, previous-session
  state snapshot, etc.). Order matters: `data_logger` is the most-shared dependency. SDK-connection
  subsystems (e.g., Zaber motors) may omit `data_logger` entirely.
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

### Bring-up sequence (subsystem-type-specific)

The bring-up sequence and method surface are specific to each subsystem type. The full per-type
sequences are documented in [subsystem-types.md](subsystem-types.md):

- **Microcontroller subsystems**: start each controller → `initialize_local_assets()` on every
  `SharedMemoryArray`-backed wrapper → push runtime parameters via `set_parameters()`.
- **Camera subsystems**: a `start_<role>_camera()` (acquisition) → `save_<role>_camera_frames()`
  (saving) split.
- **Third-party-SDK subsystems**: connect in `__init__`; expose `connect` / `disconnect` plus position
  methods for their lifecycle.

The one invariant across types: if any bring-up step fails, the subsystem's runtime state is unsafe
and bring-up MUST raise rather than silently leaving the subsystem half-started.

### Lifecycle method conventions

| Method                       | When                                                                                             |
|------------------------------|--------------------------------------------------------------------------------------------------|
| `start()`                    | Mandatory. Idempotent. Brings the subsystem online.                                              |
| `stop()`                     | Mandatory. Idempotent. Tears the subsystem down.                                                 |
| `start_<sub_device>()`       | Optional. When per-device lifecycle granularity is useful (e.g., starting one camera at a time). |
| `save_<sub_device>_frames()` | Optional. When the subsystem has a "saving" distinct from "acquiring" (cameras).                 |
| `restore_position()`         | Optional. When the subsystem has cross-session state (motors).                                   |
| `is_started` property        | Optional. Read-only view of `_started`.                                                          |

Subsystems that wrap a third-party SDK connection (e.g., the Zaber motor subsystem) expose `connect` (in
`__init__`) / `disconnect` plus position methods (`restore_position`, `park_motors` / `unpark_motors`)
for their lifecycle.

---

## Layer 3: Lifecycle orchestrator

The lifecycle orchestrator is one class (typically named `<System>System` or `<System>VRSystem`)
per acquisition system that composes the Layer-2 binding classes and owns the master start/stop.

This skill documents the *contract* the orchestrator must honor; the specific state machine and
runtime modes for any given system are documented in that system's behavior/runtime skill.

### Construction and bring-up order

**Construction (instantiation)** is fixed by one hard build dependency — the DataLogger must exist
before any controller:

```text
DataLogger(...)              ── instantiated first (owns the output directory each controller registers against)
    ↓
MicroControllerInterfaces(...)
    ↓
VideoSystems(...)
    ↓
<other subsystem bindings>... (motors, custom devices)
    ↓
VRTaskDriver(...)            ── last, and only when the system drives Unity (experiment sessions)
```

Each `MicroControllerInterface.__init__` registers a manifest entry in the DataLogger's output
directory, so the DataLogger (and its started communication process) MUST precede every controller.
The remaining instantiation order follows the subsystem list; runtime behavior is governed by the
*bring-up* staging below.

**Bring-up (start)** is staged by the runtime state
machine and governed by two principles:

1. **Dependency ordering — start an asset only after the assets it depends on are running.**
   *Concrete example (Mesoscope-VR):* the microcontrollers are started before the Unity VR setup runs,
   because Unity's interactive setup requires the VR screens to be powered on, and the screens are
   driven by an ACTOR microcontroller module. Running Unity setup against dark screens is unhelpful.

2. **Keep as few assets running as possible until interactive setup is done.** The interactive setup
   phase, such as Unity environment setup or mesoscope alignment — are lengthy and operator-driven, so
   starting only what each step strictly needs keeps the main PC's resources free.
   *Concrete example (Mesoscope-VR):* the behavior cameras and any microcontroller sensors not needed
   for setup stay off until setup completes; only the screen-driving microcontroller is up for Unity
   setup. This frees CPU cores so an operator can preprocess a previous session while setting up the
   next.

The precise staging (which asset starts in which runtime state) is the runtime skill's domain — see
`experiment:acquisition-system-runtime` and the per-system runtime skill. The design contract here is
only that the orchestrator (a) honors construction and runtime dependencies and (b) defers
resource-heavy assets until interactive setup needs them.

### Shutdown order

Shutdown applies the **same dependency principle in reverse**: stop each asset before the assets that
record from or depend on it, and stop the DataLogger last.

- **A producer is stopped before its recorder.** *Concrete example (Mesoscope-VR):* the microcontrollers
  record the mesoscope's frame-clock TTL pulses, so the mesoscope is shut down before the
  microcontrollers — otherwise the recorder would stop while the source is still emitting.
- **The DataLogger is stopped last.** Every binding class's `stop()` may write final messages to it,
  and it records the data streams from all sources, so it MUST outlive every consumer.

After shutdown, the downstream preprocessing pipeline calls `assemble_log_archives()` (from
`ataraxis_data_structures`) to consolidate the per-source log entries into the final session output;
this runs as a separate preprocessing step, not inside the orchestrator's `stop()`.

### Keepalive

AXCI handles microcontroller keepalive. Once a `MicroControllerInterface` is constructed with a
non-zero `keepalive_interval`, it sends keepalive messages to its controller at that interval and
raises a `RuntimeError` if the controller misses its deadline (the `KEEPALIVE_TIMEOUT` Kernel event,
code 10, after the microcontroller performs its emergency reset).

The orchestrator/binding-class responsibility is **configuration**: pass each `MicroControllerInterface`
its `keepalive_interval` (from the configuration's `keepalive_interval_ms` field; `0` disables
keepalive) at construction. AXCI owns detection and aborting. See
`experiment:microcontroller-interface` for the per-interface keepalive surface.

### Cross-subsystem synchronization

When two subsystems must agree at runtime (e.g., one subsystem's state feeds another's startup), the
orchestrator wires the cross-subsystem signals. Examples:

- A subsystem that emits TTL into a microcontroller input (a camera or instrument frame-clock, say) →
  the orchestrator brings the microcontroller's monitoring online before the emitter starts, so the
  microcontroller (the TTL *receiver*) timestamps the pulses from the first one.
- Motor position needed to compute Unity initial state → orchestrator reads the position from the
  motor binding class and passes it to the Unity MQTT setup.

Cross-subsystem signaling lives in the orchestrator, NOT in the individual binding classes. Each binding
class stays oblivious to the existence of other subsystems; the orchestrator is the only place that knows
the full hardware composition.

---

## Cross-layer contracts

Three contracts must hold for the architecture to function:

### Contract 1: Configuration field naming agreement

Each per-subsystem configuration dataclass field that feeds a wrapper constructor MUST match the wrapper's
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

Any change to a configuration dataclass field (add, remove, rename, type-change) is a **schema change**
and MUST be paired with a version bump on the package that owns the dataclass. Older YAML files
written against the old schema MUST fail to load (with a clear error) rather than silently producing
a misconfigured system.

`YamlConfig`'s default behavior is permissive — extra fields in the YAML are ignored, missing fields
fall back to defaults. Both are dangerous for configuration. The pattern is:

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
