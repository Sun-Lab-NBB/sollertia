# slmc firmware conventions

These conventions extend or deviate from `ataraxis@microcontroller:firmware-module` and apply to every
slmc `Module` subclass. See [`../SKILL.md`](../SKILL.md) for the registry, cross-side contract,
allocation rules, and workflows that govern this layer.

---

## Header conventions

- **Include guards**: `AXMC_<MODULE_NAME>_MODULE_H` (e.g., `AXMC_ENCODER_MODULE_H`). The base skill
  permits `#pragma once`; slmc uses traditional guards with the `AXMC_` prefix for consistency with the
  upstream ataraxis-micro-controller library.
- **File-header Doxygen**: Every header starts with `/** @file @brief ... */` and uses `@warning`,
  `@note`, `@tparam`, `@param` tags. `ataraxis@automation:cpp-style` is the authoritative reference for the
  format; this skill notes only that file headers are mandatory.
- **Header-only modules**: All `Module` subclasses live entirely in `.h` files. No `.cpp` files exist
  under `src/` for module classes; everything is template-instantiated from `main.cpp`. New modules
  MUST follow this pattern.
- **`final` on Module subclass**: Every `Module` subclass uses `class FooModule final : public Module`.
  Subclassing of an existing slmc Module is not supported.
- **Explicit defaulted destructor**: `~FooModule() override = default;` — included for completeness
  even when no destructor logic is needed.

---

## Template-parameterized pin and behavior configuration

Every slmc module is a class template whose pin assignments are compile-time parameters. Modules add
further compile-time parameters only for behavior that is fixed at build time — relay polarity
(`kNormallyClosed`, `kNormallyEngaged`), signal direction (`kInvertDirection`), I/O mode (`kOutput`), or
initial state (`kStartClosed`) — while modules that need none (e.g., `LickModule<kPin>`) take only their
pin(s):

```cpp
template <const uint8_t kPin, const bool kNormallyClosed, const bool kStartClosed = true>
class ValveModule final : public Module { ... };
```

- All pin parameters are `const uint8_t`.
- Polarity flags (`kNormallyEngaged`, `kNormallyClosed`, `kNormallyOff`) are `const bool`.
- Calibration constants the firmware needs at compile time (e.g., `kBaseline` on `TorqueModule`) are
  template parameters, not runtime parameters.
- Optional pins use **255** as the sentinel value (`kUnusedTonePin = 255`; see `ValveModule`). The
  current tone-pin checks are runtime `if (kTonePin != kUnusedTonePin)` guards, but prefer resolving
  compile-time-known choices as `constexpr` (as with the polarity constants below) wherever it reads
  cleanly; a runtime guard is a fallback for cases a `constexpr` form would not express cleanly.

---

## Mandatory `LED_BUILTIN` static_assert

Every pin template parameter MUST be paired with a `static_assert(kPin != LED_BUILTIN, ...)` at the top
of the class body, before `public:`. The base ataraxis skill recommends this; slmc treats it as
mandatory because the LED-builtin pin is reserved for runtime error indication and accidental reuse
silently destroys debuggability.

When a module has multiple pins, every pin gets its own static_assert with a specific error message
(see `EncoderModule` with pins A, B, X).

---

## `constexpr` polarity logic

For modules with `kNormallyOpen` / `kNormallyClosed` / `kNormallyEngaged` polarity flags, derive the
"active" and "inactive" signal levels as `static constexpr bool` members instead of branching at
runtime:

```cpp
static constexpr bool kEngage    = kNormallyEngaged ? LOW  : HIGH;
static constexpr bool kDisengage = kNormallyEngaged ? HIGH : LOW;
```

Then use `kEngage` / `kDisengage` directly in `digitalWriteFast()` calls. This pattern appears in
`BrakeModule`, `ValveModule`, and `ScreenModule`.

---

## Initial-state reporting from `SetupModule()`

Every `SetupModule()` MUST emit at least one `SendData(...)` call after configuring its pins to
inform the PC of the module's starting state. This is a slmc convention beyond the ataraxis base —
the host-PC wrappers and downstream log processing assume every module reports an initial state, so
omitting this leaves a wrapper's instantaneous-state tracker (e.g., `_valve_tracker[2]`) unsynchronized
with the firmware.

For input-side modules, the initial-state report uses a zero-magnitude payload (e.g., `kRotatedCW, 0`
on `EncoderModule`) to establish a baseline that subsequent delta-threshold logic can compare against.

---

## Stage-based commands and blocking exceptions

Multistep commands (any command that involves a timed delay) MUST use the stage-based pattern:
`get_command_stage()` → `AdvanceCommandStage()` → `WaitForMicros()` → `CompleteCommand()`. The base
skill documents the mechanics; slmc uses this pattern for every output-pulse command
(`BrakeModule::SendPulse`, `ValveModule::Pulse`, `ValveModule::Tone`, `ScreenModule::Toggle`,
`TTLModule::SendPulse`).

**Blocking exceptions**: `ValveModule::Calibrate` and `EncoderModule::GetPPR` block the runtime
in-place via `delayMicroseconds()` / busy-wait loops. They are explicitly marked with `@warning`
Doxygen blocks and are intended only for offline calibration — never during an active acquisition
session. If a new module needs a long blocking command for calibration, follow the same `@warning`
convention.

---

## Performance primitives

- Prefer `digitalWriteFast` / `pinModeFast` / `digitalReadFast` over the standard `digitalWrite` /
  `pinMode` / `digitalRead` for all pin operations whose pin is known at compile time (every slmc
  module pin qualifies because pins are template parameters).
- The `Encoder` library member must be statically initialized at class scope (`Encoder _encoder =
  Encoder(kPinA, kPinB);`). Deferred initialization in `SetupModule()` crashes the runtime; this is a
  Paul Stoffregen library constraint, not an slmc choice.

---

## Multi-target `main.cpp` via preprocessor

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
following the same structure. See
[Controller board allocation principles](../SKILL.md#controller-board-allocation-principles) for when
to add a new target vs. extending an existing one.

The fallback `static_assert(false, ...)` block under `#else` MUST remain — compiling without selecting
a target should fail loudly.

The keepalive interval (`kKeepaliveInterval = 500` ms) and `analogReadResolution(12)` are set
globally for all targets in `main.cpp` and apply to every module regardless of board.
