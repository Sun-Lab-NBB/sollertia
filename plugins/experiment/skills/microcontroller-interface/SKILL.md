---
name: sollertia-microcontroller-interface
description: >-
  Documents how Sollertia experiment binding classes wrap ataraxis-communication-interface
  MicroControllerInterface and ModuleInterface instances for the Mesoscope-VR system. Covers
  MesoscopeMicroControllers configuration, actor/sensor/encoder role assignment, and
  binding-class lifecycle. Use when adding or modifying microcontroller modules; delegates
  low-level API and firmware work to the ataraxis communication skills.
user-invocable: true
---

# Sollertia Microcontroller Interface

Documents how Sollertia experiment binding classes wrap `ataraxis-communication-interface` and
`ataraxis-micro-controller` for the Mesoscope-VR acquisition system. Focuses exclusively on the Sollertia
data class semantics and binding patterns.

For low-level firmware development and PC interface API, this skill delegates to the `ataraxis@communication`
and `ataraxis@microcontroller` plugin skills.

---

## Scope

**Covers:**
- The `MesoscopeMicroControllers` configuration dataclass and its fields
- Controller role assignment (actor / sensor / encoder) on the Mesoscope-VR system
- Where microcontroller configuration lives in the Sollertia configuration tree
- Calibration data semantics (valve calibration, brake torque, encoder PPR, etc.)

**Does not cover** (delegated to ataraxis):
- MicroControllerInterface, ModuleInterface, or MQTTCommunication API
- Custom hardware Module subclass implementation in C++ firmware
- Microcontroller discovery, manifest management, MQTT broker setup
- Log archive assembly and message extraction

---

## Required ataraxis skill handoffs

You MUST read the relevant ataraxis skill before writing or modifying any microcontroller code. The
Sollertia binding layer is intentionally thin — all hardware interaction goes through
ataraxis-communication-interface and ataraxis-micro-controller.

| Task                                          | Required ataraxis skill                                      |
|-----------------------------------------------|--------------------------------------------------------------|
| Hardware discovery / MQTT verification        | `ataraxis@communication:microcontroller-setup`               |
| Writing PC-side MicroControllerInterface code | `ataraxis@communication:microcontroller-interface`           |
| Implementing a new C++ firmware Module        | `ataraxis@microcontroller:firmware-module`                   |
| Diagnosing MCP server issues                  | `ataraxis@communication:communication-mcp-environment-setup` |
| Log archive layout / extraction config        | `ataraxis@communication:log-input-format`                    |

---

## Sollertia binding semantics

### MesoscopeMicroControllers configuration dataclass

Defined in `sollertia_shared_assets.configuration.mesoscope_configuration.MesoscopeMicroControllers`, this
dataclass captures the full per-controller configuration for the Mesoscope-VR system. It is loaded from
disk as part of `MesoscopeSystemConfiguration` and passed to the binding class at runtime.

The dataclass groups its fields into three semantic blocks:

**Port assignment (Sollertia → ataraxis-communication-interface):**

| Field                   | Role    | Notes                                                                                       |
|-------------------------|---------|---------------------------------------------------------------------------------------------|
| `actor_port`            | ACTOR   | Microcontroller that drives output hardware (valves, brakes, screen triggers)               |
| `sensor_port`           | SENSOR  | Microcontroller that reads behavioral sensors (lick, torque, mesoscope frame TTL)           |
| `encoder_port`          | ENCODER | Microcontroller dedicated to the running wheel quadrature encoder                           |
| `keepalive_interval_ms` | —       | Heartbeat interval used by the binding class to verify all three controllers are responsive |

The Mesoscope-VR system requires **exactly three** microcontrollers in these specific roles. Adding or
removing controllers requires changes to both the dataclass and the binding class, plus a coordinated
firmware update.

**Calibration data (per-module physical parameters):**

These fields parameterize specific firmware Modules running on the three controllers. They are passed
verbatim to the relevant `ModuleInterface` constructors at runtime.

