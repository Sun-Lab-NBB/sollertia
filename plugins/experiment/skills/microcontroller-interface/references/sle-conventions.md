# sle Python wrapper conventions

These conventions extend or deviate from `communication:microcontroller-interface` (ataraxis marketplace) and apply to
every `ModuleInterface` subclass in `src/sollertia_experiment/cross_system/module_interfaces.py`. See
[`../SKILL.md`](../SKILL.md) for the registry, cross-side contract, and workflows that govern this layer.

---

## File location and naming

- **One shared file**: All `ModuleInterface` subclasses live in `cross_system/module_interfaces.py`, not in any
  system-specific subpackage. The file is system-agnostic, so adding a wrapper here makes it available to any current or
  future acquisition-system binding class.
- **Class naming**: a firmware module is wrapped by one *or more* interface classes, so the mapping is **not
  one-to-one**. A module with a single general-purpose wrapper names it `<FirmwareModuleName-without-Module>Interface`
  (`EncoderModule` → `EncoderInterface`, `LickModule` → `LickInterface`, `BrakeModule` → `BrakeInterface`,
  `TorqueModule` → `TorqueInterface`, `ScreenModule` → `ScreenInterface`). When a module is consumed in one or more
  specific application roles, each wrapper is named after its role instead: `ValveModule` is wrapped by
  `WaterValveInterface` (id 1) and `GasPuffValveInterface` (id 2), and `TTLModule` is wrapped by
  `MesoscopeFrameTTLInterface`.

---

## Constructor signature

