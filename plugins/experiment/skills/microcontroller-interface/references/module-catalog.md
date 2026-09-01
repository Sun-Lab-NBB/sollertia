# Module catalog

State snapshot of the currently-deployed paired Modules + Interfaces in the Sollertia microcontroller stack:

- **sollertia-micro-controllers** firmware: C++ `Module` subclasses in `src/*.h`
- **sollertia-experiment** Python: `ModuleInterface` subclasses in
  `src/sollertia_experiment/cross_system/module_interfaces.py`

This file documents **what is currently deployed**, not the rules for extending the stack. See
[`../SKILL.md`](../SKILL.md) for the conventions, cross-side contract, allocation rules, and workflows that govern this
layer. Update this file whenever a module is added, removed, or modified (see SKILL.md's "Maintenance contract"
section).

---

## Type-code registry

The Sollertia platform currently uses type codes 1-7.

| Type | Module          | Interface                                                    | Direction       | Instance ids in use | Notes                                                                 |
|------|-----------------|--------------------------------------------------------------|-----------------|---------------------|-----------------------------------------------------------------------|
| 1    | `TTLModule`     | `MesoscopeFrameTTLInterface`                                 | Input OR Output | 1                   | Per-instance pin mode is set at compile time via `kOutput` template   |
| 2    | `EncoderModule` | `EncoderInterface`                                           | Input           | 1                   | Uses `ENCODER_USE_INTERRUPTS`, incompatible with other interrupt libs |
| 3    | `BrakeModule`   | `BrakeInterface`                                             | Output          | 1                   | PWM-controlled electromagnetic particle brake                         |
| 4    | `LickModule`    | `LickInterface`                                              | Input           | 1                   | Analog ADC-threshold conductive sensor                                |
| 5    | `ValveModule`   | `WaterValveInterface` (id 1), `GasPuffValveInterface` (id 2) | Output          | 1, 2                | Same firmware module, two Python wrappers with different calibration  |
| 6    | `TorqueModule`  | `TorqueInterface`                                            | Input           | 1                   | AD620-amplified analog torque sensor                                  |
| 7    | `ScreenModule`  | `ScreenInterface`                                            | Output          | 1                   | Pulses FET gates on VR-screen power boards                            |

**Next unused code:** 8. The "Extending the Library" section of `slmc/README.md` carries the same allocation, and the
two move together, because the firmware repository owns the codes its targets instantiate.

Nothing in slmc or sle enforces the allocation at build time. The ataraxis `Kernel::ResolveTargetModule` scans
`modules[]` and returns the first entry whose type and id match (`ataraxis-micro-controller/src/kernel.h:683`), so on
the controller a duplicated pair leaves the later module permanently unaddressable, and the Kernel declares no status
code for the condition, its only miss code being `kTargetModuleNotFound`. The host catches it instead.
`MicroControllerInterface.__init__` raises `ValueError` when two `ModuleInterface` instances on one controller share a
combined type + id code (`interface.py:663-672`), and `_verify_microcontroller_communication`, run when `start()`
launches the communication process, raises `ValueError` when the board's own module-identification responses contain a
duplicated type + id pair (`interface.py:973-979`). The collision therefore surfaces as a startup abort rather than as
silently missing data. Confirm a candidate pair against the table above and against every target block of
`slmc/src/main.cpp` before instantiating it.

Type 5 is the only type with two instance ids in the current slmc deployment. Its `ACTOR` target instantiates
`reward_valve` at `(5, 1)` and `gas_puff_valve` at `(5, 2)` (`slmc/src/main.cpp`), so it is the one worked
example of the multi-`module_id` pattern.

A firmware `Module` is generic and reusable, and its `ModuleInterface` wrapper may specialize it for one equipment role
through its name, its calibration, and the data it processes. `MesoscopeFrameTTLInterface` is an input-only role
specialization of the generic firmware `TTLModule`, and the `WaterValveInterface` / `GasPuffValveInterface` split over a
single `ValveModule` is the second example. A role-specialized interface still lives in the shared `cross_system` layer
and stays reusable by any system that drives the same hardware.

---

## Hardware-surface catalog

Each block below documents one firmware `Module` and the interface(s) that wrap it as a contract unit, because a
firmware type may have more than one interface (`ValveModule` has two). The firmware section names the C++ class,
template parameters, custom event codes, and commands. The Python section(s) name the interface class, constructor
calibration knobs, shared-memory state surfaced to other processes, and the public methods exposed to consumers.

