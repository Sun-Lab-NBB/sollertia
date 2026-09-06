# slmc firmware conventions

These conventions extend or deviate from `microcontroller:firmware-module` (ataraxis marketplace) and apply to every
slmc `Module` subclass. See [`../SKILL.md`](../SKILL.md) for the registry, cross-side contract, allocation rules, and
workflows that govern this layer.

---

## Header conventions

- **Include guards**: `SLMC_<MODULE_NAME>_MODULE_H` (e.g., `SLMC_ENCODER_MODULE_H`). This is the
  `LIBRARY_PREFIX_FILE_NAME_H` form `automation:cpp-style` mandates, instantiated with slmc's `SLMC_` prefix so a
  module header cannot collide with a same-named header in the upstream ataraxis-micro-controller library.
- **File-header Doxygen**: Every header starts with `/** @file @brief ... */` and uses `@warning`, `@note`, `@tparam`,
  `@param` tags. `automation:cpp-style` is the authoritative reference for the format, and this skill notes only that
  file headers are mandatory.
- **Header-only modules**: All `Module` subclasses live entirely in `.h` files. No `.cpp` files exist under `src/` for
  module classes, and everything is template-instantiated from `main.cpp`. New modules MUST follow this pattern.
- **`final` on Module subclass**: Every `Module` subclass uses `class FooModule final : public Module`. Subclassing of
  an existing slmc Module is not supported.
- **Explicit defaulted destructor**: `~FooModule() override = default;`, included for completeness even when no
  destructor logic is needed.

---

## Template-parameterized pin and behavior configuration

Every slmc module is a class template whose pin assignments are compile-time parameters. Modules add further
compile-time parameters only for behavior that is fixed at build time, covering relay polarity (`kNormallyClosed`,
`kNormallyEngaged`), signal direction (`kInvertDirection`), I/O mode (`kOutput`), and initial state (`kStartClosed`).
Modules that need none of those, such as `LickModule<kPin>`, take only their pin(s):

```cpp
template <
    const uint8_t kValvePin,
    const bool kNormallyClosed,
    const bool kStartClosed = true,
    const uint8_t kTonePin  = kUnusedTonePin,
    const bool kNormallyOff = true,
    const bool kStartOff    = true>
class ValveModule final : public Module { ... };
```

- All pin parameters are `const uint8_t`.
- Polarity flags (`kNormallyEngaged`, `kNormallyClosed`, `kNormallyOff`) are `const bool`.
- Calibration constants the firmware needs at compile time (e.g., `kBaseline` on `TorqueModule`) are template
  parameters, not runtime parameters.
- Optional pins use **255** as the sentinel value (`kUnusedTonePin = 255`, see `ValveModule`). Resolve the sentinel
  comparison into a single `static constexpr bool` member (`kToneEnabled = kTonePin != kUnusedTonePin`) and branch on it
  with `if constexpr`, so the pin-absent path is elided at compile time instead of being retested on every cycle. A
  runtime guard is a fallback only for cases a `constexpr` form would not express cleanly.

---

## Named parameter defaults

Every field of a module's `CustomRuntimeParameters` struct takes its default from a named `static constexpr` member
rather than an inline literal, and `SetupModule()` assigns the same constants:

```cpp
struct CustomRuntimeParameters
{
        uint16_t signal_threshold = kDefaultSignalThreshold;  ///< The minimum voltage level reported.
        uint8_t average_pool_size = kDefaultAveragePoolSize;  ///< The number of readouts averaged.
} PACKED_STRUCT _custom_parameters;

/// Stores the default minimum voltage level to report to the PC, in 12-bit ADC units. The value sits just
/// above the typical noise floor.
static constexpr uint16_t kDefaultSignalThreshold = 300;
```

The constant is where the value's unit, derivation, and rationale are documented, so the struct field and the
`SetupModule()` assignment cannot drift apart or carry two different explanations of the same number. All seven modules
follow this pattern, and the naming is `kDefault<FieldNameInPascalCase>`. A constant that expresses a hardware limit
takes a name describing what it bounds, as with `BrakeModule`'s `kMaximumDutyCycle`, `kFullEngageDuty`, and
`kFullDisengageDuty`.

