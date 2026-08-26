# sollertia-micro-controllers extension seams

Carries seams 29 through 32 of the extension seam table in [SKILL.md](../SKILL.md). Firmware paths are cited as
`slmc/...`, and sollertia-experiment mirrors are cited relative to `src/sollertia_experiment/`.
`/microcontroller-interface` owns the paired Module and Interface conventions, and
`microcontroller:firmware-module` owns the base ataraxis `Module` mechanics these steps extend.

---

## Seam 1, a new firmware module

Nine steps, in order.

1. Create `slmc/src/<name>_module.h` guarded by `SLMC_<NAME>_MODULE_H`, following the `SLMC_BRAKE_MODULE_H` guard
   of `slmc/src/brake_module.h`.
2. Declare `template <...> class <Name>Module final : public Module`, following `BrakeModule` in
   `slmc/src/brake_module.h`, and add a `static_assert(kPin != LED_BUILTIN, ...)` per pin, following the same class.
   A multi-pin module also asserts its pins pairwise distinct, following `EncoderModule` in
   `slmc/src/encoder_module.h` and `ValveModule` in `slmc/src/valve_module.h`.
3. Declare `enum class kCustomStatusCodes : uint8_t` starting at **51** and `enum class kModuleCommands : uint8_t`
   starting at **1**. The convention holds in all seven existing headers.
4. Give the class a three-argument constructor forwarding `(module_type, module_id, Communication&)` to `Module`,
   following the `BrakeModule` constructor in `slmc/src/brake_module.h`. `module_type` is supplied in `main.cpp`
   rather than baked into the class.
5. Override the three virtuals. `SetCustomParameters()` calls `ExtractParameters(_custom_parameters)`, following
   `TorqueModule` in `slmc/src/torque_module.h`. `RunActiveCommand()` switches over
   `static_cast<kModuleCommands>(get_active_command())` and returns `false` from `default`, following `BrakeModule`
   in `slmc/src/brake_module.h`. `SetupModule()` assigns every parameter default **and** re-initializes every
   mutable tracker, because the Kernel re-runs it on each reset and keepalive timeout. `TorqueModule::SetupModule`
   in `slmc/src/torque_module.h` and `ValveModule::SetupModule` in `slmc/src/valve_module.h` show both halves.
6. Declare the parameter struct as `struct CustomRuntimeParameters { ... } PACKED_STRUCT _custom_parameters;`,
   following the `CustomRuntimeParameters` of `BrakeModule` in `slmc/src/brake_module.h`.
7. Keep every multi-stage command non-blocking, driving it through a `get_command_stage()` switch with
   `AdvanceCommandStage()`, `WaitForMicros()`, `CompleteCommand()`, and `AbortCommand()`, as `ScreenModule::Toggle`
   in `slmc/src/screen_module.h` and `TTLModule::SendPulse` in `slmc/src/ttl_module.h` do. Blocking past
   `kKeepaliveInterval` in `slmc/src/main.cpp`, which is 500 ms, trips the Kernel's emergency reset.
8. Wire the module into `slmc/src/main.cpp`: add the `#include` inside the target's `#ifdef` block, instantiate it
   with a unique `(module_type, module_id)` pair, and append its address to that target's `modules[]` array.
9. Pin any new third-party library in the `lib_deps` of `[teensy41_base]` with a caret constraint
   (`slmc/platformio.ini`).

**The synchronized sollertia-experiment change is required.** The module stays unusable until a matching
`ModuleInterface` subclass exists that carries the same `module_type`, `module_id`, command codes, status codes, and
parameter field order. The eight existing mirrors live in `cross_system/module_interfaces.py`.

| Type | Id | Interface class              |
|------|----|------------------------------|
| 1    | 1  | `MesoscopeFrameTTLInterface` |
| 2    | 1  | `EncoderInterface`           |
| 3    | 1  | `BrakeInterface`             |
| 4    | 1  | `LickInterface`              |
| 5    | 1  | `WaterValveInterface`        |
| 5    | 2  | `GasPuffValveInterface`      |
| 6    | 1  | `TorqueInterface`            |
| 7    | 1  | `ScreenInterface`            |

Two of those pairs share a firmware family. Type 5 carries two ids, so one firmware module family backs two wrappers
with different behavior, which is the pattern a new role specialization follows.

---

## Seam 2, a new controller target

Three steps.

1. Add `[env:<board>_<name>]` to `slmc/platformio.ini` with `extends = teensy41_base` and
   `build_flags = ${teensy41_base.build_flags} -D <MACRO>`, following `[env:teensy41_actor]` in
   `slmc/platformio.ini`.
2. Add an `#elif defined <MACRO>` branch to `slmc/src/main.cpp`, following its `SENSOR` and `ENCODER` branches,
   holding the module `#include`s, a `static constexpr uint8_t kControllerID`, the module instantiations, and a
   `Module* modules[]` array.
3. Add the macro to the `static_assert` message in the `#else` branch that lists the supported targets, and to the
   target-macro comment above the selection block, both in `slmc/src/main.cpp`.

No board, `lib_deps`, or Kernel edits are needed. `Kernel axmc_kernel(...)` in `slmc/src/main.cpp` is target-agnostic
and consumes whichever `kControllerID` and `modules[]` the selected branch defined.

**The synchronized sollertia-experiment change is required.** The controller ID is mirrored in a
`MicroControllerInterface(controller_id=...)` call, following `MicroControllerInterfaces.__init__` in
`mesoscope_vr/binding_classes.py`. The `module_interfaces` tuple beside each ID lists exactly the interfaces whose
firmware counterparts appear in that target's `modules[]`. The ataraxis advised `MicroControllerInterface` ID range
and the IDs the current slmc deployment uses are documented in `/microcontroller-interface`.

