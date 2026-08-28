---
name: microcontroller-interface
description: >-
  Registry of paired Module (sollertia-micro-controllers) and ModuleInterface (sollertia-experiment) classes available
  to Sollertia acquisition systems, plus the conventions on top of the ataraxis base templates and principles for adding
  modules or controller boards. Use when extending hardware support or modifying a paired Module + Interface.
user-invocable: false
---

# Sollertia microcontroller interface

Documents the Sollertia platform's paired microcontroller interface stack at the pre-binding-class level:

- **slmc** (`sollertia-micro-controllers`, C++ firmware) holds the `Module` subclasses that run on Arduino-compatible
  microcontroller boards. Teensy 4.1 is the only board family the current `platformio.ini` targets
  (the `[teensy41_base]` template and its three `[env:teensy41_*]` environments in `slmc/platformio.ini`), and the
  conventions in this skill carry to any board family slmc adds.
- **sle** (`sollertia-experiment`, Python) holds the `ModuleInterface` subclasses in
  `src/sollertia_experiment/cross_system/module_interfaces.py` that wrap the firmware modules. The pairing is
  directional, **not one-to-one**: each wrapper binds to exactly one firmware module type, but a single type can be
  wrapped by several role-specific interfaces (e.g., `ValveModule` → `WaterValveInterface` + `GasPuffValveInterface`).

This skill is the canonical registry of paired modules currently available to any Sollertia acquisition system and the
source of truth for the slmc/sle conventions layered on top of the ataraxis base templates. Binding-class composition,
which assembles `MicroControllerInterface` instances, configuration dataclasses, and system configurations, is
**system-specific** and lives in per-system skills. Those are currently `mesoscope:mesoscope-vr` for hardware
composition and `mesoscope:mesoscope-vr-runtime` for runtime behavior. The platform-general pattern those skills follow
lives in `/acquisition-system-design`.

---

## Scope

**Covers:**
- Registry of available paired Modules + Interfaces with data surface, calibration knobs, command and event codes
- slmc firmware conventions beyond the ataraxis `Module` base template
- sle Python wrapper conventions beyond the ataraxis `ModuleInterface` base template
- Cross-side contract requirements (what must agree between firmware and Python wrapper)
- Principles for adding or removing modules on both sides
- Principles for allocating modules across controller boards
- Workflows for adding paired modules and adding new controller boards

**Does not cover:**
- Base `Module` / `ModuleInterface` API, `PACKED_STRUCT` mechanics, `SendData` patterns, event-code ranges,
  `MicroControllerInterface` lifecycle, MQTTCommunication, DataLogger topology, keepalive mechanics. See
  `microcontroller:firmware-module` (ataraxis marketplace) and `communication:microcontroller-interface`.
- Microcontroller discovery, manifest management via MCP tools. See `communication:microcontroller-setup`.
- C++ style, Python style, header guards conventions enforcement. See `automation:cpp-style`, `automation:python-style`.
- **Binding-class composition** (per-system `MicroControllerInterfaces`, configuration dataclasses, system configuration
  YAML, runtime orchestration), covered by `/acquisition-system-design` for the platform-general pattern and
  `mesoscope:mesoscope-vr` for the current Mesoscope-VR instance. This skill ends at the Python wrapper layer, and
  everything that composes those wrappers into a runnable acquisition system is owned downstream.
- Hardware discovery / post-flash verification. See `/acquisition-system-setup`.

---

## Authoritative bases

Before reading the conventions or workflows in this skill, you MUST be familiar with the ataraxis base templates this
layer builds on:

| Concern                                         | Authority                                 |
|-------------------------------------------------|-------------------------------------------|
| C++ `Module` API, parameter structs, `SendData` | `microcontroller:firmware-module`         |
| Python `ModuleInterface` API, abstract methods  | `communication:microcontroller-interface` |
| Wire protocol, event-code ranges, message types | Either ataraxis skill (mirrored sections) |
| Microcontroller discovery and verification      | `communication:microcontroller-setup`     |
| `platformio.ini` and `library.json` conventions | `automation:platformio-config`            |
| C++ formatting, naming, Doxygen blocks          | `automation:cpp-style`                    |
| Python formatting, typing, docstrings           | `automation:python-style`                 |

The Sollertia layer **inherits all base mechanics and only documents deviations and expansions**. When a section below
cites a base behavior, treat the ataraxis skill as the source of truth.

---

## Hardware-surface registry