Changing one of these constants changes the module's post-reset behavior, so treat it as a cross-side change and
reconcile it with the host-PC configuration that overwrites it at session start. See
[`../SKILL.md`](../SKILL.md#cross-side-contract).

---

## Mandatory `LED_BUILTIN` static_assert

Every pin template parameter MUST be paired with a `static_assert(kPin != LED_BUILTIN, ...)` at the top of the class
body, before `public:`. The base ataraxis skill recommends this, and slmc treats it as mandatory because the LED-builtin
pin is reserved for runtime error indication and accidental reuse silently destroys debuggability.

When a module has multiple pins, every pin gets its own static_assert with a specific error message (see `EncoderModule`
with pins A, B, X).

A multi-pin module additionally asserts that its pins are distinct from one another. `EncoderModule` rejects any equal
pair among A, B, and X, and `ValveModule` rejects a tone pin equal to the valve pin, written so the `kUnusedTonePin`
sentinel still passes:

```cpp
static_assert(
    kTonePin == kUnusedTonePin || kValvePin != kTonePin,
    "The solenoid valve and the tone buzzer cannot share a pin. Select a different tone pin for the "
    "ValveModule instance."
);
```

Two peripherals wired to one pin is a configuration error the compiler can catch, so catch it there.

---

## `constexpr` polarity logic

For modules with `kNormallyClosed` / `kNormallyEngaged` / `kNormallyOff` polarity flags, derive the "active" and
"inactive" signal levels as `static constexpr bool` members instead of branching at runtime:

```cpp
static constexpr bool kEngage    = kNormallyEngaged ? LOW  : HIGH;
static constexpr bool kDisengage = kNormallyEngaged ? HIGH : LOW;
```

Then use `kEngage` / `kDisengage` directly in `digitalWriteFast()` calls. This pattern appears in `BrakeModule`,
`ValveModule`, and `ScreenModule`. `BrakeModule` is the one module that routes every command-path digital write
through a private `WriteDigital()` helper instead of calling `digitalWriteFast()` inline. `SetupModule()` is the
exception: it writes inline because the `pinMode()` call directly above it already reclaimed the pin. The module also
drives the pin with `analogWrite()`, so it has to reclaim the pin from the PWM peripheral before the write lands.

---

## Initial-state reporting from `SetupModule()`

Every `SetupModule()` MUST emit at least one `SendData(...)` call after configuring its pins to inform the PC of the
module's starting state. This is a slmc convention beyond the ataraxis base. The host-PC wrappers and downstream log
processing assume every module reports an initial state, so omitting this leaves a wrapper's instantaneous-state tracker
(e.g., `_valve_tracker[2]`) unsynchronized with the firmware.

For input-side modules, the initial-state report uses a zero-magnitude payload (e.g., `kRotatedCW, 0` on
`EncoderModule`) to establish a baseline against which subsequent delta-threshold logic compares.

---

## Per-instance state, and restoring it in `SetupModule()`

Cross-call state belongs to an instance member, never to a function-local `static`. A `static` local inside a template
method is shared by every instance of that specialization, so two instances of the same module on one board would
silently share one change-detection baseline. `LickModule::_previous_readout`, `TorqueModule::_previous_readout`, and
`TTLModule::_previous_input_status` are all members for this reason.

`SetupModule()` MUST then restore every one of those members alongside the pins and the parameter struct. The Kernel
re-runs `SetupModule()` on every controller reset and every keepalive timeout, and a member left holding its pre-reset
value contradicts the zero baseline the same method just reported to the PC:

```cpp
// Realigns the change-detection state with the zero baseline reported below. The Kernel re-runs this
// method on every controller reset and keepalive timeout, so the state has to be restored alongside it.
_previous_readout = 0;
_previous_zero    = true;
```

The same rule covers derived caches and peripheral trackers: `EncoderModule` re-derives its amortization caps,
`ValveModule` clears `_tone_active` and re-derives `_tone_time_delta`, and `BrakeModule` clears `_analog_mode` to match
the `pinMode()` call it just made. The test is simple: any member written outside `SetupModule()` is a member
`SetupModule()` has to reset.

---

## Stage-based commands and blocking exceptions

Multistep commands (any command that involves a timed delay) MUST use the stage-based pattern: `get_command_stage()` →
`AdvanceCommandStage()` → `WaitForMicros()` → `CompleteCommand()`. The base skill documents the mechanics, and slmc
uses this pattern for every output-pulse command (`BrakeModule::SendPulse`, `ValveModule::Pulse`, `ValveModule::Tone`,
`ScreenModule::Toggle`, `TTLModule::SendPulse`).

**Blocking exceptions**: `ValveModule::Calibrate` and `EncoderModule::GetPPR` block the runtime in-place via
`delayMicroseconds()` / busy-wait loops. They are explicitly marked with `@warning` Doxygen blocks and are intended only
for offline calibration, never during an active acquisition session. If a new module needs a long blocking command for
calibration, follow the same `@warning` convention.

---

## Pin primitives

- **Pin writes** use the Teensy core's `digitalWriteFast()`, an always-inline register-level write. Its pin argument is
  a compile-time constant in every slmc module, because pins are template parameters. `BrakeModule` reaches it through
  `WriteDigital()` for the reason given under [`constexpr` polarity logic](#constexpr-polarity-logic).
- **Pin modes** use the Teensy core's `pinMode()`, called from `SetupModule()` in all seven headers.
- **Pin reads** in polling modules go through the base `Module` helpers `AnalogRead<kPin>(pool_size)` and
  `DigitalRead<kPin>(pool_size)`, which take the pin as a template argument and apply the instance's `average_pool_size`
  in one call. `LickModule`, `TorqueModule`, and `TTLModule` all read this way. `EncoderModule::GetPPR` reads the index
  pin with the core's `digitalReadFast()` directly, since it busy-waits on that pin and has no pool size to apply.
- The `Encoder` library member must be statically initialized at class scope
  (`Encoder _encoder = Encoder(kPinA, kPinB);`). Deferred initialization in `SetupModule()` crashes the runtime, which
  is a Paul Stoffregen library constraint rather than an slmc choice.

---

## Multi-target `main.cpp` via preprocessor

slmc partitions its modules across multiple microcontroller boards using a single `main.cpp` with preprocessor `#ifdef`
blocks:

```cpp
#ifdef ACTOR   // The building PlatformIO environment defines exactly one target macro.
#include "brake_module.h"
// ... the remaining ACTOR headers ...

    static constexpr uint8_t kControllerID = 101;
    BrakeModule<33, false, true> wheel_brake(3, 1, axmc_communication);
    // ... the remaining ACTOR instances ...
    Module* modules[] = {&wheel_brake, /* ... */};

#elif defined SENSOR
    static constexpr uint8_t kControllerID = 152;
    // ... the SENSOR includes, instances, and modules[] array ...

#elif defined ENCODER
    static constexpr uint8_t kControllerID = 203;
    // ... the ENCODER includes, instances, and modules[] array ...

#else
static_assert(
    false,
    "Unable to resolve the target microcontroller. Build with a PlatformIO environment that defines one of the "
    "supported target macros (ACTOR, SENSOR, ENCODER)."
);
#endif
```

The module `#include` directives sit inside the selected branch, so a target compiles only the headers it instantiates
(`slmc/src/main.cpp`). `Kernel axmc_kernel(kControllerID, axmc_communication, modules, kKeepaliveInterval);` follows
the block and is target-agnostic, consuming whichever `kControllerID` and `modules[]` the selected branch defined.

A PlatformIO environment in `platformio.ini` supplies the target macro through its `build_flags`. The environments are
named `<board>_<target>` and extend a shared, non-buildable `[<board>_base]` template holding every field common to
them. `pio run` without `-e` then compiles every target and fails on a break in a target other than the one being
flashed. Adding a target therefore means adding both an `#elif defined` block and its matching environment.

This is the slmc-general pattern for supporting multiple controller boards from one firmware codebase. The current slmc
deployment defines exactly three target macros, `ACTOR`, `SENSOR`, and `ENCODER` (`slmc/src/main.cpp`). A new
acquisition system defines new target macros, such as `STIMULUS` or `RECORD`, following the same structure. See
[Controller board allocation principles](../SKILL.md#controller-board-allocation-principles) for when to add a new
target rather than extending an existing one.

The fallback `static_assert(false, ...)` block under `#else` MUST remain the last branch, and its message MUST list
every supported target macro, so a build that selects none fails with the list it could have chosen from
(`slmc/src/main.cpp`).

Three named constants at the top of `main.cpp` are set globally for all targets and apply to every module regardless of
board. They are `kKeepaliveInterval = 500` milliseconds, `kSerialBaudRate = 115200`, which Teensy boards ignore, and
`kAnalogReadResolution = 12`, the 0-4095 readout range the analog modules assume. All three are cross-repo constants, so
see the constants table in [`../SKILL.md`](../SKILL.md#cross-repo-constants-that-move-together) for the host mirror each
one carries.

The target-selection block is wrapped in a `NOLINTBEGIN(*-magic-numbers)` / `NOLINTEND` band. Its literals are
hardware assignments, covering pin numbers, module type codes, per-controller instance ids, and the torque
sensor's ADC baseline. A named constant would restate the number without adding meaning. Keep new target blocks inside
the band rather than suppressing the check per line.

---

## The `platformio.ini` build matrix

`[teensy41_base]` in `slmc/platformio.ini` is the template every environment extends. The name omits the `env:` prefix,
which is what keeps PlatformIO from building the template itself.

| Key             | Value      | Key               | Value        |
|-----------------|------------|-------------------|--------------|
| `platform`      | `teensy`   | `test_framework`  | `unity`      |
| `board`         | `teensy41` | `upload_protocol` | `teensy-cli` |
| `framework`     | `arduino`  | `build_flags`     | `-std=c++17` |
| `monitor_speed` | `115200`   | `check_tool`      | `clangtidy`  |

Three environments extend it, `[env:teensy41_actor]`, `[env:teensy41_sensor]`, and `[env:teensy41_encoder]`, each
appending one `-D <MACRO>` to the inherited `build_flags`. Every environment therefore inherits board `teensy41` and
monitor speed 115200, so Teensy 4.1 is the only board family slmc targets today. A second board family
means a second non-`env:` template plus one environment per target macro.

`lib_deps` holds three caret-pinned entries: `inkaros/ataraxis-transport-layer-mc@^4.0.2`,
`inkaros/ataraxis-micro-controller@^4.0.3`, and `paulstoffregen/Encoder@^1.4.4`. A new third-party library is added here
with the same caret pin. `slmc` ships no `library.json`, because it is a firmware project rather than a PlatformIO
library, so `automation:platformio-config`'s `lib_deps` mirroring rule does not apply.

---

## Flashing a target

Each controller board runs one target firmware, so an upload names the environment carrying that target's macro. The
experimenter runs every upload, because a flash needs physical access to the board.

```bash
pio run -e <board>_<target> -t upload    # Compiles one target and flashes it to the connected board
```

Two rules govern the procedure, and `slmc/README.md` states both as the operator-facing source of truth:

- Connect exactly one microcontroller for the duration of an upload, because some hosts select the wrong upload target
  when several boards are attached. Flash one board, disconnect it, then attach the next one.
- Always pass `-e`. An upload command naming no environment processes every environment in turn, flashing the connected
  board with each target firmware and leaving it running the last one.

Which environment a given physical board takes is a consumer-owned assignment rather than a firmware-general one,
because slmc defines the target mechanism while the acquisition system defines the targets themselves. The consuming
system's skill names the environment, controller id, and module set of each board it wires, and that skill is
`mesoscope:mesoscope-vr` today.

A board carrying the wrong target announces the wrong controller id, so the `MicroControllerInterface` bound to that
port raises `ValueError` during its identification handshake. Re-flash a board whenever a module's command codes, event
codes, or parameter-struct layout change, because the wire format is positional and carries no field names.

---

## Clang-tidy gate

`platformio.ini` enables clang-tidy as a PlatformIO check tool, so `pio check` is a gate on firmware changes alongside
`pio run`. The curated check list lives in `.clang-tidy`, and the `check_flags` key in `slmc/platformio.ini` narrows
and corrects the invocation through six flags:

- `--config-file=.clang-tidy` is mandatory. PlatformIO appends `--checks=*` when it is absent, which discards the
  curated list.
- `--header-filter=.*/sollertia-micro-controllers/src/.*` restricts reporting to this repository's own headers, keeping
  framework, toolchain, and dependency headers out.
- `--extra-arg=--target=arm-none-eabi` gives clang the board's pointer and integer widths.
- `--extra-arg=-ferror-limit=0` keeps clang parsing to the end of the translation unit. Without it clang stops at the
  twentieth error and analyses a truncated syntax tree, which reports findings the source does not contain.
- `--extra-arg=-Wno-invalid-constexpr` and `--extra-arg=-Wno-unusable-partial-specialization` silence the GCC libstdc++
  headers that the bundled clang 15 cannot fully parse. A compiler diagnostic bypasses the header filter, so these two
  have to be turned off at the source of the invocation.

A new module is expected to pass the gate without new suppressions. Where a suppression is unavoidable, use the
narrowest `NOLINT` form that covers the site and state the reason in the adjacent comment.