**Reading the "Identity" bullet.** It reproduces the five values that a wrapper hardcodes in its `super().__init__`
call. `module_type` and `module_id` must match the firmware constructor arguments in `slmc/src/main.cpp`, `data_codes`
and `error_codes` must draw from the firmware `kCustomStatusCodes` values listed in the same block, and `name` labels
the interface in the log archive. Citations are relative to `src/sollertia_experiment/cross_system/`.

**Reading the "Boot defaults" row.** It gives the `CustomRuntimeParameters` field order, C++ types, and the values
`SetupModule()` assigns, which are the firmware's own named `static constexpr` members, `kDefault<Field>` for a plain
default and a bound-describing name for a hardware limit (`BrakeModule`'s `kFullEngageDuty`). During a session a module
runs on the values the host-PC binding class pushes through `set_parameters()` at session start, which overwrite the
whole struct. The boot defaults therefore govern exactly two windows: the span before the first `set_parameters()` call,
and the span after a controller reset or keepalive timeout makes the Kernel re-run `SetupModule()`. Field order and
types are the load-bearing half of the row, because the wrapper's `send_parameters()` tuple must match them exactly.

### TTLModule + MesoscopeFrameTTLInterface (type 1)

**Firmware**: `src/ttl_module.h`, `TTLModule<kPin, kOutput, kStartOn>`. A single instance is either an output or an
input, not both. Output mode supports `SendPulse`, `ToggleOn`, and `ToggleOff`. Input mode supports `CheckState` with
state-change suppression to limit PC traffic.

| Item                | Value                                                                                |
|---------------------|--------------------------------------------------------------------------------------|
| Template params     | `kPin` (digital pin), `kOutput=true`, `kStartOn=false`                               |
| Boot defaults       | `pulse_duration: uint32_t = 10000 us`, `average_pool_size: uint8_t = 0`              |
| Custom event codes  | 51 `kInputOn`, 52 `kInputOff`, 53 `kInvalidPinMode`, 54 `kOutputOn`, 55 `kOutputOff` |
| Commands            | 1 `kSendPulse`, 2 `kToggleOn`, 3 `kToggleOff`, 4 `kCheckState`                       |
| Wrong-mode handling | Output commands on an input instance and CheckState on an output emit 53 and abort   |

**Wrapper**: `MesoscopeFrameTTLInterface(polling_frequency: int)`. An input-only role specialization of the generic
firmware `TTLModule`. It caches `_check_state = 4` alone, so the three output command codes stay unbound, and a consumer
that needs them adds them as new instance attributes following the existing pattern. The equipment this wrapper is named
for belongs to the current worked example, see `mesoscope:mesoscope-vr`.

- Identity: `module_type=1`, `module_id=1`, `name="mesoscope_frame"`, `data_codes={51}`, `error_codes={53: ...}`
  (`MesoscopeFrameTTLInterface.__init__` in `module_interfaces.py`)
- Shared memory: `<type>_<id>_pulse_tracker`, `np.uint64[1]` (cumulative pulse count)
- Error code 53 `kInvalidPinMode` raises `RuntimeError` and aborts the runtime
- Public methods: `set_parameters(averaging_pool_size)`, `set_monitoring_state(*, state)`, `pulse_count` (property),
  `reset_pulse_count()`

### EncoderModule + EncoderInterface (type 2)

**Firmware**: `src/encoder_module.h`, `EncoderModule<kPinA, kPinB, kPinX, kInvertDirection>`. Wraps the Paul Stoffregen
`Encoder` library with `ENCODER_USE_INTERRUPTS` enabled for hardware-interrupt resolution. Implements amortization on
the non-reported direction to suppress micro-jitter into spurious events. The `GetPPR` command runs a **blocking**
10-revolution measurement intended only for offline calibration.

| Item                 | Value                                                                                                                                             |
|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| Template params      | `kPinA`, `kPinB`, `kPinX`, `kInvertDirection=false`                                                                                               |
| Boot defaults        | `report_ccw: bool = true`, `report_cw: bool = false`, `delta_threshold: uint32_t = 15`                                                            |
| Custom event codes   | 51 `kRotatedCCW`, 52 `kRotatedCW`, 53 `kPPR`                                                                                                      |
| Commands             | 1 `kCheckState`, 2 `kReset`, 3 `kGetPPR` (BLOCKING, offline only)                                                                                 |
| Interrupt dependency | `ENCODER_USE_INTERRUPTS` consumes interrupt slots, so the module is incompatible with other `AttachInterrupt()`-using libraries on the same board |

**Wrapper**: `EncoderInterface(encoder_ppr, wheel_diameter, polling_frequency)`. Computes `cm_per_pulse` in `__init__`
as a full-precision `np.float64` and maintains a 2-element shared-memory tracker for total distance (cm, index 0) and
the signed encoder displacement in pulses (index 1, from which the absolute Unity position is derived). The
centimeters-per-Unity-unit conversion is NOT a constructor argument. It is supplied at experiment start via
`set_unity_scale(cm_per_unity_unit)` (the value is read from the active `TaskTemplate`), which derives
`_unity_unit_per_pulse`.

- Identity: `module_type=2`, `module_id=1`, `name="encoder"`, `data_codes={51, 52}`, `error_codes=None`
  (`EncoderInterface.__init__` in `module_interfaces.py`)
- Shared memory: `<type>_<id>_distance_tracker`, `np.float64[2]`. Index 0 holds the cumulative cm. Index 1 holds the
  signed encoder displacement in pulses relative to runtime onset, converted to Unity units at read time by the
  `absolute_position` property.
- Public methods: `set_parameters(report_ccw, report_cw, delta_threshold)`, `set_unity_scale(cm_per_unity_unit)`,
  `set_monitoring_state(*, state)`, `cm_per_pulse` (property), `absolute_position` (property), `traveled_distance`
  (property), `reset_distance_tracker()`

### BrakeModule + BrakeInterface (type 3)

**Firmware**: `src/brake_module.h`, `BrakeModule<kPin, kNormallyEngaged, kStartEngaged>`. Drives a PWM-capable pin to
control an electromagnetic particle brake. Inverts the PWM duty cycle when the relay is normally engaged so that
strength 255 always means "fully engaged" from the PC's perspective. `kSetBrakingPower` reports `kEngaged` or
`kDisengaged` at the two duty-cycle extremes and `kVariable` only for an intermediate strength, because the extremes are
driven as digital levels rather than as a PWM waveform. The module tracks which peripheral owns the pin and routes
every command-path digital write through a helper that writes the level first and then reclaims GPIO control from
the PWM peripheral. That reclaim is needed because `analogWrite()` re-points the pin and the two peripherals use
separate registers.

| Item               | Value                                                                                  |
|--------------------|----------------------------------------------------------------------------------------|
| Template params    | `kPin` (must support `analogWrite`), `kNormallyEngaged`, `kStartEngaged=true`          |
| Boot defaults      | `braking_strength: uint8_t = kFullEngageDuty`, `pulse_duration: uint32_t = 1000000 us` |
| Custom event codes | 51 `kEngaged`, 52 `kDisengaged`, 53 `kVariable`                                        |
| Commands           | 1 `kToggleOn`, 2 `kToggleOff`, 3 `kSetBrakingPower`, 4 `kSendPulse`                    |

`kFullEngageDuty` is `0` when `kNormallyEngaged` is true and `255` otherwise. It is expressed in the inverted frame
`SetCustomParameters()` stores, so the boot default engages the brake at full strength while leaving the pin on the GPIO
peripheral.

**Wrapper**: `BrakeInterface(minimum_brake_strength, maximum_brake_strength)`. Calibration inputs are in **grams
centimeter** and are converted to **Newton centimeter** in `__init__` (hardcoded 0.00981 factor). The wrapper currently
exposes binary on/off and pulse semantics. PWM-strength control stays unexposed at the wrapper level, though the
underlying command code 3 exists in firmware.

- Identity: `module_type=3`, `module_id=1`, `name="brake"`, `data_codes=None`, `error_codes=None`
  (`BrakeInterface.__init__` in `module_interfaces.py`)
- Shared memory: none
- Cached commands: `_engage = 1`, `_disengage = 2`, `_pulse = 4` (`BrakeInterface.__init__` in
  `module_interfaces.py`)
- Public methods: `set_state(*, state)`, `send_pulse(duration_ms)`, `maximum_brake_strength` (property),
  `minimum_brake_strength` (property)

### LickModule + LickInterface (type 4)

**Firmware**: `src/lick_module.h`, `LickModule<kPin>`. Analog-pin sensor using `INPUT_PULLDOWN`. Emits `kChanged` only
when the ADC delta exceeds `delta_threshold`, and emits a single zero-pull trailer when signal drops back below
`signal_threshold`. Assumes 12-bit ADC resolution (`kAnalogReadResolution`, passed to
`analogReadResolution()` in `main.cpp`).

| Item               | Value                                                                                                   |
|--------------------|---------------------------------------------------------------------------------------------------------|
| Template params    | `kPin` (analog)                                                                                         |
| Boot defaults      | `signal_threshold: uint16_t = 300`, `delta_threshold: uint16_t = 300`, `average_pool_size: uint8_t = 2` |
| Custom event codes | 51 `kChanged`                                                                                           |
| Commands           | 1 `kCheckState`                                                                                         |

**Wrapper**: `LickInterface(lick_threshold, polling_frequency)`. Maintains a single-element shared-memory counter
incremented only on rising transitions above `lick_threshold` (one increment per zero-cross-and-return cycle), which
prevents the counter from inflating during sustained tongue contact.

- Identity: `module_type=4`, `module_id=1`, `name="lick"`, `data_codes={51}`, `error_codes=None`
  (`LickInterface.__init__` in `module_interfaces.py`)
- Shared memory: `<type>_<id>_lick_tracker`, `np.uint64[1]` (cumulative lick count)
- Public methods: `set_parameters(signal_threshold, delta_threshold, average_pool_size)`,
  `set_monitoring_state(*, state)`, `lick_count` (property), `lick_threshold` (property)

### ValveModule + WaterValveInterface + GasPuffValveInterface (type 5, ids 1 and 2)

**Firmware**: `src/valve_module.h`, `ValveModule<kValvePin, kNormallyClosed, kStartClosed, kTonePin=255,
kNormallyOff=true, kStartOff=true>`. Drives a solenoid valve with an optional co-driven tone buzzer. When `kTonePin ==
255` the `kToneEnabled` constant elides the tone-pin configuration in `SetupModule()` and the tone branches in
`Pulse()`, and forces `tone_duration` to 0. `SetupModule()` still reports 55 `kToneOff` in that case, so the host sees a
consistent initial state either way. `Tone()` (command 5) is not elided: it aborts at its first stage on the runtime
`tone_duration == 0` check and reports 56 `kInvalidToneConfiguration`, which is why that code exists. A pulse that
energized the buzzer always reaches the silencing stage, so the tone is never left sounding. A tone longer than the
valve pulse extends past it by the difference, and a shorter one still sounds for the full pulse duration. The
`Calibrate` command is **blocking** (delayMicroseconds-based burst) intended only for offline calibration.

| Item               | Value                                                                                                                                                       |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Template params    | `kValvePin`, `kNormallyClosed`, `kStartClosed=true`, `kTonePin=255`, `kNormallyOff=true`, `kStartOff=true`                                                  |
| Boot defaults      | `pulse_duration: uint32_t = 39410 us`, `calibration_count: uint16_t = 200`, `tone_duration: uint32_t = 300000 us`                                           |
| Custom event codes | 51 `kOpen`, 52 `kClosed`, 53 `kCalibrated`, 54 `kToneOn`, 55 `kToneOff`, 56 `kInvalidToneConfiguration`                                                     |
| Commands           | 1 `kSendPulse`, 2 `kToggleOn`, 3 `kToggleOff`, 4 `kCalibrate` (BLOCKING, offline only), 5 `kTonePulse`                                                      |
| Safety bound       | Both wrappers cap a requested pulse at `_MAXIMUM_VALVE_PULSE_DURATION_MS = 400`, held below the 500 ms firmware keepalive interval (`module_interfaces.py`) |

**Wrapper A**: `WaterValveInterface(valve_calibration_data)`, the water-reward solenoid with an audible tone. Fits a
power-law model (`a * pulse_duration ** b`) to the supplied calibration tuple in `__init__` using
`scipy.optimize.curve_fit` and inverts it to map requested volumes to pulse durations. Uses a `PrecisionTimer`
(initialized in `initialize_remote_assets` because it is non-picklable) to integrate delivered volume across open/close
transitions reported by the firmware.

- Identity: `module_type=5`, `module_id=1`, `name="valve"`, `data_codes={51, 52, 53}`, `error_codes={56: ...}`
  (`WaterValveInterface.__init__` in `module_interfaces.py`)
- Shared memory: `<type>_<id>_valve_tracker`, `np.float64[3]`. Index 0 holds the cumulative volume in uL. Index 1 holds
  the calibration state, 0 calibrating and 1 calibrated. Index 2 holds the instantaneous valve state, 0 closed and 1
  open.
- Public methods: `set_state(*, state)`, `deliver_reward(volume, tone_duration)`, `simulate_reward(tone_duration)`,
  `reference_valve()`, `calibrate_valve(pulse_duration)`, `get_duration_from_volume(target_volume)`, `scale_coefficient`
  (property), `nonlinearity_exponent` (property), `delivered_volume` (property), `calibrating` (property)

**Wrapper B**: `GasPuffValveInterface()`, the same firmware module in a different application. Calibration is omitted
because gas-volume precision is not critical, leaving duration-only control. Hardcodes `name="gas_puff"` and
`module_id=2`.

- Identity: `module_type=5`, `module_id=2`, `name="gas_puff"`, `data_codes={51, 52}`, `error_codes=None`
  (`GasPuffValveInterface.__init__` in `module_interfaces.py`)
- Shared memory: `<type>_<id>_puff_tracker`, `np.uint32[2]` (index 0 cumulative puff count, index 1 instantaneous valve
  state)
- Cached commands: `_pulse = 1`, `_open = 2`, `_close = 3` (`GasPuffValveInterface.__init__` in
  `module_interfaces.py`)
- Public methods: `set_state(*, state)`, `deliver_puff(duration_ms)`, `puff_count` (property)

> **Multi-instance pattern**: Two valve wrappers exist because the same firmware module serves two application roles
> (water reward + gas aversive). When a single firmware module can be physically reused with different calibration or
> semantics, add a new wrapper class with the same `module_type` and a fresh `module_id` rather than creating a new
> firmware module.

### TorqueModule + TorqueInterface (type 6)

**Firmware**: `src/torque_module.h`, `TorqueModule<kPin, kBaseline, kInvertDirection>`. Reads an AD620-amplified analog
signal. Signals above `kBaseline` encode CCW torque, and signals below encode CW. Implements the same delta-threshold +
zero-pull trailer pattern as `LickModule`.

| Item               | Value                                                                                                                                                        |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Template params    | `kPin` (analog), `kBaseline` (ADC units = 0 torque), `kInvertDirection=false`                                                                                |
| Boot defaults      | `report_ccw: bool = true`, `report_cw: bool = true`, `signal_threshold: uint16_t = 150`, `delta_threshold: uint16_t = 100`, `average_pool_size: uint8_t = 4` |
| Custom event codes | 51 `kCCWTorque`, 52 `kCWTorque`                                                                                                                              |
| Commands           | 1 `kCheckState`                                                                                                                                              |

**Wrapper**: `TorqueInterface(baseline_voltage, maximum_voltage, sensor_capacity, polling_frequency)`. Computes
`torque_per_adc_unit` in `__init__` using sensor capacity in g·cm converted to N·cm (hardcoded 0.00981). Currently
maintains no shared memory and processes no incoming data online, because the wrapper exists primarily to expose
calibration math to consumers.

- Identity: `module_type=6`, `module_id=1`, `name="torque"`, `data_codes=None`, `error_codes=None`
  (`TorqueInterface.__init__` in `module_interfaces.py`)
- Shared memory: none, so torque data reaches disk through the base interface's automatic logging alone
- Public methods: `set_parameters(report_ccw, report_cw, signal_threshold, delta_threshold, averaging_pool_size)`,
  `set_monitoring_state(*, state)`, `torque_per_adc_unit` (property)

### ScreenModule + ScreenInterface (type 7)

**Firmware**: `src/screen_module.h`, `ScreenModule<kPin, kNormallyClosed>`. A degenerate `SendPulse`-like module whose
single command pulses a FET gate that shorts the screen power board's button terminals, simulating a physical button
press. slmc currently provides only `kToggle`, so the consumer tracks software-side state.

| Item               | Value                                  |
|--------------------|----------------------------------------|
| Template params    | `kPin`, `kNormallyClosed=false`        |
| Boot defaults      | `pulse_duration: uint32_t = 500000 us` |
| Custom event codes | 51 `kOn`, 52 `kOff`                    |
| Commands           | 1 `kToggle`                            |

**Wrapper**: `ScreenInterface()`. Stateful, tracking `_enabled` and issuing a toggle command only when the requested
state differs from the cached state. The wrapper trusts the consumer to know the initial hardware state (defaults to OFF
on initialization).

- Identity: `module_type=7`, `module_id=1`, `name="screen"`, `data_codes=None`, `error_codes=None`
  (`ScreenInterface.__init__` in `module_interfaces.py`)
- Shared memory: none
- Public methods: `set_parameters(pulse_duration)`, `set_state(*, state)`, `state` (property)
