---
name: sollertia-microcontroller-interface
description: >-
  Registry of paired Module (sollertia-micro-controllers) and ModuleInterface
  (sollertia-experiment) classes available to Sollertia acquisition systems, plus the
  conventions on top of the ataraxis base templates and principles for adding modules or
  controller boards. Use when extending hardware support or modifying a paired Module + Interface.
user-invocable: true
---

# Sollertia microcontroller interface

Documents the Sollertia platform's paired microcontroller interface stack at the pre-binding-class level:

- **slmc** (`sollertia-micro-controllers`, C++ firmware) — `Module` subclasses that run on
  Arduino-compatible microcontroller boards. The current deployment uses Teensy 4.1 boards, but the
  firmware library is board-agnostic; this skill's conventions apply to any board slmc targets.
- **sle** (`sollertia-experiment`, Python) — `ModuleInterface` subclasses in
  `src/sollertia_experiment/shared_components/module_interfaces.py` that wrap each firmware module as a
  contract pair.

This skill is the canonical registry of paired modules currently available to any Sollertia acquisition
system and the source of truth for the slmc/sle conventions layered on top of the ataraxis base templates.
Binding-class composition (assembling `MicroControllerInterface` instances, calibration dataclasses,
system configurations) is **system-specific** and lives in per-system skills (currently
`experiment:mesoscope-vr` for hardware composition and `experiment:mesoscope-vr-runtime` for
runtime behavior), not here. The platform-general pattern those skills follow lives in
`experiment:acquisition-system-design`.

---

## Scope

**Covers:**
- Registry of available paired Modules + Interfaces with data surface, calibration knobs, command and
  event codes
- slmc firmware conventions beyond the ataraxis `Module` base template
- sle Python wrapper conventions beyond the ataraxis `ModuleInterface` base template
- Cross-side contract requirements (what must agree between firmware and Python wrapper)
- Principles for adding or removing modules on both sides
- Principles for allocating modules across controller boards
- Workflows for adding paired modules and adding new controller boards

**Does not cover** (delegated):
- Base `Module` / `ModuleInterface` API, `PACKED_STRUCT` mechanics, `SendData` patterns, event-code ranges,
  `MicroControllerInterface` lifecycle, MQTTCommunication, DataLogger topology, keepalive mechanics —
  see `ataraxis@microcontroller:firmware-module` and `ataraxis@communication:microcontroller-interface`.
- Microcontroller discovery, manifest management via MCP tools — see `ataraxis@communication:microcontroller-setup`.
- C++ style, Python style, header guards conventions enforcement —
  see `automation:cpp-style`, `automation:python-style`.
- **Binding-class composition** (per-system `MicroControllerInterfaces`, calibration dataclasses,
  system configuration YAML, runtime orchestration) — covered by `experiment:acquisition-system-design`
  for the platform-general pattern and `experiment:mesoscope-vr` for the current Mesoscope-VR
  instance. This skill ends at the per-firmware-module Python wrapper; everything that composes
  those wrappers into a runnable acquisition system is owned downstream.
- Hardware discovery / post-flash verification — see `experiment:acquisition-system-setup`.

---

## Authoritative bases

Before reading the conventions or workflows in this skill, you MUST be familiar with the ataraxis base
templates this layer builds on:

| Concern                                          | Authority                                            |
|--------------------------------------------------|------------------------------------------------------|
| C++ `Module` API, parameter structs, `SendData`  | `ataraxis@microcontroller:firmware-module`           |
| Python `ModuleInterface` API, abstract methods   | `ataraxis@communication:microcontroller-interface`   |
| Wire protocol, event-code ranges, message types  | Either ataraxis skill (mirrored sections)            |
| Microcontroller discovery and verification       | `ataraxis@communication:microcontroller-setup`       |

The Sollertia layer **inherits all base mechanics and only documents deviations and expansions**. When a
section below cites a base behavior, treat the ataraxis skill as the source of truth.

---

## Hardware-surface registry