---

## Seam 3, a new board family

Five steps.

1. Add a second non-`env:` template to `slmc/platformio.ini` mirroring `[teensy41_base]`, with the new `platform`,
   `board`, `framework`, and `upload_protocol`, and with its own `check_flags ... --extra-arg=--target=<arch>` entry.
   The current value is `arm-none-eabi`, and it makes clang apply the board's pointer and integer widths.
2. Add one `[env:<board>_<target>]` per controller target extending the new template, each defining the same target
   macro as its Teensy sibling (`slmc/platformio.ini`).
3. Re-verify every `LED_BUILTIN` `static_assert` in the module headers under `slmc/src/`, and every pin literal in
   the target blocks of `slmc/src/main.cpp`, because
   pin numbering is board-specific.
4. Confirm the board supports the `kAnalogReadResolution` of 12 bits that `setup()` passes to
   `analogReadResolution()` in `slmc/src/main.cpp`. That ADC width underpins the 12-bit-unit parameter defaults of
   the analog sensor modules and the analog module baselines set in `main.cpp`.
5. Confirm the `paulstoffregen/Encoder` entry of `lib_deps` (`slmc/platformio.ini`) supports the architecture's
   interrupt pins, because `slmc/src/encoder_module.h` forces `ENCODER_USE_INTERRUPTS`. Keep `kSerialBaudRate`
   (`slmc/src/main.cpp`) matching `monitor_speed` (`slmc/platformio.ini`), since Teensy ignores the value and other
   architectures do not.

**The synchronized sollertia-experiment change happens only when controller IDs, event codes, or the baud rate
change.** The host side binds on a serial port plus a controller ID and on `_MICROCONTROLLER_BAUDRATE`
(`interfaces/get.py`), so it is otherwise board-agnostic.

Every environment today inherits `board = teensy41` and `monitor_speed = 115200` from the `[teensy41_base]` template
(`slmc/platformio.ini`), so Teensy 4.1 is the only board family for which the firmware builds.
`automation:platformio-config` owns the `platformio.ini` conventions these steps edit.

---

## Seam 4, constants that move together

Eight constants exist on both sides of the serial boundary, and each row moves as a unit. A one-sided change produces
firmware that compiles and a host that runs while the data between them is garbage, because `PACKED_STRUCT` carries
no padding and no field is self-describing on the wire.

| # | Constant                                        | Firmware side                                           | sollertia-experiment side                                                                                                                             |
|---|-------------------------------------------------|---------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | Keepalive interval, 500 ms                      | `kKeepaliveInterval` in `slmc/src/main.cpp`             | The `keepalive_interval` each system passes to `MicroControllerInterface` (`MicroControllerInterfaces.__init__` in `mesoscope_vr/binding_classes.py`) |
| 2 | Serial baud rate, 115200                        | `kSerialBaudRate` in `slmc/src/main.cpp`                | `_MICROCONTROLLER_BAUDRATE` in `interfaces/get.py`                                                                                                    |
| 3 | Controller IDs                                  | `kControllerID` in `slmc/src/main.cpp`                  | The `controller_id` arguments in `MicroControllerInterfaces.__init__` (`mesoscope_vr/binding_classes.py`)                                             |
| 4 | `(module_type, module_id)` pairs                | The target blocks of `slmc/src/main.cpp`                | The `module_type` and `module_id` each wrapper passes to `super().__init__` (`cross_system/module_interfaces.py`)                                     |
| 5 | `kCustomStatusCodes` values                     | The seven module headers                                | The `data_codes` and `error_codes` sets of each wrapper (`cross_system/module_interfaces.py`)                                                         |
| 6 | `kModuleCommands` values                        | The seven module headers                                | The per-interface command constants such as `_pulse`, `_open`, and `_close` (`cross_system/module_interfaces.py`)                                     |
| 7 | `CustomRuntimeParameters` field order and types | The seven module headers                                | The parameter payload each wrapper's `set_parameters` sends                                                                                           |
| 8 | Calibration count, 200                          | `kDefaultCalibrationCount` in `slmc/src/valve_module.h` | `self._calibration_count = np.uint16(200)` in `WaterValveInterface` (`cross_system/module_interfaces.py`)                                             |

Rows 5, 6, and 7 are per-module. `/microcontroller-interface` carries the full catalog of every module's status
codes, command codes, and parameter fields, so verify a row against that catalog rather than restating it here.

---

## Version source of truth

The firmware repository ships no `library.json`, because it is a firmware project rather than a PlatformIO library.
Two files declare the version instead, `PROJECT_NUMBER = "5.0.0"` in `slmc/Doxyfile` and `release = '5.0.0'` in
`slmc/docs/source/conf.py`, while `slmc/platformio.ini` carries no version field. Update both to match the
intended release tag, because a one-sided bump leaves the Sphinx pages stamped with the previous version.

---

## Operational caution

Two firmware commands block the runtime in place and are offline-only. `ValveModule::Calibrate`
(`slmc/src/valve_module.h`) blocks through `delayMicroseconds()` for the whole calibration burst, and
`EncoderModule::GetPPR` (`slmc/src/encoder_module.h`) spins on unbounded index-pin waits until the operator
rotates the encoder through every measured revolution. Neither one belongs in an active behavior session.

Calibration and hardware positioning are experimenter-operated through the maintenance runtime GUI. You MUST NOT
write an instruction that has an agent upload firmware, run a `pio` command, or drive hardware.