The wrapper's `__init__` exposes **calibration and policy parameters as regular parameters that call sites pass by
keyword**, and fixes the contract identity inside the `super().__init__(...)` call. Constructors carry no `*` separator.
True keyword-only syntax is reserved for the binary state setters described under [Public-method
patterns](#public-method-patterns). What is fixed versus caller-supplied varies by field:

- **`module_type` is always hardcoded.** This is an architectural decision: a wrapper class is permanently bound to one
  firmware module type, and the caller never supplies it.
- **A firmware type is not one-to-one with a wrapper.** The same firmware type, reused for a different role, gets its
  own interface class that hardcodes the same `module_type` but exposes different calibration, semantics, and methods.
  `ValveModule` (type 5) is wrapped by both `WaterValveInterface` and `GasPuffValveInterface`, the same valve firmware,
  but a gas puff is not a water reward, so the operating principles, calibration model, and exposed surface differ.
- **`name`, `data_codes`, and `error_codes` are hardcoded per wrapper.** `data_codes` / `error_codes` are the subset of
  the firmware type's event codes that role actually handles, and can differ between two wrappers of the same type
  (`WaterValveInterface` routes `{51, 52, 53}`, `GasPuffValveInterface` routes `{51, 52}`).
- **`module_id` is hardcoded only because each current wrapper drives a single physical instance.** This is the most
  efficient form for the present deployment rather than a principle. A wrapper that needs to serve several instances of
  the same role (e.g., two water valves at ids 1 and 2) would expose `module_id` as a constructor argument instead,
  while the type would still be hardcoded.

```python
def __init__(
    self,
    encoder_ppr: int,
    wheel_diameter: float,
    polling_frequency: int,
) -> None:
    super().__init__(
        module_type=np.uint8(2),  # always hardcoded (architectural)
        module_id=np.uint8(1),    # hardcoded: a single encoder instance is deployed
        name="encoder",
        data_codes={np.uint8(51), np.uint8(52)},
        error_codes=None,
    )
    # ... calibration math, shared memory creation, command-code precomputation ...
```

Caller-facing constructor arguments are calibration values, polling frequencies, or domain-specific configuration (e.g.,
`valve_calibration_data` is the calibration tuple, not a raw parameter).

---

## Calibration math in `__init__`

Unit conversions and calibration-driven derived quantities are computed in `__init__` and cached as `np.float64`
instance attributes at full precision. Examples:

- `EncoderInterface`: `_cm_per_pulse = np.float64((pi * wheel_diameter) / ppr)` in `__init__`. `_unity_unit_per_pulse`
  is derived later in `set_unity_scale(cm_per_unity_unit)`, called at experiment start with the conversion value read
  from the active `TaskTemplate`, rather than in `__init__`.
- `TorqueInterface`: `_torque_per_adc_unit = np.float64(sensor_capacity) * np.float64(0.00981) / (max_v - baseline_v)`
- `BrakeInterface`: minimum/maximum strength in g·cm converted to N·cm using `0.00981`
- `WaterValveInterface`: `curve_fit` of the power-law model yields `_scale_coefficient` and `_nonlinearity_exponent`,
  both stored as `np.float64`

Rounding happens at the point of use, where a derived value becomes an integer command argument. `WaterValveInterface`
rounds the millisecond-to-microsecond tone conversion in `deliver_reward` and `simulate_reward`, and rounds the pulse
duration it returns from `get_duration_from_volume`.

When a module's calibration requires unit conversion, perform it **in the wrapper**, not in the binding-class layer.
Binding classes pass raw user-input values to the wrapper unchanged.

---

## Lifecycle: four methods, not three

The ataraxis base defines three abstract methods (`initialize_remote_assets`, `terminate_remote_assets`,
`process_received_data`). sle wrappers that use `SharedMemoryArray` add a fourth: **`initialize_local_assets()`**.

| Method                     | Process                  | Purpose                                                                                                |
|----------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| `__init__`                 | parent                   | Create `SharedMemoryArray.create_array(..., exists_ok=True)`, compute calibration, cache command codes |
| `initialize_local_assets`  | parent                   | Connect to the shared memory + call `enable_buffer_destruction()` so the parent process owns cleanup   |
| `initialize_remote_assets` | communication subprocess | Connect to the shared memory, initialize non-picklable assets (`PrecisionTimer`)                       |
| `process_received_data`    | communication subprocess | Process incoming `ModuleData` / `ModuleState` messages, update shared memory                           |
| `terminate_remote_assets`  | communication subprocess | Disconnect from shared memory                                                                          |
| `__del__`                  | parent                   | `disconnect()` then `destroy()` the shared memory                                                      |

The `initialize_local_assets()` method is sle-specific. Binding-class code calls it explicitly, immediately after each
`MicroControllerInterface.start()` has spawned its communication subprocess, so the parent process connects to the
shared-memory buffers while the subprocesses are already running. The ataraxis `ModuleInterface` base declares only
`initialize_remote_assets`, `terminate_remote_assets`, and `process_received_data`, so there is no inherited default to
fall back on.

Wrappers without shared memory (e.g., `BrakeInterface`, `TorqueInterface`, `ScreenInterface`) omit
`initialize_local_assets()` and leave both remote-asset methods as `return` no-ops. Binding classes call
`initialize_local_assets()` only on the wrappers that define it.

---

## `SharedMemoryArray` naming

Use a deterministic name derived from the module type and id:

```python
self._foo_tracker: SharedMemoryArray = SharedMemoryArray.create_array(
    name=f"{self._module_type}_{self._module_id}_foo_tracker",
    prototype=np.zeros(shape=N, dtype=np.float64 | np.uint64 | np.uint32),
    exists_ok=True,
)
```

The `exists_ok=True` flag is required because the array may already exist from a prior run that did not clean up. The
parent process re-claims and destroys it on shutdown via `enable_buffer_destruction()`.

---

## Cached command codes

Cache each command code as an `np.uint8` instance attribute in `__init__` rather than re-creating `np.uint8(1)` on every
call site:

```python
self._check_state: np.uint8 = np.uint8(1)
self._reset_encoder: np.uint8 = np.uint8(2)
```

This avoids repeated numpy scalar allocation on hot paths and makes the available command surface visible at one glance
in the constructor.

---

## Module-level numpy constants

`module_interfaces.py` defines shared numpy constants at the top of the file for use in hot paths:

```python
_ZERO_UINT64  = np.uint64(0)
_ZERO_FLOAT64 = np.float64(0.0)
_ZERO_UINT32  = np.uint32(0)
_FALSE: np.bool_ = np.bool_(0)
```

Reuse these in `send_command` / `send_parameters` calls instead of creating fresh `np.uint32(0)` instances each time.

---

## Public-method patterns

**Typed `set_parameters` wrapper** (mandatory, not optional): expose every parameter-struct field as a named keyword
argument matching the firmware field name. The base ataraxis skill calls this an "optional" maintainability pattern, and
sle treats it as the standard surface for any wrapper that sets runtime parameters.

```python
def set_parameters(
    self,
    signal_threshold: np.uint16,
    delta_threshold: np.uint16,
    average_pool_size: np.uint8,
) -> None:
    self.send_parameters(parameter_data=(signal_threshold, delta_threshold, average_pool_size))
```

**State-setter pattern**: command methods that toggle a binary state use keyword-only `state: bool`:

```python
def set_state(self, *, state: bool) -> None:
    if state == self._configured_state:
        return  # idempotent, no-op if already in target state
    self.send_command(...)
    self._configured_state = state
```

The idempotency guard (`if state == ...: return`) is part of the convention, because it prevents redundant serial
traffic when the binding class repeatedly asserts the same state.

**Cache-and-skip parameter updates**: when a command's parameter struct rarely changes between calls, cache the
last-sent value and only re-send parameters when the new value differs:

```python
if duration_ms != self._previous_pulse_duration:
    self._previous_pulse_duration = duration_ms
    self.send_parameters(...)
self.send_command(...)
```

Exercised by `BrakeInterface.send_pulse`, `WaterValveInterface.deliver_reward`, `GasPuffValveInterface.deliver_puff`.
Critical for reward-delivery hot paths where the same volume is delivered hundreds of times per session.

**Cap-and-warn for safety bounds**: when a user-supplied value can exceed a hardware safety limit, cap to the safe
maximum and emit a console warning rather than raising:

```python
if duration_ms > _MAXIMUM_VALVE_PULSE_DURATION_MS:
    console.echo(message=..., level=LogLevel.WARNING)
    duration_ms = _MAXIMUM_VALVE_PULSE_DURATION_MS
```

The `_MAXIMUM_VALVE_PULSE_DURATION_MS = 400` constant is set to keep valve pulses safely below the 500 ms keepalive
interval, because longer pulses would trigger the keepalive watchdog and abort the runtime. Any new safety bound MUST
cite the underlying constraint (e.g., keepalive interval, motor current limit) in a top-of-file constant comment.

---

## Property accessors

Expose derived calibration values and read-only state as `@property`-decorated accessors. Examples: `cm_per_pulse`,
`lick_count`, `delivered_volume`, `maximum_brake_strength`. Consumers (binding classes) read these to populate their own
state without touching wrapper internals.