The current registry of paired Modules + Interfaces — type codes, parameter structs, event codes,
commands, calibration knobs, shared-memory layouts, and the public method surface of each wrapper —
lives in [`references/module-catalog.md`](references/module-catalog.md). That file is a state
snapshot of the currently-deployed pair set and MUST be updated whenever a module is added, removed,
or modified (see [Maintenance contract](#maintenance-contract-for-this-skill)).

### Type-code allocation rules

These rules are durable and govern how new codes are assigned — consult the catalog file for the
currently-used values.

- Module type codes are `uint8_t`. Value 0 is reserved by the runtime as the "no active command"
  sentinel and SHOULD NOT be used as a type code either.
- The `(module_type, module_id)` pair MUST be unique on a single controller board. Two firmware
  instances of the same `Module` subclass on the same board take different `module_id` values.
- When allocating a new type code, pick the next unused value from the catalog rather than recycling
  a freed one. Historical type codes may still appear in archived log data and reuse would conflate
  old and new modules during processing.
- Custom event codes use the 51-250 range per the ataraxis base; see
  `ataraxis@microcontroller:firmware-module` for the full range table.

---

## slmc firmware conventions (deviations from ataraxis base)

The following conventions extend or deviate from `ataraxis@microcontroller:firmware-module`. They apply
to every slmc `Module` subclass.

### Header conventions

- **Include guards**: `AXMC_<MODULE_NAME>_MODULE_H` (e.g., `AXMC_ENCODER_MODULE_H`). The base skill
  permits `#pragma once`; slmc uses traditional guards with the `AXMC_` prefix for consistency with the
  upstream ataraxis-micro-controller library.
- **File-header Doxygen**: Every header starts with `/** @file @brief ... */` and uses `@warning`,
  `@note`, `@tparam`, `@param` tags. `automation:cpp-style` is the authoritative reference for the
  format; this skill notes only that file headers are mandatory.
- **Header-only modules**: All `Module` subclasses live entirely in `.h` files. No `.cpp` files exist
  under `src/` for module classes; everything is template-instantiated from `main.cpp`. New modules
  MUST follow this pattern.
- **`final` on Module subclass**: Every `Module` subclass uses `class FooModule final : public Module`.
  Subclassing of an existing slmc Module is not supported.
- **Explicit defaulted destructor**: `~FooModule() override = default;` — included for completeness
  even when no destructor logic is needed.

### Template-parameterized pin and behavior configuration

Every slmc module is a class template with compile-time pin and polarity parameters:

```cpp
template <const uint8_t kPin, const bool kNormallyClosed, const bool kStartClosed = true>
class ValveModule final : public Module { ... };
```

- All pin parameters are `const uint8_t`.
- Polarity flags (`kNormallyEngaged`, `kNormallyClosed`, `kNormallyOff`) are `const bool`.
- Calibration constants the firmware needs at compile time (e.g., `kBaseline` on `TorqueModule`) are
  template parameters, not runtime parameters.
- Optional pins use **255** as the sentinel value, paired with `if constexpr (kPin != 255)` checks in
  the implementation (see `ValveModule::kTonePin = 255`).

### Mandatory `LED_BUILTIN` static_assert

Every pin template parameter MUST be paired with a `static_assert(kPin != LED_BUILTIN, ...)` at the top
of the class body, before `public:`. The base ataraxis skill recommends this; slmc treats it as
mandatory because the LED-builtin pin is reserved for runtime error indication and accidental reuse
silently destroys debuggability.

When a module has multiple pins, every pin gets its own static_assert with a specific error message
(see `EncoderModule` with pins A, B, X).

### `constexpr` polarity logic

For modules with `kNormallyOpen` / `kNormallyClosed` / `kNormallyEngaged` polarity flags, derive the
"active" and "inactive" signal levels as `static constexpr bool` members instead of branching at
runtime:

```cpp
static constexpr bool kEngage    = kNormallyEngaged ? LOW  : HIGH;
static constexpr bool kDisengage = kNormallyEngaged ? HIGH : LOW;
```

Then use `kEngage` / `kDisengage` directly in `digitalWriteFast()` calls. This pattern appears in
`BrakeModule`, `ValveModule`, and `ScreenModule`.

### Initial-state reporting from `SetupModule()`

Every `SetupModule()` MUST emit at least one `SendData(...)` call after configuring its pins to
inform the PC of the module's starting state. This is a slmc convention beyond the ataraxis base —
the host-PC wrappers and downstream log processing assume every module reports an initial state, so
omitting this leaves a wrapper's instantaneous-state tracker (e.g., `_valve_tracker[2]`) unsynchronized
with the firmware.

For input-side modules, the initial-state report uses a zero-magnitude payload (e.g., `kRotatedCW, 0`
on `EncoderModule`) to establish a baseline that subsequent delta-threshold logic can compare against.

### Stage-based commands and blocking exceptions

Multi-step commands (any command that involves a timed delay) MUST use the stage-based pattern:
`get_command_stage()` → `AdvanceCommandStage()` → `WaitForMicros()` → `CompleteCommand()`. The base
skill documents the mechanics; slmc uses this pattern for every output-pulse command
(`BrakeModule::SendPulse`, `ValveModule::Pulse`, `ValveModule::Tone`, `ScreenModule::Toggle`,
`TTLModule::SendPulse`).

**Blocking exceptions**: `ValveModule::Calibrate` and `EncoderModule::GetPPR` block the runtime
in-place via `delayMicroseconds()` / busy-wait loops. They are explicitly marked with `@warning`
Doxygen blocks and are intended only for offline calibration — never during an active acquisition
session. If a new module needs a long blocking command for calibration, follow the same `@warning`
convention.

### Performance primitives

- Prefer `digitalWriteFast` / `pinModeFast` / `digitalReadFast` over the standard `digitalWrite` /
  `pinMode` / `digitalRead` for all pin operations whose pin is known at compile time (every slmc
  module pin qualifies because pins are template parameters).
- The `Encoder` library member must be statically initialized at class scope (`Encoder _encoder =
  Encoder(kPinA, kPinB);`). Deferred initialization in `SetupModule()` crashes the runtime; this is a
  Paul Stoffregen library constraint, not an slmc choice.

### Multi-target `main.cpp` via preprocessor

slmc partitions its modules across multiple microcontroller boards using a single `main.cpp` with
preprocessor `#ifdef` blocks:

```cpp
#define ACTOR   // Selected at upload time. Exactly one target must be defined.

#ifdef ACTOR
    static constexpr uint8_t kControllerID = 101;
    BrakeModule<33, false, true> wheel_brake(3, 1, axmc_communication);
    // ... ACTOR-specific instances ...
    Module* modules[] = { &wheel_brake, /* ... */ };

#elif defined SENSOR
    static constexpr uint8_t kControllerID = 152;
    // ... SENSOR-specific instances ...

#elif defined ENCODER
    static constexpr uint8_t kControllerID = 203;
    // ... ENCODER-specific instances ...

#else
    static_assert(false, "Define one of the supported microcontroller targets.");
#endif
```

This is the slmc-general pattern for supporting multiple controller boards from one firmware codebase.
The specific targets `ACTOR`, `SENSOR`, `ENCODER` are the Mesoscope-VR instance of this pattern —
when adding a different acquisition system, define new target macros (e.g., `STIMULUS`, `RECORD`)
following the same structure. See [Controller board allocation principles](#controller-board-allocation-principles)
below for when to add a new target vs. extending an existing one.

The fallback `static_assert(false, ...)` block under `#else` MUST remain — compiling without selecting
a target should fail loudly.

The keepalive interval (`kKeepaliveInterval = 500` ms) and `analogReadResolution(12)` are set
globally for all targets in `main.cpp` and apply to every module regardless of board.

---

## sle Python wrapper conventions (deviations from ataraxis base)

The following conventions extend or deviate from `ataraxis@communication:microcontroller-interface`.
They apply to every `ModuleInterface` subclass in
`src/sollertia_experiment/shared_components/module_interfaces.py`.

### File location and naming

- **One shared file**: All `ModuleInterface` subclasses live in `shared_components/module_interfaces.py`,
  not in any system-specific subpackage. The file is system-agnostic — adding a wrapper here makes it
  available to any current or future acquisition-system binding class.
- **Class naming**: `<FirmwareModuleName-without-Module>Interface`. `EncoderModule` → `EncoderInterface`,
  `ValveModule` → `ValveInterface`. Specialized wrappers for additional application roles of a
  single firmware module follow a `<Role>Interface` pattern (`GasPuffValveInterface` for the
  type-5 gas-puff role).

### Constructor signature

The wrapper's `__init__` exposes **calibration and policy parameters as keyword arguments only**. It
NEVER takes `module_type`, `module_id`, `name`, `data_codes`, or `error_codes` from the caller — those
are hardcoded inside the constructor in the `super().__init__(...)` call. This is the principal
deviation from the ataraxis base, which shows these as caller-provided parameters.

```python
def __init__(
    self,
    encoder_ppr: int,
    wheel_diameter: float,
    cm_per_unity_unit: float,
    polling_frequency: int,
) -> None:
    super().__init__(
        module_type=np.uint8(2),  # hardcoded
        module_id=np.uint8(1),    # hardcoded
        name="encoder",           # hardcoded
        data_codes={np.uint8(51), np.uint8(52)},
        error_codes=None,
    )
    # ... calibration math, shared memory creation, command-code precomputation ...
```

Caller-facing constructor arguments are calibration values, polling frequencies, or domain-specific
configuration (e.g., `valve_calibration_data` is the calibration tuple, not a raw parameter).

### Calibration math in `__init__`

Unit conversions and calibration-driven derived quantities are computed in `__init__` and cached as
`np.float64` instance attributes rounded to 8 decimals for repeatability. Examples:

- `EncoderInterface`: `_cm_per_pulse = round(pi * wheel_diameter / ppr, 8)`,
  `_unity_unit_per_pulse = round(pi * wheel_diameter / (ppr * cm_per_unity_unit), 8)`
- `TorqueInterface`: `_torque_per_adc_unit = round(sensor_capacity * 0.00981 / (max_v - baseline_v), 8)`
- `BrakeInterface`: minimum/maximum strength in g·cm converted to N·cm using `0.00981`
- `ValveInterface`: `curve_fit` of the power-law model yields `_scale_coefficient` and
  `_nonlinearity_exponent`, both rounded to 8 decimals

When a module's calibration requires unit conversion, perform it **in the wrapper**, not in the
binding-class layer. Binding classes pass raw user-input values to the wrapper unchanged.

### Lifecycle: four methods, not three

The ataraxis base defines three abstract methods (`initialize_remote_assets`,
`terminate_remote_assets`, `process_received_data`). sle wrappers that use `SharedMemoryArray` add a
fourth: **`initialize_local_assets()`**.

| Method                     | Process                  | Purpose                                                                                                |
|----------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| `__init__`                 | parent                   | Create `SharedMemoryArray.create_array(..., exists_ok=True)`; compute calibration; cache command codes |
| `initialize_local_assets`  | parent                   | Connect to the shared memory + call `enable_buffer_destruction()` so the parent process owns cleanup   |
| `initialize_remote_assets` | communication subprocess | Connect to the shared memory; initialize non-picklable assets (`PrecisionTimer`)                       |
| `process_received_data`    | communication subprocess | Process incoming `ModuleData` / `ModuleState` messages; update shared memory                           |
| `terminate_remote_assets`  | communication subprocess | Disconnect from shared memory                                                                          |
| `__del__`                  | parent                   | `disconnect()` then `destroy()` the shared memory                                                      |

The `initialize_local_assets()` method is sle-specific; it is called explicitly by binding-class code
before the communication subprocess starts. Wrappers that do not use shared memory may omit it (the
inherited base behavior is a no-op).

Wrappers without shared memory (e.g., `BrakeInterface`, `TorqueInterface`, `ScreenInterface`) leave
both remote-asset methods as `return` no-ops.

### `SharedMemoryArray` naming

Use a deterministic name derived from the module type and id:

```python
self._foo_tracker: SharedMemoryArray = SharedMemoryArray.create_array(
    name=f"{self._module_type}_{self._module_id}_foo_tracker",
    prototype=np.zeros(shape=N, dtype=np.float64 | np.uint64 | np.uint32),
    exists_ok=True,
)
```

The `exists_ok=True` flag is required because the array may already exist from a prior run that did
not clean up; the parent process re-claims and destroys it on shutdown via
`enable_buffer_destruction()`.

### Cached command codes

Cache each command code as an `np.uint8` instance attribute in `__init__` rather than re-creating
`np.uint8(1)` on every call site:

```python
self._check_state: np.uint8 = np.uint8(1)
self._reset_encoder: np.uint8 = np.uint8(2)
```

This avoids repeated numpy scalar allocation on hot paths and makes the available command surface
visible at one glance in the constructor.

### Module-level numpy constants

`module_interfaces.py` defines shared numpy constants at the top of the file for use in hot paths:

```python
_ZERO_UINT64  = np.uint64(0)
_ZERO_FLOAT64 = np.float64(0.0)
_ZERO_UINT32  = np.uint32(0)
_FALSE: np.bool_ = np.bool_(0)
```

Reuse these in `send_command` / `send_parameters` calls instead of creating fresh `np.uint32(0)`
instances each time.

### Public-method patterns

**Typed `set_parameters` wrapper** (mandatory, not optional): expose every parameter-struct field as a
named keyword argument matching the firmware field name. The base ataraxis skill calls this an
"optional" maintainability pattern; sle treats it as the standard surface for any wrapper that sets
runtime parameters.

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
        return  # idempotent — no-op if already in target state
    self.send_command(...)
    self._configured_state = state
```

The idempotency guard (`if state == ...: return`) is part of the convention; it prevents redundant
serial traffic when the binding class repeatedly asserts the same state.

**Cache-and-skip parameter updates**: when a command's parameter struct rarely changes between calls,
cache the last-sent value and only re-send parameters when the new value differs:

```python
if duration_ms != self._previous_pulse_duration:
    self._previous_pulse_duration = duration_ms
    self.send_parameters(...)
self.send_command(...)
```

Exercised by `BrakeInterface.send_pulse`, `ValveInterface.deliver_reward`,
`GasPuffValveInterface.deliver_puff`. Critical for reward-delivery hot paths where the same volume is
delivered hundreds of times per session.

**Cap-and-warn for safety bounds**: when a user-supplied value can exceed a hardware safety limit,
cap to the safe maximum and emit a console warning rather than raising:

```python
if duration_ms > _MAXIMUM_VALVE_PULSE_DURATION_MS:
    console.echo(message=..., level=LogLevel.WARNING)
    duration_ms = _MAXIMUM_VALVE_PULSE_DURATION_MS
```

The `_MAXIMUM_VALVE_PULSE_DURATION_MS = 400` constant is set to keep valve pulses safely below the
500 ms keepalive interval; longer pulses would trigger the keepalive watchdog and abort the runtime.
Any new safety bound MUST cite the underlying constraint (e.g., keepalive interval, motor
current limit) in a top-of-file constant comment.

### Property accessors

Expose derived calibration values and read-only state as `@property`-decorated accessors. Examples:
`cm_per_pulse`, `lick_count`, `delivered_volume`, `maximum_brake_strength`. Consumers (binding
classes) read these to populate their own state without touching wrapper internals.

---

## Cross-side contract

Both sides MUST agree on the following, exactly:

| Item                        | Authority                                                                                                                           |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `module_type` (uint8)       | Type-code registry in `references/module-catalog.md`; same value on both sides                                                      |
| `module_id` (uint8)         | Per-instance; same value on both sides                                                                                              |
| Command codes (1-255)       | Firmware `kModuleCommands` enum is authoritative; Python caches them                                                                |
| Custom event codes (51-250) | Firmware `kCustomStatusCodes` enum is authoritative; Python `data_codes` / `error_codes` reference the same values                  |
| Parameter-struct layout     | Firmware `CustomRuntimeParameters` struct is authoritative; Python `send_parameters` tuple must match field count, order, and types |
| Numpy ↔ C++ type mapping    | See `ataraxis@microcontroller:firmware-module` parameter-struct table                                                               |

A parameter-struct mismatch (e.g., a wrapper sends `np.uint16(x)` for a `uint32_t` field) silently
corrupts every subsequent field because `PACKED_STRUCT` lays the struct out contiguously with no
padding. The same applies to event codes: if the firmware adds a new code at 53 and the Python wrapper
expects it at 52, the wrapper either misroutes the message or raises a spurious RuntimeError on the
PC side.

When modifying either side, modify the other in the same commit (or in tightly coupled commits on a
feature branch). Cross-repository drift is the most common source of "the firmware compiles and the
PC runs but data is garbage" bugs in this stack.

---

## Adding modules — principles

### Decision: new firmware module vs. new wrapper for existing firmware module

Before writing a new firmware module, evaluate whether you actually need new firmware:

1. **New firmware module** is required when the underlying hardware physics or sensing modality is
   different from any existing module: a new sensor type, a new actuator type, a new communication
   protocol with peripheral hardware.

2. **New Python wrapper only** is the right answer when an existing firmware module's behavior is
   physically suitable but the calibration model, exposed surface, or application semantics differ.
   The `ValveModule` + `GasPuffValveInterface` pair is the canonical example: same firmware (solenoid
   pulse + optional tone), different wrapper (no calibration, no tone, different safety bounds).

3. **New command on an existing firmware module** is right when the new capability is a small variant
   of existing commands and the parameter struct can absorb the new fields. Avoid this when the new
   capability would more than double the firmware module's complexity — at that point, fork the
   module class.

The reuse-first preference exists because adding a firmware module forces a firmware rebuild and
firmware reflash on every consumer of that module; adding a Python wrapper is a Python-only change.

### Workflow: adding a paired Module + Interface

1. **Allocate a type code**: pick the next unused value listed in
   [`references/module-catalog.md`](references/module-catalog.md). Update the registry table in that
   file to include the new entry.

2. **Write the firmware**:
   - New header in `slmc/src/<module>_module.h` following the slmc conventions above and the base
     `ataraxis@microcontroller:firmware-module` mechanics.
   - Update `slmc/Doxyfile` `INPUT` list and `slmc/docs/source/api.rst` to include the new header.
   - Update `slmc/src/main.cpp` — add the `#include` and instantiation under the appropriate target
     block (or add a new target — see [Controller board allocation](#controller-board-allocation-principles)).

3. **Write the Python wrapper**:
   - New class in `sle/src/sollertia_experiment/shared_components/module_interfaces.py` following the
     sle conventions above and the base `ataraxis@communication:microcontroller-interface` mechanics.
   - Hardcode `module_type`, `module_id`, `name`, `data_codes`, `error_codes` in `super().__init__()`.
   - Expose calibration as keyword-only constructor arguments.
   - Implement `set_parameters` / `set_state` / domain-specific methods per the sle public-method
     patterns.

4. **Verify the contract**: Walk the [Cross-side contract](#cross-side-contract) table item by item.
   Compile the firmware (`pio run`) and instantiate the wrapper in a Python REPL to check that the
   parameter tuple's numpy types match the firmware struct's C++ types.

5. **Update the catalog**: Add a new block to
   [`references/module-catalog.md`](references/module-catalog.md) following the same template as
   existing entries.

6. **Bump versions**: Bump `slmc`'s version (release git tag — slmc is a firmware project and tracks
   versions through tags, not a manifest file) and `sle`'s `pyproject.toml` version so older
   deployments refuse to load against the new module surface.

7. **Hand off to per-system skills**: Binding-class integration (composing this module into a
   `MicroControllerInterface` instance, surfacing calibration knobs into a system-specific dataclass,
   updating the system configuration YAML schema) is owned by `experiment:mesoscope-vr` for the
   current Mesoscope-VR system. The pattern those steps follow is documented in
   `experiment:acquisition-system-design`.

### Workflow: adding a wrapper for an existing firmware module

Same as above, skipping steps 2 and 6's `slmc` version bump. The new wrapper takes the existing
`module_type` and the next-available `module_id`. Update the registry in
[`references/module-catalog.md`](references/module-catalog.md) to show the additional instance id, and
add a new wrapper subsection to the corresponding catalog block.

---

## Removing modules — principles

Removing a module is a coordinated change across slmc and sle. Before removing:

1. **Audit consumers**: grep all per-system binding-class files for the module's wrapper class name.
   Every consumer must either migrate to a replacement or stop using the module before removal.

2. **Decide retirement vs. deletion**:
   - **Retirement**: Remove the wrapper's instantiation from all binding classes, but leave the
     wrapper class and firmware module in place. Cheaper; preserves the ability to bring the module
     back without a firmware rebuild. Annotate the wrapper class docstring with a deprecation note.
   - **Deletion**: Remove the wrapper class from `module_interfaces.py`, the firmware header from
     `slmc/src/`, the instantiation from `slmc/src/main.cpp`, the Doxygen / Sphinx entries, and the
     catalog block from `references/module-catalog.md`. Only delete after at least one minor-version
     cycle of retirement so downstream deployments can adapt.

3. **Free the type code**: A deleted firmware module's type code becomes available for reuse, but
   prefer allocating the next unused code instead of recycling a freed one. The historical type code
   may still appear in archived log data, and reusing it would conflate old and new modules during
   processing.

4. **Update the catalog**: Remove the module's block from
   [`references/module-catalog.md`](references/module-catalog.md), or annotate it with a "Retired in
   version X" note for retirement.

5. **Bump versions on both sides** so older deployments refuse to load against the new schema.

---

## Controller board allocation principles

Each controller board runs one firmware binary corresponding to one target macro in `main.cpp`. Deciding
which board a module belongs on — or whether a new board is needed — is the most consequential
design decision in this stack. The decision lives at the slmc level because it is a firmware-layout
decision, but it is informed by per-system binding-class needs.

### When to add to an existing controller board

Default to adding new modules to an existing board. Reasons to consolidate:

- **Pin budget headroom**: Each board has a finite pin count (check the target board's spec sheet).
  Each module consumes 1-3 pins. Confirm the target board's free pin count exceeds the new module's
  pin requirements before adding it.
- **Bandwidth headroom**: Each board's serial bandwidth depends on its USB or UART configuration.
  Boards running few high-rate (sub-millisecond polling) sensors generally have bandwidth headroom;
  a high-rate sensor on an already-saturated board may need its own board.
- **Role coherence**: The new module shares the same input/output role as the board's existing
  modules. Mixing input and output modules on one board is permitted but reduces debuggability —
  emergency resets driven by sensor-side keepalive lapses also reset the actuators on the same
  board.

### When to add a new controller board

Stand up a new board (= new target macro in `main.cpp`) when one of these applies:

1. **Interrupt isolation**: The new module needs exclusive control over hardware interrupts (e.g., the
   `EncoderModule`'s `ENCODER_USE_INTERRUPTS` flag is incompatible with any other
   `AttachInterrupt()`-using library on the same board). The slmc Mesoscope-VR ENCODER target exists
   precisely for this reason.

2. **Reset isolation**: The new module's correct operation must not be interrupted if another module
   on the same board causes a keepalive-triggered emergency reset. Actuators that must hold state
   reliably (e.g., a long-running brake engagement) belong on a board separated from high-frequency
   sensors whose polling could lapse the keepalive.

3. **Latency budget**: Multiple polling-style sensors on one board share the `RuntimeCycle()`
   iteration budget. If the new module requires sub-100us polling and the existing board already
   runs several polling sensors, the cumulative cycle time may exceed the budget. Split onto a
   dedicated board.

4. **Pin or bandwidth exhaustion**: An existing board has run out of physical pins for the new module's
   requirements, or the board's USB serial bandwidth is saturated by existing high-rate data. (In
   practice, the boards currently in slmc do not saturate their serial bandwidth from the existing module set; this
   constraint only triggers for hypothetical extreme cases.)

5. **Role separation policy**: The acquisition system's architecture deliberately partitions modules
   by role for debuggability or safety. The Mesoscope-VR ACTOR / SENSOR / ENCODER split is the
   canonical example — ACTOR holds outputs (valves, brake, screen), SENSOR holds inputs (lick,
   torque, TTL), ENCODER is dedicated because of constraint #1.

### Mesoscope-VR as a worked example, not a prescription

The current `main.cpp` defines three targets — ACTOR (id 101), SENSOR (id 152), ENCODER (id 203) —
because Mesoscope-VR is the only acquisition system slmc currently supports. These specific names,
ids, and module assignments are **not** part of the slmc-general convention. They are one valid
application of the principles above. A new acquisition system might:

- Use only one target (a simple system with few modules and no interrupt-isolation needs).
- Use two targets (split by interrupt isolation alone, no role separation).
- Use four or more targets (large system with multiple isolation needs).

The principles, not the example, are what this skill prescribes.

### Workflow: adding a new controller board

1. **Pick a target macro name**: short, all-caps, semantically meaningful (e.g., `STIMULUS`, `RECORD`).
   The macro is conventionally one word; avoid underscores or punctuation. Document the macro's
   purpose in a comment on the `#elif defined <NAME>` line in `main.cpp`.

2. **Allocate a controller ID**: `uint8_t`, must be unique across all controller boards a single
   DataLogger will ingest from. The ataraxis advised range for `MicroControllerInterface` instances
   is 101-150, but the current slmc deployment uses 101, 152, 203 across the three Mesoscope-VR
   boards. Pick a value not currently used by any slmc target and that does not collide with the
   advised ranges of other ataraxis libraries (video systems, etc.). Coordinate with the
   binding-class layer in sle.

3. **Update `main.cpp`**:
   - Add the new `#elif defined <NEW_TARGET>` block.
   - Set `kControllerID` to the chosen value.
   - Include only the headers for modules instantiated on this board.
   - Build the per-board `Module* modules[]` array.

4. **Keep the trailing `#else static_assert(false, ...)` block intact** — it MUST remain the last
   branch so that compilations without a defined target fail with a clear message.

5. **Update slmc README and CLAUDE.md**: Add the new target to the per-target configuration table.
   The README and CLAUDE.md are platform-general and SHOULD list every supported target, not only
   the Mesoscope-VR three.

6. **Hand off to `experiment:mesoscope-vr`** (for the current Mesoscope-VR consumer): The host-PC binding class must add a new
   `MicroControllerInterface` instance for the new board, with the new controller ID and the
   appropriate `ModuleInterface` instances. This skill does not cover that step.

---

## Maintenance contract for this skill

This skill is a knowledge repository split across two files:

- **SKILL.md** (this file) — durable conventions, contracts, principles, and workflows. Update when a
  new convention is established or a workflow changes.
- **[`references/module-catalog.md`](references/module-catalog.md)** — state snapshot of the
  currently-deployed Module + Interface pairs. Update whenever modules are added, removed, or
  modified, per the rules below.

Update `references/module-catalog.md` whenever:

- A new firmware module is added or removed — update the type-code registry table and add or remove
  the corresponding hardware-surface block.
- An existing module's command codes, event codes, parameter struct, or template parameters change —
  update the corresponding catalog block.
- A new `ModuleInterface` wrapper is added or an existing one's constructor signature or
  public-method surface changes — update the wrapper section of the corresponding catalog block.

Update SKILL.md whenever:

- A new target macro is added to `main.cpp` — note it in the
  [Workflow: adding a new controller board](#workflow-adding-a-new-controller-board) section's
  discussion of currently-deployed targets.
- A new slmc or sle convention is established that future modules must follow — add it to the
  appropriate conventions section.

Out-of-date catalog entries are worse than missing entries — an agent acting on stale data will ship
a firmware/wrapper pair that doesn't match the actual codebase. When in doubt, re-read the source
files (`slmc/src/*_module.h`, `sle/.../module_interfaces.py`, `slmc/src/main.cpp`) and reconcile
`references/module-catalog.md` against ground truth.

---

## Related skills

| Skill                                                         | Relationship                                                                                                                                          |
|---------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ataraxis@microcontroller:firmware-module`                    | Authoritative base for C++ `Module` mechanics; this skill defers all base patterns and only adds the slmc layer.                                      |
| `ataraxis@communication:microcontroller-interface`            | Authoritative base for Python `ModuleInterface` mechanics; this skill defers and adds the sle layer.                                                  |
| `ataraxis@communication:microcontroller-setup`                | Post-flash discovery / MQTT verification; called after adding a board or module to confirm the hardware.                                              |
| `automation:cpp-style`                                        | Authoritative for slmc Doxygen file headers, formatting, naming.                                                                                      |
| `automation:python-style`                                     | Authoritative for sle docstrings, type annotations, formatting.                                                                                       |
| `experiment:acquisition-system-setup`                         | Post-flash hardware enumeration / verification at the acquisition-system level.                                                                       |
| `experiment:acquisition-system-design`             | Platform-general pattern for composing wrappers into binding classes and a system configuration. |
| `experiment:mesoscope-vr`                          | Current Mesoscope-VR worked instance — composes the wrappers documented here into `MicroControllerInterfaces`. |
| `experiment:mesoscope-vr-runtime`                  | Mesoscope-VR runtime behavior (state machine, training modes, CLI). Consumes wrapper APIs documented here. |

---

## Verification checklist

```text
When adding or modifying a paired Module + Interface:

Firmware (slmc):
- [ ] Type code allocated from the next unused value in the registry, OR reused id chosen for an existing type with a fresh module_id
- [ ] Header file at slmc/src/<name>_module.h with AXMC_<NAME>_MODULE_H include guards
- [ ] Doxygen file header with @file @brief; per-template-parameter @tparam blocks
- [ ] Class declared as `final` inheriting publicly from Module
- [ ] static_assert(kPin != LED_BUILTIN, ...) for every pin template parameter
- [ ] CustomRuntimeParameters struct uses PACKED_STRUCT and matches the wrapper's send_parameters tuple
- [ ] SetupModule() emits at least one SendData() initial-state report after pin configuration
- [ ] Multi-stage commands use get_command_stage() + AdvanceCommandStage() + WaitForMicros()
- [ ] Blocking calibration commands (if any) carry an @warning Doxygen block
- [ ] digitalWriteFast / pinModeFast used for all pin operations
- [ ] Module added to the appropriate #ifdef target block in main.cpp and to the Module* modules[] array
- [ ] Module added to slmc/Doxyfile INPUT list and slmc/docs/source/api.rst
- [ ] slmc release version bumped (git tag — slmc has no manifest version file)

Wrapper (sle):
- [ ] Class in shared_components/module_interfaces.py named <FirmwareModuleName-without-Module>Interface
- [ ] Constructor exposes calibration as keyword-only parameters; module_type / module_id / name / data_codes / error_codes hardcoded in super().__init__()
- [ ] Calibration math (unit conversion, curve_fit, derived factors) computed in __init__ and rounded to 8 decimals
- [ ] SharedMemoryArray (if used) created with exists_ok=True and named f"{module_type}_{module_id}_<purpose>"
- [ ] initialize_local_assets() implemented for shared-memory parent-process setup
- [ ] initialize_remote_assets() connects shared memory and initializes non-picklable assets (e.g., PrecisionTimer)
- [ ] terminate_remote_assets() disconnects shared memory
- [ ] __del__ disconnects and destroys shared memory
- [ ] Command codes cached as np.uint8 instance attributes in __init__
- [ ] Module-level numpy constants (_ZERO_UINT64, _FALSE, etc.) reused on hot paths
- [ ] Typed set_parameters() with named keyword args matching firmware field names and types
- [ ] State-toggling methods use keyword-only `*, state: bool` and an idempotency guard
- [ ] Cache-and-skip applied to parameter updates that rarely change
- [ ] Cap-and-warn applied to user-supplied values that could exceed hardware safety bounds
- [ ] sle pyproject.toml version bumped

Cross-side contract:
- [ ] module_type and module_id match between firmware constructor args (main.cpp) and super().__init__() (wrapper)
- [ ] Command codes in the wrapper match the firmware kModuleCommands enum value-for-value
- [ ] Event codes in data_codes / error_codes match the firmware kCustomStatusCodes enum value-for-value
- [ ] Parameter-struct field count, order, and numpy/C++ types match (verify against ataraxis Python ↔ C++ type table)
- [ ] Both repositories committed in the same change set or tightly coupled commits

Catalog (`references/module-catalog.md`):
- [ ] Type-code registry table updated with the new (or removed) entry
- [ ] Hardware-surface block added (or removed) with all fields populated
- [ ] If a new controller board target was introduced, SKILL.md board-allocation discussion updated

Verification:
- [ ] pio run succeeds for every affected target
- [ ] After flash, the new module is visible via /microcontroller-setup discovery
- [ ] A Python REPL can instantiate the wrapper without error and round-trip a set_parameters / send_command call
```