| Field group                           | Module role                    | Controller               |
|---------------------------------------|--------------------------------|--------------------------|
| `*_brake_strength_g_cm`               | Wheel brake torque limits      | ACTOR                    |
| `lick_*_adc`                          | Lick sensor thresholds         | SENSOR                   |
| `torque_*`                            | Wheel torque sensor            | SENSOR                   |
| `wheel_encoder_*`                     | Quadrature encoder             | ENCODER                  |
| `wheel_diameter_cm`                   | Encoder unit conversion        | ENCODER                  |
| `cm_per_unity_unit`                   | VR ↔ real-world distance scale | derived in binding class |
| `valve_calibration_data`              | Water valve dispense lookup    | ACTOR                    |
| `screen_trigger_pulse_duration_ms`    | VR screen TTL                  | ACTOR                    |
| `sensor_polling_delay_ms`             | Generic sensor polling rate    | SENSOR                   |
| `mesoscope_frame_averaging_pool_size` | Mesoscope TTL ingestion        | SENSOR                   |

These values **must** match what the firmware expects. Whenever the firmware version on
`sollertia-micro-controllers` changes, validate that all calibration fields are still consumed in the
same units and have not been renamed. See `ataraxis@microcontroller:firmware-module` for the firmware
parameter structure layout.

**Note:** ACTOR runs two `ValveModule` instances (`reward_valve` and `gas_puff_valve`) for water reward
and aversive air-puff delivery respectively. The `valve_calibration_data` calibration table applies to
the reward valve only; the gas puff valve uses the firmware's default pulse parameters or per-trial
configuration sent at runtime.

### Binding class lifecycle

The Sollertia binding wraps the three MicroControllerInterface instances inside `MesoscopeVRSystem`.
The binding is responsible for:

1. **Construction order.** Build all three `MicroControllerInterface` instances before constructing any
   `ModuleInterface`. The DataLogger and MQTTCommunication must exist before any controller.
2. **Module attachment.** Each controller receives its specific set of `ModuleInterface` subclasses
   instantiated from the calibration fields above. The actor/sensor/encoder split is hard-coded — do
   not move modules between controllers without a firmware change.
3. **Keepalive monitoring.** The binding registers a periodic check at `keepalive_interval_ms` and aborts
   the session if any controller misses its heartbeat.
4. **Coordinated start/stop.** All three controllers start in a fixed order and stop in reverse order
   to ensure pending DataLogger writes are flushed.

For the canonical PC-side implementation pattern, see `ataraxis@communication:microcontroller-interface`.

---

## Adding a new module to Mesoscope-VR

1. **Implement the firmware Module** following `ataraxis@microcontroller:firmware-module`. Decide which
   of the three controllers (actor/sensor/encoder) it belongs on based on whether it produces output or
   reads input.
2. **Add calibration fields** to `MesoscopeMicroControllers` for any physical parameters the module needs.
   Group them with the module's other fields and follow the existing naming convention
   (`<module>_<parameter>_<unit>`).
3. **Build the firmware** for the affected controller and flash it to the Teensy. Bump
   `sollertia-micro-controllers`.
4. **Add a `ModuleInterface` subclass** in the sollertia-experiment binding layer following
   `ataraxis@communication:microcontroller-interface`. Wire it to the new dataclass fields.
5. **Update the binding class** to instantiate the new ModuleInterface on the correct controller and
   register it in the start/stop lifecycle.
6. **Hand off to the assets plugin's `/system-configuration` skill** to regenerate the host machine's
   system configuration YAML against the new schema. This skill must not call `write_system_configuration_tool`
   or invoke `slsa` directly — system configuration writes are owned by `/system-configuration`.
   The hand-off responsibility includes bumping the `sollertia-shared-assets` version pin so older configurations
   no longer load against the new schema.
7. **Verify** the new module appears in the controller manifest using
   `ataraxis@communication:microcontroller-setup` MCP tools.

---

## Verification checklist

```text
- [ ] New calibration fields follow the existing <module>_<parameter>_<unit> naming convention
- [ ] Field placement (actor/sensor/encoder) matches the firmware Module's controller assignment
- [ ] sollertia-micro-controllers version bumped after firmware change
- [ ] Handed off to /system-configuration to regenerate the host machine's YAML and bump sollertia-shared-assets
- [ ] Binding class lifecycle preserves DataLogger → MQTT → Controller → Module ordering
- [ ] Read ataraxis@microcontroller:firmware-module for C++ Module structure
- [ ] Read ataraxis@communication:microcontroller-interface for PC-side patterns
- [ ] Read ataraxis@communication:microcontroller-setup for hardware verification
```
