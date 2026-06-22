# Module catalog

State snapshot of the currently-deployed paired Modules + Interfaces in the Sollertia microcontroller
stack:

- **sollertia-micro-controllers** firmware: C++ `Module` subclasses in `src/*.h`
- **sollertia-experiment** Python: `ModuleInterface` subclasses in
  `src/sollertia_experiment/cross_system/module_interfaces.py`

This file documents **what is currently deployed**, not the rules for extending the stack. See
[`../SKILL.md`](../SKILL.md) for the conventions, cross-side contract, allocation rules, and
workflows that govern this layer. Update this file whenever a module is added, removed, or modified
(see SKILL.md's "Maintenance contract" section).

---

## Type-code registry

The Sollertia platform currently uses type codes 1-7.

| Type | Module          | Interface                                                    | Direction       | Instance ids in use | Notes                                                                 |
|------|-----------------|--------------------------------------------------------------|-----------------|---------------------|-----------------------------------------------------------------------|
| 1    | `TTLModule`     | `MesoscopeFrameTTLInterface`                                 | Input OR Output | 1                   | Per-instance pin mode is set at compile time via `kOutput` template   |
| 2    | `EncoderModule` | `EncoderInterface`                                           | Input           | 1                   | Uses `ENCODER_USE_INTERRUPTS`; incompatible with other interrupt libs |
| 3    | `BrakeModule`   | `BrakeInterface`                                             | Output          | 1                   | PWM-controlled electromagnetic particle brake                         |
| 4    | `LickModule`    | `LickInterface`                                              | Input           | 1                   | Analog ADC-threshold conductive sensor                                |
| 5    | `ValveModule`   | `WaterValveInterface` (id 1), `GasPuffValveInterface` (id 2) | Output          | 1, 2                | Same firmware module; two Python wrappers with different calibration  |
| 6    | `TorqueModule`  | `TorqueInterface`                                            | Input           | 1                   | AD620-amplified analog torque sensor                                  |
| 7    | `ScreenModule`  | `ScreenInterface`                                            | Output          | 1                   | Pulses FET gates on VR-screen power boards                            |

**Next unused code:** 8.

The Mesoscope-VR ACTOR board currently exercises the multi-`module_id` pattern with `ValveModule`
instances at id 1 (water reward) and id 2 (gas puff). No other module currently has multiple
instances on a single controller board.

A firmware `Module` is generic and reusable, but its `ModuleInterface` wrapper may be specialized for
specific equipment — in its name, its calibration, and how its data is processed downstream.
`MesoscopeFrameTTLInterface` (a generic `TTLModule` named and processed for the mesoscope's
frame-acquisition signal) and the `WaterValveInterface` / `GasPuffValveInterface` split over a single
`ValveModule` are both examples. The specialization is intentional; an equipment-specific interface still
lives in the shared `cross_system` layer and stays reusable by any system that drives the same hardware.

---

## Hardware-surface catalog

Each block below documents one firmware `Module` and the interface(s) that wrap it as a contract unit —
a firmware type may have more than one interface (`ValveModule` has two). The firmware section names
the C++ class, template parameters, custom event codes, and commands; the Python section(s) name the
interface class, constructor calibration knobs, shared-memory state surfaced to other processes, and
the public methods exposed to consumers.

### TTLModule + MesoscopeFrameTTLInterface (type 1)

**Firmware**: `src/ttl_module.h` — `TTLModule<kPin, kOutput, kStartOn>`. A single instance is either an
output or an input, not both. Output mode supports `SendPulse`, `ToggleOn`, `ToggleOff`; input mode
supports `CheckState` with state-change suppression to limit PC traffic.

| Item                | Value                                                                                |
|---------------------|--------------------------------------------------------------------------------------|
| Template params     | `kPin` (digital pin), `kOutput=true`, `kStartOn=false`                               |
| Parameters struct   | `pulse_duration: uint32_t = 10000 us`, `average_pool_size: uint8_t = 0`              |
| Custom event codes  | 51 `kInputOn`, 52 `kInputOff`, 53 `kInvalidPinMode`, 54 `kOutputOn`, 55 `kOutputOff` |
| Commands            | 1 `kSendPulse`, 2 `kToggleOn`, 3 `kToggleOff`, 4 `kCheckState`                       |
| Wrong-mode handling | Output commands on an input instance emit code 53 and abort                          |

**Wrapper**: `MesoscopeFrameTTLInterface(polling_frequency: int)`. Currently, exposes only the input-side surface
(`set_monitoring_state`, `pulse_count`) because the Mesoscope-VR consumer uses TTL only for receiving
mesoscope-frame trigger pulses. Output-side command codes are not exposed; if a consumer needs them, add
them as new instance attributes following the existing pattern. The wrapper hard-codes `name="mesoscope_frame"`
— rename it before reusing the wrapper for a non-Mesoscope-VR consumer.

- Shared memory: `<type>_<id>_pulse_tracker` — `np.uint64[1]` (cumulative pulse count)
- Error codes: `{53}` (invalid pin mode raises RuntimeError on the PC side)
- Public methods: `set_parameters(averaging_pool_size)`, `set_monitoring_state(*, state)`,
  `pulse_count` (property), `reset_pulse_count()`

### EncoderModule + EncoderInterface (type 2)

**Firmware**: `src/encoder_module.h` — `EncoderModule<kPinA, kPinB, kPinX, kInvertDirection>`. Wraps the
Paul Stoffregen `Encoder` library with `ENCODER_USE_INTERRUPTS` enabled for hardware-interrupt resolution.
Implements amortization on the non-reported direction to suppress micro-jitter into spurious events. The
`GetPPR` command runs a **blocking** 10-revolution measurement intended only for offline calibration.

| Item                 | Value                                                                                                                                      |
|----------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Template params      | `kPinA`, `kPinB`, `kPinX`, `kInvertDirection=false`                                                                                        |
| Parameters struct    | `report_ccw: bool = true`, `report_cw: bool = true`, `delta_threshold: uint32_t = 15`                                                      |
| Custom event codes   | 51 `kRotatedCCW`, 52 `kRotatedCW`, 53 `kPPR`                                                                                               |
| Commands             | 1 `kCheckState`, 2 `kReset`, 3 `kGetPPR` (BLOCKING, offline only)                                                                          |
| Interrupt dependency | `ENCODER_USE_INTERRUPTS` consumes interrupt slots; module is incompatible with other `AttachInterrupt()`-using libraries on the same board |

**Wrapper**: `EncoderInterface(encoder_ppr, wheel_diameter, polling_frequency)`. Computes
`cm_per_pulse` in `__init__` (rounded to 8 decimals) and maintains a 2-element shared-memory tracker
for total distance (cm, index 0) and the signed encoder displacement in pulses (index 1, from which the
absolute Unity position is derived). The centimeters-per-Unity-unit conversion is
NOT a constructor argument — it is supplied at experiment start via `set_unity_scale(cm_per_unity_unit)`
(the value is read from the active `TaskTemplate`), which derives `unity_unit_per_pulse`.

- Shared memory: `<type>_<id>_distance_tracker` — `np.float64[2]` (index 0: cumulative cm; index 1:
  signed encoder displacement in pulses, relative to runtime onset — converted to Unity units at read
  time by the `absolute_position` property)
- Public methods: `set_parameters(report_ccw, report_cw, delta_threshold)`, `set_unity_scale(cm_per_unity_unit)`,
  `set_monitoring_state(*, state)`, `cm_per_pulse` (property), `absolute_position` (property),
  `traveled_distance` (property), `reset_distance_tracker()`

### BrakeModule + BrakeInterface (type 3)

**Firmware**: `src/brake_module.h` — `BrakeModule<kPin, kNormallyEngaged, kStartEngaged>`. Drives a
PWM-capable pin to control an electromagnetic particle brake. Inverts the PWM duty cycle when the relay
is normally engaged so that strength 255 always means "fully engaged" from the PC's perspective.

| Item               | Value                                                                         |
|--------------------|-------------------------------------------------------------------------------|
| Template params    | `kPin` (must support `analogWrite`), `kNormallyEngaged`, `kStartEngaged=true` |
| Parameters struct  | `braking_strength: uint8_t = 128`, `pulse_duration: uint32_t = 1000000 us`    |
| Custom event codes | 51 `kEngaged`, 52 `kDisengaged`, 53 `kVariable`                               |
| Commands           | 1 `kToggleOn`, 2 `kToggleOff`, 3 `kSetBrakingPower`, 4 `kSendPulse`           |

**Wrapper**: `BrakeInterface(minimum_brake_strength, maximum_brake_strength)`. Calibration inputs are in
**grams centimeter** and are converted to **Newton centimeter** in `__init__` (hardcoded 0.00981 factor).
The wrapper currently exposes binary on/off and pulse semantics; PWM-strength control is not exposed at
the wrapper level (the underlying command code 3 exists in firmware).

- Shared memory: none
- Public methods: `set_state(*, state)`, `send_pulse(duration_ms)`, `maximum_brake_strength` (property),
  `minimum_brake_strength` (property)

### LickModule + LickInterface (type 4)

**Firmware**: `src/lick_module.h` — `LickModule<kPin>`. Analog-pin sensor using `INPUT_PULLDOWN`.
Emits `kChanged` only when the ADC delta exceeds `delta_threshold`, and emits a single zero-pull
trailer when signal drops back below `signal_threshold`. Assumes 12-bit ADC resolution
(`analogReadResolution(12)` set in `main.cpp`).

| Item               | Value                                                                                                   |
|--------------------|---------------------------------------------------------------------------------------------------------|
| Template params    | `kPin` (analog)                                                                                         |
| Parameters struct  | `signal_threshold: uint16_t = 300`, `delta_threshold: uint16_t = 300`, `average_pool_size: uint8_t = 0` |
| Custom event codes | 51 `kChanged`                                                                                           |
| Commands           | 1 `kCheckState`                                                                                         |

**Wrapper**: `LickInterface(lick_threshold, polling_frequency)`. Maintains a single-element
shared-memory counter incremented only on rising transitions above `lick_threshold` (one increment per
zero-cross-and-return cycle), which prevents the counter from inflating during sustained tongue
contact.

- Shared memory: `<type>_<id>_lick_tracker` — `np.uint64[1]` (cumulative lick count)
- Public methods: `set_parameters(signal_threshold, delta_threshold, average_pool_size)`,
  `set_monitoring_state(*, state)`, `lick_count` (property), `lick_threshold` (property)

### ValveModule + WaterValveInterface + GasPuffValveInterface (type 5, ids 1 and 2)

**Firmware**: `src/valve_module.h` —
`ValveModule<kValvePin, kNormallyClosed, kStartClosed, kTonePin=255, kNormallyOff=true, kStartOff=true>`.
Drives a solenoid valve with an optional co-driven tone buzzer. When `kTonePin == 255` the tone subsystem
is disabled at runtime (`tone_duration` is forced to 0 so the tone branches are skipped). Supports
per-pulse tone-extension if `tone_duration > pulse_duration`.
The `Calibrate` command is **blocking** (delayMicroseconds-based burst) intended only for offline
calibration.

| Item               | Value                                                                                                                                                               |
|--------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Template params    | `kValvePin`, `kNormallyClosed`, `kStartClosed=true`, `kTonePin=255`, `kNormallyOff=true`, `kStartOff=true`                                                          |
| Parameters struct  | `pulse_duration: uint32_t = 35000 us`, `calibration_count: uint16_t = 500`, `tone_duration: uint32_t = 300000 us`                                                   |
| Custom event codes | 51 `kOpen`, 52 `kClosed`, 53 `kCalibrated`, 54 `kToneOn`, 55 `kToneOff`, 56 `kInvalidToneConfiguration`                                                             |
| Commands           | 1 `kSendPulse`, 2 `kToggleOn`, 3 `kToggleOff`, 4 `kCalibrate` (BLOCKING, offline only), 5 `kTonePulse`                                                              |
| Safety bound       | Pulse durations longer than ~400 ms trigger the keepalive watchdog on the host PC; the wrappers cap requested durations to `_MAXIMUM_VALVE_PULSE_DURATION_MS = 400` |

**Wrapper A — `WaterValveInterface(valve_calibration_data)`**: water-reward solenoid with audible tone. Fits a
power-law model (`a * pulse_duration ** b`) to the supplied calibration tuple in `__init__` using
`scipy.optimize.curve_fit` and inverts it to map requested volumes to pulse durations. Uses a
`PrecisionTimer` (initialized in `initialize_remote_assets` because it is non-picklable) to integrate
delivered volume across open/close transitions reported by the firmware.

- Shared memory: `<type>_<id>_valve_tracker` — `np.float64[3]` (index 0: cumulative volume in uL;
  index 1: calibration state — 0 calibrating, 1 calibrated; index 2: instantaneous valve state — 0 closed, 1 open)
- Public methods: `set_state(*, state)`, `deliver_reward(volume, tone_duration)`,
  `simulate_reward(tone_duration)`, `reference_valve()`, `calibrate_valve(pulse_duration)`,
  `get_duration_from_volume(target_volume)`, `scale_coefficient` (property),
  `nonlinearity_exponent` (property), `delivered_volume` (property), `calibrating` (property)

**Wrapper B — `GasPuffValveInterface()`**: same firmware module, different application. No calibration
(gas-volume precision is not critical); duration-only control. Hardcodes `name="gas_puff"` and `module_id=2`.

- Shared memory: `<type>_<id>_puff_tracker` — `np.uint32[2]` (index 0: cumulative puff count; index 1:
  instantaneous valve state)
- Public methods: `set_state(*, state)`, `deliver_puff(duration_ms)`, `puff_count` (property)

> **Multi-instance pattern**: Two valve wrappers exist because the same firmware module
> serves two application roles (water reward + gas aversive). When a single firmware module can be
> physically reused with different calibration or semantics, add a new wrapper class with the same
> `module_type` and a fresh `module_id` rather than creating a new firmware module.

### TorqueModule + TorqueInterface (type 6)

**Firmware**: `src/torque_module.h` — `TorqueModule<kPin, kBaseline, kInvertDirection>`. Reads an
AD620-amplified analog signal; signals above `kBaseline` encode CCW torque, signals below encode CW.
Implements the same delta-threshold + zero-pull trailer pattern as `LickModule`.

| Item               | Value                                                                                                                                                       |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Template params    | `kPin` (analog), `kBaseline` (ADC units = 0 torque), `kInvertDirection=false`                                                                               |
| Parameters struct  | `report_ccw: bool = true`, `report_cw: bool = true`, `signal_threshold: uint16_t = 100`, `delta_threshold: uint16_t = 70`, `average_pool_size: uint8_t = 5` |
| Custom event codes | 51 `kCCWTorque`, 52 `kCWTorque`                                                                                                                             |
| Commands           | 1 `kCheckState`                                                                                                                                             |

**Wrapper**: `TorqueInterface(baseline_voltage, maximum_voltage, sensor_capacity, polling_frequency)`.
Computes `torque_per_adc_unit` in `__init__` using sensor capacity in g·cm converted to N·cm
(hardcoded 0.00981). Currently, does not maintain shared memory or process incoming data online — the
wrapper exists primarily to expose calibration math to consumers.

- Shared memory: none (data is preserved in the log archive via DataLogger only)
- Public methods: `set_parameters(report_ccw, report_cw, signal_threshold, delta_threshold, averaging_pool_size)`,
  `set_monitoring_state(*, state)`, `torque_per_adc_unit` (property)

### ScreenModule + ScreenInterface (type 7)

**Firmware**: `src/screen_module.h` — `ScreenModule<kPin, kNormallyClosed>`. A degenerate
`SendPulse`-like module whose single command pulses a FET gate that shorts the screen power board's
button terminals, simulating a physical button press. slmc currently provides only `kToggle`; no
direct on/off semantics — the consumer tracks software-side state.

| Item               | Value                                   |
|--------------------|-----------------------------------------|
| Template params    | `kPin`, `kNormallyClosed=false`         |
| Parameters struct  | `pulse_duration: uint32_t = 1000000 us` |
| Custom event codes | 51 `kOn`, 52 `kOff`                     |
| Commands           | 1 `kToggle`                             |

**Wrapper**: `ScreenInterface()`. Stateful: tracks `_enabled` and only issues a toggle command when the
requested state differs from the cached state. The wrapper trusts the consumer to know the initial
hardware state (defaults to OFF on initialization).

- Shared memory: none
- Public methods: `set_parameters(pulse_duration)`, `set_state(*, state)`, `state` (property)
