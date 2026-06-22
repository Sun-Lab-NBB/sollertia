---
name: microcontroller-interface
description: >-
  Registry of paired Module (sollertia-micro-controllers) and ModuleInterface
  (sollertia-experiment) classes available to Sollertia acquisition systems, plus the
  conventions on top of the ataraxis base templates and principles for adding modules or
  controller boards. Use when extending hardware support or modifying a paired Module + Interface.
user-invocable: false
---

# Sollertia microcontroller interface

Documents the Sollertia platform's paired microcontroller interface stack at the pre-binding-class level:

- **slmc** (`sollertia-micro-controllers`, C++ firmware) — `Module` subclasses that run on
  Arduino-compatible microcontroller boards. The current deployment uses Teensy 4.1 boards, but the
  firmware library is board-agnostic; this skill's conventions apply to any board slmc targets.
- **sle** (`sollertia-experiment`, Python) — `ModuleInterface` subclasses in
  `src/sollertia_experiment/cross_system/module_interfaces.py` that wrap the firmware modules. The
  pairing is directional, **not one-to-one**: each wrapper binds to exactly one firmware module type,
  but a single type can be wrapped by several role-specific interfaces (e.g., `ValveModule` →
  `WaterValveInterface` + `GasPuffValveInterface`).

This skill is the canonical registry of paired modules currently available to any Sollertia acquisition
system and the source of truth for the slmc/sle conventions layered on top of the ataraxis base templates.
Binding-class composition (assembling `MicroControllerInterface` instances, configuration dataclasses,
system configurations) is **system-specific** and lives in per-system skills (currently
`mesoscope:mesoscope-vr` for hardware composition and `mesoscope:mesoscope-vr-runtime` for
runtime behavior), not here. The platform-general pattern those skills follow lives in
`/acquisition-system-design`.

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
  see `ataraxis@automation:cpp-style`, `ataraxis@automation:python-style`.
- **Binding-class composition** (per-system `MicroControllerInterfaces`, configuration dataclasses,
  system configuration YAML, runtime orchestration) — covered by `/acquisition-system-design`
  for the platform-general pattern and `mesoscope:mesoscope-vr` for the current Mesoscope-VR
  instance. This skill ends at the Python wrapper layer; everything that composes
  those wrappers into a runnable acquisition system is owned downstream.
- Hardware discovery / post-flash verification — see `/acquisition-system-setup`.

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

The slmc firmware `Module` subclass conventions that extend or deviate from
`ataraxis@microcontroller:firmware-module` — header guards, template-parameterized pins, the mandatory
`LED_BUILTIN` static_assert, `constexpr` polarity logic, initial-state reporting from `SetupModule()`,
stage-based commands and blocking exceptions, performance primitives, and the multi-target `main.cpp`
pattern — live in [`references/slmc-conventions.md`](references/slmc-conventions.md). Every slmc
`Module` subclass MUST follow them.

---

## sle Python wrapper conventions (deviations from ataraxis base)

The sle Python `ModuleInterface` wrapper conventions that extend or deviate from
`ataraxis@communication:microcontroller-interface` — file location, keyword-only calibration
constructors, calibration math in `__init__`, the four-method lifecycle, `SharedMemoryArray` naming,
cached command codes, module-level numpy constants, the public-method patterns, and property accessors
— live in [`references/sle-conventions.md`](references/sle-conventions.md). They apply to every
`ModuleInterface` subclass in `src/sollertia_experiment/cross_system/module_interfaces.py`.

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
   - New class in `sle/src/sollertia_experiment/cross_system/module_interfaces.py` following the
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
   updating the system configuration YAML schema) is owned by `mesoscope:mesoscope-vr` for the
   current Mesoscope-VR system. The pattern those steps follow is documented in
   `/acquisition-system-design`.

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
because Mesoscope-VR is the only acquisition system slmc currently supports. These specific names, ids,
and module assignments are **not** a mandatory layout, and a new acquisition system is free to
re-partition.

"Not a prescription" does **not** mean "build from scratch." The targets above, and the modules assigned
to them, are **reusable assets** — the whole stack is designed for reuse, and reuse is heavily preferred
over standing up new firmware. When bringing up a new acquisition system, prefer, in order:

1. **Reuse an existing target unchanged** when its module set already covers the new system's needs —
   the same firmware binary can drive a different rig with no rebuild.
2. **Reuse existing modules on an existing target** — add an already-defined module (or a new Python
   wrapper around one, per [Adding modules](#adding-modules--principles)) to a target that still has pin
   and bandwidth headroom.
3. **Add a new target** only when the criteria in
   [When to add a new controller board](#when-to-add-a-new-controller-board) force a split no existing
   target can absorb.

A new system might therefore reuse all three targets, reuse one, or add a fourth — but that choice is
driven by the principles and a reuse-first bias, never by reflexively copying or discarding the
Mesoscope-VR layout. The principles and the preference for reuse, not the specific three-target example,
are what this skill prescribes.

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

6. **Hand off to `mesoscope:mesoscope-vr`** (for the current Mesoscope-VR consumer): The host-PC
   binding class must add a new `MicroControllerInterface` instance for the new board, with the new
   controller ID and the appropriate `ModuleInterface` instances. This skill does not cover that step.

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

| Skill                                              | Relationship                                                                                                     |
|----------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `ataraxis@microcontroller:firmware-module`         | Authoritative base for C++ `Module` mechanics; this skill defers all base patterns and only adds the slmc layer. |
| `ataraxis@communication:microcontroller-interface` | Authoritative base for Python `ModuleInterface` mechanics; this skill defers and adds the sle layer.             |
| `ataraxis@communication:microcontroller-setup`     | Post-flash discovery / MQTT verification; called after adding a board or module to confirm the hardware.         |
| `ataraxis@automation:cpp-style`                    | Authoritative for slmc Doxygen file headers, formatting, naming.                                                 |
| `ataraxis@automation:python-style`                 | Authoritative for sle docstrings, type annotations, formatting.                                                  |
| `/acquisition-system-setup`                        | Post-flash hardware enumeration / verification at the acquisition-system level.                                  |
| `/acquisition-system-design`                       | Platform-general pattern for composing wrappers into binding classes and a system configuration.                 |
| `mesoscope:mesoscope-vr`                           | Current Mesoscope-VR worked instance — composes the wrappers documented here into `MicroControllerInterfaces`.   |
| `mesoscope:mesoscope-vr-runtime`                   | Mesoscope-VR runtime behavior (state machine, training modes, CLI). Consumes wrapper APIs documented here.       |

---

## Verification checklist

```text
When adding or modifying a paired Module + Interface:

Firmware (slmc):
- [ ] Type code allocated from the next unused value in the registry, OR reused id chosen for an existing type
      with a fresh module_id
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
- [ ] Class in cross_system/module_interfaces.py named <FirmwareModuleName-without-Module>Interface
      (or <Role>Interface for a role-specific wrapper, e.g. WaterValveInterface, MesoscopeFrameTTLInterface)
- [ ] Constructor exposes calibration as keyword-only parameters; module_type / module_id / name /
      data_codes / error_codes hardcoded in super().__init__()
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
- [ ] After flash, the new module is visible via ataraxis@communication:microcontroller-setup discovery
- [ ] A Python REPL can instantiate the wrapper without error and round-trip a set_parameters / send_command call
```