The current registry of paired Modules + Interfaces lives in
[`references/module-catalog.md`](references/module-catalog.md). It records type codes, parameter structs, event codes,
commands, calibration knobs, shared-memory layouts, and the public method surface of each wrapper. That file is a state
snapshot of the currently-deployed pair set and MUST be updated whenever a module is added, removed, or modified (see
[Maintenance contract](#maintenance-contract-for-this-skill)).

### Type-code allocation rules

These rules are durable and govern how new codes are assigned. Consult the catalog file for the currently-used values.

- Module type codes are `uint8_t` in the range 1-255. The base `ModuleInterface` constructor raises `TypeError` for any
  value outside that range, so 0 is unusable as a type code. Separately, command code 0 is reserved by the firmware
  runtime as the "no active command" sentinel, so command enums start at 1.
- The `(module_type, module_id)` pair MUST be unique on a single controller board. Two firmware instances of the same
  `Module` subclass on the same board take different `module_id` values.
- When allocating a new type code, pick the next unused value from the catalog rather than recycling a freed one.
  Historical type codes may still appear in archived log data and reuse would conflate old and new modules during
  processing.
- Custom event codes use the 51-250 range per the ataraxis base. See `microcontroller:firmware-module` for the full
  range table.

---

## slmc firmware conventions (deviations from ataraxis base)

The slmc firmware `Module` subclass conventions that extend or deviate from `microcontroller:firmware-module` live in
[`references/slmc-conventions.md`](references/slmc-conventions.md). They cover header guards, template-parameterized
pins, named parameter defaults, and the mandatory `LED_BUILTIN` and pin-collision static_asserts. They also cover
`constexpr` polarity logic, initial-state reporting from `SetupModule()`, per-instance state and its restoration on
reset, stage-based commands and blocking exceptions, pin primitives, the multi-target `main.cpp` pattern, and the
clang-tidy gate. Every slmc `Module` subclass MUST follow them.

---

## sle Python wrapper conventions (deviations from ataraxis base)

The sle Python `ModuleInterface` wrapper conventions that extend or deviate from
`communication:microcontroller-interface` live in [`references/sle-conventions.md`](references/sle-conventions.md). They
cover file location, constructor signatures, calibration math in `__init__`, the four-method lifecycle,
`SharedMemoryArray` naming, cached command codes, module-level numpy constants, the public-method patterns, and property
accessors. They apply to every `ModuleInterface` subclass in
`src/sollertia_experiment/cross_system/module_interfaces.py`.

---

## Shared logging contract

The `MicroControllerInterface` communication process auto-logs every message sent to or received from the
microcontroller through its `SerialCommunication` instance, so no wrapper in `cross_system/module_interfaces.py`
writes its own log entries. `data_codes` select which received events
additionally reach `process_received_data()`, and `error_codes` map the event codes that raise `RuntimeError` and abort
the runtime. Both sets draw their values from the same firmware `kCustomStatusCodes` enum, and every code must lie in
the custom event-code range. See `communication:microcontroller-interface` for the base mechanics.

The `SharedMemoryArray` buffers that five wrappers maintain are live IPC state read by other runtime processes. They
are not a logging channel, so a value that must survive the session reaches disk through the auto-logged message rather
than through the array.

---

## Cross-side contract

Both sides MUST agree on the following, exactly:

| Item                        | Authority                                                                                                                           |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `module_type` (uint8)       | Type-code registry in `references/module-catalog.md`, same value on both sides                                                      |
| `module_id` (uint8)         | Per-instance, same value on both sides                                                                                              |
| Command codes (1-255)       | Firmware `kModuleCommands` enum is authoritative, Python caches them                                                                |
| Custom event codes (51-250) | Firmware `kCustomStatusCodes` enum is authoritative, Python `data_codes` / `error_codes` reference the same values                  |
| Parameter-struct layout     | Firmware `CustomRuntimeParameters` struct is authoritative, Python `send_parameters` tuple must match field count, order, and types |
| Numpy ↔ C++ type mapping    | See `microcontroller:firmware-module` parameter-struct table                                                                        |

A parameter-struct mismatch (e.g., a wrapper sends `np.uint16(x)` for a `uint32_t` field) silently corrupts every
subsequent field because `PACKED_STRUCT` lays the struct out contiguously with no padding. The same applies to event
codes: if the firmware adds a new code at 53 and the Python wrapper expects it at 52, the wrapper either misroutes the
message or raises a spurious RuntimeError on the PC side.

When modifying either side, modify the other in the same commit (or in tightly coupled commits on a feature branch).
Cross-repository drift is the most common source of "the firmware compiles and the PC runs but data is garbage" bugs in
this stack.

---

## Cross-repo constants that move together

Each row names a value that is declared twice, once in firmware and once on the host. A one-sided change usually
produces a runtime that connects and then misreads every message. Two rows fail differently. Teensy boards ignore the
declared `kSerialBaudRate` value (`slmc/src/main.cpp`), so that row binds only on a board family that honors it. The ADC
resolution corrupts no message at all, and the note below the table gives its failure mode. Citations under `slmc/` are
firmware, and the rest are relative to `src/sollertia_experiment/`.

| Constant                              | Firmware declaration                                                             | Host mirror                                                                                                 |
|---------------------------------------|----------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Keepalive interval, 500 ms            | `kKeepaliveInterval` (`slmc/src/main.cpp`)                                       | The active system's configuration passes it to every `MicroControllerInterface`                             |
| Serial baud rate, 115200              | `kSerialBaudRate` (`slmc/src/main.cpp`), `monitor_speed` (`slmc/platformio.ini`) | `_MICROCONTROLLER_BAUDRATE: int = 115200` (`interfaces/get.py`)                                             |
| Controller ids 101, 152, 203          | the per-target `kControllerID` constants (`slmc/src/main.cpp`)                   | The active system's binding class passes each id to one `MicroControllerInterface`                          |
| `(module_type, module_id)` pairs      | the first two arguments of each module instantiation (`slmc/src/main.cpp`)       | the `module_type` and `module_id` arguments in each wrapper's `super().__init__()` (`module_interfaces.py`) |
| `kCustomStatusCodes` values           | each `slmc/src/<name>_module.h`                                                  | the `data_codes` and `error_codes` sets in each wrapper's `__init__` (`module_interfaces.py`)               |
| `kModuleCommands` values              | each `slmc/src/<name>_module.h`                                                  | the cached `np.uint8` command attributes in each wrapper's `__init__` (`module_interfaces.py`)              |
| `CustomRuntimeParameters` field order | each `slmc/src/<name>_module.h`                                                  | the tuple each wrapper's `set_parameters()` hands to `send_parameters()`                                    |
| Valve calibration count, 200          | `kDefaultCalibrationCount` (`slmc/src/valve_module.h`)                           | `self._calibration_count = np.uint16(200)` (`WaterValveInterface.__init__` in `module_interfaces.py`)       |
| Brake full-strength value, 255        | `kMaximumDutyCycle` (`slmc/src/brake_module.h`)                                  | `_MAXIMUM_BRAKING_STRENGTH: np.uint8 = np.uint8(255)` (`module_interfaces.py`)                              |
| ADC resolution, 12 bits               | `kAnalogReadResolution` (`slmc/src/main.cpp`)                                    | the `*_adc` calibration fields of the active system's configuration (`mesoscope_vr/system.py`)              |
| Module template parameters            | the template arguments of each instantiation (`slmc/src/main.cpp`)               | `torque_baseline_voltage_adc: int = 2048` for `TorqueModule`'s `kBaseline` (`mesoscope_vr/system.py`)       |
| Per-target module layout              | each target's `modules[]` array (`slmc/src/main.cpp`)                            | the `module_interfaces` tuple of each `MicroControllerInterface` (`mesoscope_vr/binding_classes.py`)        |

The ADC resolution is the row that fails silently. `analogReadResolution(kAnalogReadResolution)` in `slmc/src/main.cpp`
fixes the readout range at 0 to 4095, and every ADC-unit value on both sides is scaled to it: `TorqueModule`'s
`kBaseline` template argument of 2048 and the `kDefault*` thresholds of the analog modules in firmware, and the `*_adc`
calibration fields in the active system's configuration. Narrowing the width keeps every struct the same size, so the
wire stays valid and each message still parses while its numbers silently mean something else. Treat a change to it as a
re-calibration of both sides rather than a firmware-only setting.

The keepalive interval also bounds the host-side valve safety cap. `_MAXIMUM_VALVE_PULSE_DURATION_MS = 400` is set
below the 500 ms interval so a pulse cannot outlast the handshake (`module_interfaces.py`). The controller ids
and the interval reach the firmware through the acquisition system's own configuration, so the two rows naming the
active system route through that system's skill. `mesoscope:mesoscope-vr` holds the current worked example.

---

## Adding modules

### Decision: new firmware module vs. new wrapper for existing firmware module

Before writing a new firmware module, evaluate whether you actually need new firmware:

1. **New firmware module** is required when the underlying hardware physics or sensing modality is different from any
   existing module: a new sensor type, a new actuator type, a new communication protocol with peripheral hardware.

2. **New Python wrapper only** is the right answer when an existing firmware module's behavior is physically suitable
   but the calibration model, exposed surface, or application semantics differ. The `ValveModule` +
   `GasPuffValveInterface` pair is the canonical example: same firmware (solenoid pulse + optional tone), different
   wrapper (no calibration, no tone, different safety bounds).

3. **New command on an existing firmware module** is right when the new capability is a small variant of existing
   commands and the parameter struct can absorb the new fields. Avoid this when the new capability would more than
   double the firmware module's complexity. At that point, fork the module class.

The reuse-first preference exists because adding a firmware module forces a firmware rebuild and firmware reflash on
every consumer of that module, while adding a Python wrapper is a Python-only change.

### Workflow: adding a paired Module + Interface

1. **Allocate a type code**: pick the next unused value listed in
   [`references/module-catalog.md`](references/module-catalog.md). Update the registry table in that file to include the
   new entry.

2. **Write the firmware**:
   - New header in `slmc/src/<module>_module.h` following the slmc conventions above and the base
     `microcontroller:firmware-module` mechanics.
   - Update `slmc/Doxyfile` `INPUT` list and `slmc/docs/source/api.rst` to include the new header.
   - Update `slmc/src/main.cpp` to add the `#include` and the instantiation under the appropriate target block. Add a
     new target instead when [Controller board allocation](#controller-board-allocation-principles) calls for one.

3. **Write the Python wrapper**:
   - New class in `cross_system/module_interfaces.py` following the sle conventions above and the base
     `communication:microcontroller-interface` mechanics.
   - Hardcode `module_type`, `module_id`, `name`, `data_codes`, `error_codes` in `super().__init__()`.
   - Expose calibration and policy values as regular constructor parameters that call sites pass by keyword. Reserve
     true keyword-only syntax (a `*` separator) for the binary state setters.
   - Implement `set_parameters` / `set_state` / domain-specific methods per the sle public-method patterns.
   - Add the class to the `from .module_interfaces import (...)` block and to `__all__` in
     `cross_system/__init__.py`, because every binding class imports its wrappers from the package rather than the
     submodule, through the `from ..cross_system import (...)` block of `mesoscope_vr/binding_classes.py`.

4. **Verify the contract**: Walk the [Cross-side contract](#cross-side-contract) table item by item. Compile the
   firmware (`pio run`), clear the clang-tidy gate (`pio check`), and instantiate the wrapper in a Python REPL to check
   that the parameter tuple's numpy types match the firmware struct's C++ types.

5. **Update the catalog**: Add a new block to [`references/module-catalog.md`](references/module-catalog.md) following
   the same template as existing entries.

6. **Bump versions**: Bump `sle`'s `pyproject.toml` version so older deployments refuse to load against the new module
   surface. slmc is a firmware project rather than a PlatformIO library, so it ships no `library.json` and
   `slmc/platformio.ini` carries no version field. slmc declares its version in two places, `PROJECT_NUMBER` at
   `slmc/Doxyfile` and `release` in `slmc/docs/source/conf.py`. Both stamp the generated API documentation and both
   MUST be bumped, in lockstep, alongside the sle version.

7. **Hand off to per-system skills**: Binding-class integration (composing this module into a `MicroControllerInterface`
   instance, surfacing calibration knobs into a system-specific dataclass, updating the system configuration YAML
   schema) is owned by the consuming system's own skill, currently `mesoscope:mesoscope-vr`. The platform-general
   pattern those steps follow is documented in `/acquisition-system-design`.

   The acquisition half of the handoff leaves the module logging events that nothing downstream reads, so the
   processing half is part of the same change. The system's hardware-state snapshot gains the calibration field or
   usage boolean that records the module, on Mesoscope-VR through `mesoscope:mesoscope-vr-session-schema`. The parser
   that converts the module's logged events into a processed feather file is added through
   `mesoscope:mesoscope-vr-module-parsing`, which also owns the registry entry that makes the parser reachable. A
   module wired into a binding class without both edits acquires data for which no processed output ever exists.

### Workflow: adding a wrapper for an existing firmware module

Same as above, skipping steps 2 and 6's `slmc` version bump. The new wrapper takes the existing `module_type` and the
next-available `module_id`. Update the registry in [`references/module-catalog.md`](references/module-catalog.md) to
show the additional instance id, and add a new wrapper subsection to the corresponding catalog block.

---

## Removing modules

Removing a module is a coordinated change across slmc and sle. Before removing:

1. **Audit consumers**: grep all per-system binding-class files for the module's wrapper class name. Every consumer must
   either migrate to a replacement or stop using the module before removal.

2. **Decide retirement vs. deletion**:
   - **Retirement**: Remove the wrapper's instantiation from all binding classes, but leave the wrapper class and
     firmware module in place. Cheaper, and preserves the ability to bring the module back without a firmware rebuild.
     Annotate the wrapper class docstring with a deprecation note.
   - **Deletion**: Remove the wrapper class from `module_interfaces.py`, the firmware header from `slmc/src/`, the
     instantiation from `slmc/src/main.cpp`, the Doxygen / Sphinx entries, and the catalog block from
     `references/module-catalog.md`. Only delete after at least one minor-version cycle of retirement so downstream
     deployments can adapt.

3. **Free the type code**: A deleted firmware module's type code becomes available for reuse, and the rule under
   [Type-code allocation rules](#type-code-allocation-rules) still governs, so allocate the next unused code instead.

4. **Update the catalog**: Remove the module's block from
   [`references/module-catalog.md`](references/module-catalog.md), or annotate it with a "Retired in version X" note for
   retirement.

5. **Bump versions on both sides** so older deployments refuse to load against the new schema.

---

## Controller board allocation principles

Each controller board runs one firmware binary corresponding to one target macro in `main.cpp`. Deciding which board a
module belongs on, and whether a new board is needed at all, is the most consequential design decision in this stack.
The decision lives at the slmc level because it is a firmware-layout decision, and per-system binding-class needs
inform it.

The consolidation and split criteria, the current slmc deployment and its reuse ordering, and the step-by-step workflow
for adding a board live in [`references/board-allocation.md`](references/board-allocation.md).

---

## Extension

`/library-extension` catalogues the three slmc seams, new firmware module, new controller target, and new board family,
together with the sollertia-experiment mirrors each one obliges. This skill owns the paired-module conventions and the
catalog of what currently exists, and that skill owns the seam map a new acquisition system walks.

---

## Maintenance contract for this skill

This skill is a knowledge repository split across five files, and each one carries its own update trigger.

| File                                                               | Holds                                                 | Update when                                                                                   |
|--------------------------------------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| SKILL.md                                                           | Durable conventions, contracts, principles, workflows | A new target macro reaches `main.cpp`, or a contract, allocation rule, or workflow changes    |
| [`references/board-allocation.md`](references/board-allocation.md) | Split criteria, target set, add-a-board workflow      | A target or controller id changes, a module moves targets, or the criteria or workflow change |
| [`references/module-catalog.md`](references/module-catalog.md)     | The state snapshot of the deployed pairs              | A module or wrapper is added, removed, or changed in any surface the catalog records          |
| [`references/slmc-conventions.md`](references/slmc-conventions.md) | The firmware conventions every `Module` follows       | A new slmc convention is established that future modules must follow                          |
| [`references/sle-conventions.md`](references/sle-conventions.md)   | The wrapper conventions every interface follows       | A new sle convention is established that future wrappers must follow                          |

A changed parameter **default**, meaning any constant a `CustomRuntimeParameters` field initializes from in
`slmc/src/<name>_module.h`, is the one catalog
trigger that fires alone. The struct layout, the command codes, and the event codes all stay valid, so the firmware
compiles and the wrapper still matches while the "Boot defaults" row goes quietly wrong. An agent acting on a stale
catalog ships a pair that the codebase does not contain, so re-read `slmc/src/*_module.h`, `slmc/src/main.cpp`, and
`cross_system/module_interfaces.py` and reconcile the catalog whenever the entry is in doubt.

---

## Related skills

Every entry prefixed `microcontroller:`, `communication:`, or `automation:` resolves through the ataraxis marketplace.

| Skill                                     | Relationship                                                                                                     |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `microcontroller:firmware-module`         | Authoritative base for C++ `Module` mechanics. This skill defers all base patterns and only adds the slmc layer. |
| `communication:microcontroller-interface` | Authoritative base for Python `ModuleInterface` mechanics. This skill defers and adds the sle layer.             |
| `communication:microcontroller-setup`     | Post-flash discovery and MQTT verification, run after adding a board or module to confirm the hardware.          |
| `automation:cpp-style`                    | Authoritative for slmc Doxygen file headers, formatting, naming.                                                 |
| `automation:python-style`                 | Authoritative for sle docstrings, type annotations, formatting.                                                  |
| `automation:platformio-config`            | Authoritative for `platformio.ini` structure, per-board environments, and pinned `lib_deps`.                     |
| `/library-extension`                      | Owns the slmc module, target, and board seams, and the sollertia-experiment mirror each one obliges.             |
| `/acquisition-system-setup`               | Post-flash hardware enumeration and verification at the acquisition-system level.                                |
| `/acquisition-system-design`              | Platform-general pattern for composing wrappers into binding classes and a system configuration.                 |
| `mesoscope:mesoscope-vr`                  | Current worked instance, composes the wrappers documented here into its microcontroller binding class.           |
| `mesoscope:mesoscope-vr-runtime`          | The worked instance's runtime behavior, which consumes the wrapper APIs documented here.                         |
| `mesoscope:mesoscope-vr-session-schema`   | The worked instance's hardware-state snapshot, which records a new module's calibration or usage.                |
| `mesoscope:mesoscope-vr-module-parsing`   | The worked instance's parser layer, which turns a new module's logged events into processed output.              |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Firmware (slmc):
- [ ] Type code allocated from the next unused value in the registry, OR reused id chosen for an existing type
      with a fresh module_id
- [ ] Header file at slmc/src/<name>_module.h with SLMC_<NAME>_MODULE_H include guards
- [ ] Doxygen file header with @file @brief, per-template-parameter @tparam blocks
- [ ] Class declared as `final` inheriting publicly from Module
- [ ] static_assert(kPin != LED_BUILTIN, ...) for every pin template parameter
- [ ] Multi-pin module additionally static_asserts that its pins are distinct from one another
- [ ] CustomRuntimeParameters struct uses PACKED_STRUCT and matches the wrapper's send_parameters tuple
- [ ] Every parameter-struct default comes from a named static constexpr member, kDefault<Field> for a plain
      default and a bound-describing name for a hardware limit (BrakeModule's kFullEngageDuty), which is
      also where the value's unit and derivation are documented
- [ ] SetupModule() emits at least one SendData() initial-state report after pin configuration
- [ ] Cross-call state lives in instance members, never in function-local statics
- [ ] SetupModule() restores every member written outside it, since the Kernel re-runs it on reset and
      keepalive timeout
- [ ] Multi-stage commands use get_command_stage() + AdvanceCommandStage() + WaitForMicros()
- [ ] Blocking calibration commands (if any) carry an @warning Doxygen block
- [ ] digitalWriteFast used for pin writes, pinMode for pin modes, AnalogRead<kPin> / DigitalRead<kPin>
      for polled pin reads
- [ ] Module added to the appropriate #ifdef target block in main.cpp and to the Module* modules[] array
- [ ] Module added to slmc/Doxyfile INPUT list and slmc/docs/source/api.rst
- [ ] PROJECT_NUMBER in slmc/Doxyfile and release in slmc/docs/source/conf.py bumped to the same value, the two
      in-repository version declarations slmc carries

Wrapper (sle):
- [ ] Class in cross_system/module_interfaces.py named <FirmwareModuleName-without-Module>Interface
      (or <Role>Interface for a role-specific wrapper, e.g. WaterValveInterface, MesoscopeFrameTTLInterface)
- [ ] Constructor exposes calibration as regular parameters that call sites pass by keyword
- [ ] module_type / module_id / name / data_codes / error_codes hardcoded in super().__init__()
- [ ] Calibration math (unit conversion, curve_fit, derived factors) computed in __init__ and cached at full
      np.float64 precision
- [ ] SharedMemoryArray (if used) created with exists_ok=True and named
      f"{int(self._module_type)}_{int(self._module_id)}_<purpose>"
- [ ] initialize_local_assets() implemented for shared-memory parent-process setup, and the system's binding class
      calls it after the owning MicroControllerInterface has started
- [ ] initialize_remote_assets() connects shared memory and initializes non-picklable assets (e.g., PrecisionTimer)
- [ ] terminate_remote_assets() disconnects shared memory and drops non-picklable assets
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
- [ ] If a new controller board target was introduced, references/board-allocation.md's current-deployment
      section updated

Verification:
- [ ] pio run succeeds, compiling every target in one invocation
- [ ] pio check reports no new clang-tidy findings for any target
- [ ] After flash, the new module is visible via communication:microcontroller-setup discovery
- [ ] A Python REPL can instantiate the wrapper without error and round-trip a set_parameters / send_command call
```
