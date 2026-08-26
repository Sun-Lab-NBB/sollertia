---
name: mesoscope-vr-module-parsing
description: >-
  Documents Mesoscope-VR's concrete instance of the per-system module-parser contract: the eight-entry
  module registry keyed by (module_type, module_id), each parser's output feather filename, event codes,
  MesoscopeHardwareState eligibility fields and usage flags, output column schema, and unit conversions. Use
  when interpreting or modifying a Mesoscope-VR microcontroller feather output, adding a new module parser, or
  mapping a hardware module to its calibrated processed output.
user-invocable: false
---

# Mesoscope-VR module parsing

Documents how the Mesoscope-VR processing pipeline converts pre-extracted microcontroller module feather
files into calibrated, domain-specific behavior feather files, one parser per hardware module.

This skill owns the concrete Mesoscope-VR instance of the module-parser contract. It is the single source
for the `(module_type, module_id)` to conversion mapping: which parser runs, what output filename and column
schema it produces, which event codes it reads, and which `MesoscopeHardwareState` fields gate it. It
concretizes the system-agnostic feather-parsing primitives owned by `forging:microcontroller-primitives`, and
never re-documents those primitives, only how the Mesoscope-VR registry consumes them.

All of the symbols below live in `sollertia_forgery.mesoscope_vr.microcontrollers`. The eligibility registry
(`_MODULE_REGISTRY`), the eight `_parse_*_data` functions, and the `_resolve_hardware_state` and `_is_module_eligible`
helpers are module-private. The public surface is the eight `parse_<module>` entry points plus the
`get_eligible_modules` and `get_module_event_codes` accessors, all of which the agnostic pipeline reaches through the
registries in `sollertia_forgery.registries`. The batch orchestration that invokes them is owned by
`forging:behavior-processing`.

---

## Scope

**Covers:**
- The eight-parser module registry dispatch on `(module_type, module_id)`
- Per-module output filename, event codes, hardware-state eligibility fields, usage flags, and output column schema
- The two-tuple eligibility contract that decides whether a module is parsed for a given session
- Encoder cumulative-displacement conversion and TTL state output
- Valve power-law water-volume dispensing and the tone-state column
- Gas-puff falling-edge cumulative count, lick ADC threshold binarization, torque ADC scaling, brake torque,
  screen on/off state tracking
- How this registry plugs into the agnostic discover / partition / extract / merge primitives owned by
  `forging:microcontroller-primitives`
- Confirming parser output filenames match the `BehaviorDataFiles` roster

**Does not cover:**
- The agnostic feather discover / partition / extract / merge primitives (see
  `forging:microcontroller-primitives`)
- The hardware-state YAML schema and authoring (see `assets:session-hardware-state` and
  `mesoscope:mesoscope-vr-session-schema`)
- Upstream axci production of the raw module feathers (see `forging:behavior-input-format`)
- The `BehaviorDataFiles` filename roster and the producer-to-directory mapping for processed outputs (see
  `mesoscope:mesoscope-vr-processing-schema`)
- Assembly of these per-module outputs into the session `data.feather` (see
  `mesoscope:mesoscope-vr-dataset-assembly`)
- The batch orchestration that runs module jobs (see `forging:behavior-processing`)

---

## Module parsing contract (concretizes forging:microcontroller-primitives)

Each hardware module has one public entry point with the uniform signature `parse_<module>(event_partition,
output_directory, session)`, registered under its `(acquisition system, module_type, module_id)` triplet in
`sollertia_forgery.registries`. The agnostic microcontroller pipeline resolves the parsers for the session's
acquisition system, reads each raw module feather with `pl.read_ipc(source=..., memory_map=True)`, partitions it once
with the agnostic `partition_events` primitive, and calls the entry point with the resulting event-code-keyed
dictionary. The entry point never sees the input feather path and never performs its own registry lookup.

Every entry point then runs the same three steps:

1. Resolve the session's `MesoscopeHardwareState` through `_resolve_hardware_state`, which reads
   `session.raw_data.hardware_state_path` (the session's `raw_data/hardware_state.yaml`, whose parsing class the
   shared-assets `HARDWARE_STATE_REGISTRY` dispatches on the acquisition system) and raises `FileNotFoundError` when
   no file is present at that path.
2. Return early, writing nothing, when `_is_module_eligible` reports the module ineligible for the session.
3. Delegate to the module's private `_parse_<module>_data` function, passing the event partition, the resolved output
   file path (`output_directory / BehaviorDataFiles.<MEMBER>`), and the hardware state. The TTL parser receives the
   session in place of the hardware state, which it uses only to name the session in its error message.

`output_directory` is the session's `processed_data/microcontroller_data` directory. The producer-to-directory mapping
for every Mesoscope-VR processed output is owned by `mesoscope:mesoscope-vr-processing-schema`.

Every parser reads its events out of the partition with the agnostic `get_event_timestamps` (state-only events) or
`get_event_data` (payload-carrying events) primitives, merges paired streams with `merge_event_streams`, and writes its
result with `write_ipc(file=..., compression="uncompressed")` so the output is itself memory-mappable downstream. This
skill documents the per-module specifics layered on top of that shared primitive flow. The primitives themselves are
documented in `forging:microcontroller-primitives`.

`_MODULE_REGISTRY` maps each `(module_type, module_id)` pair to a frozen `_ModuleSpecification` carrying three tuples:

| Field             | Type              | Meaning                                                                      |
|-------------------|-------------------|------------------------------------------------------------------------------|
| `required_fields` | `tuple[str, ...]` | The `MesoscopeHardwareState` field names that must not be `None`             |
| `usage_flags`     | `tuple[str, ...]` | The `MesoscopeHardwareState` boolean field names that must be truthy         |
| `event_codes`     | `tuple[int, ...]` | The axci event codes the parser reads, which also seed the extraction filter |

`_ModuleSpecification.check_eligibility(hardware_state)` applies a different test to each of the two field tuples. A
`required_fields` entry disqualifies the module only when its value is `None`, which is the `MesoscopeHardwareState`
convention for a parameter that was never configured. A `required_fields` entry whose value is `False` does not
disqualify the module, so a session recording `screens_initially_on: false` still parses its screen module. A
`usage_flags` entry disqualifies the module whenever its value is falsy, because that flag records whether the module
was used at all.

`_is_module_eligible(module_key, hardware_state)` returns `False` for an unregistered pair and otherwise delegates to
`check_eligibility`. The pipeline narrows its work before dispatch through the public `get_eligible_modules(session)`
accessor, which returns the `(module_type, module_id)` pairs the session configured for use, and derives each
controller's extraction filter from the public `get_module_event_codes()` accessor.

---

## The eight-entry module registry

`_MODULE_REGISTRY` holds one specification per Mesoscope-VR hardware module, and `sollertia_forgery.registries` maps
each `(AcquisitionSystems.MESOSCOPE_VR, module_type, module_id)` triplet to the module's public entry point. Every
output filename is a `BehaviorDataFiles` member resolved inside the entry point rather than stored on the
specification, and the resolved filename string is shown for reference.

| Module   | `(type, id)` | Entry point             | Output filename                | `BehaviorDataFiles` member |
|----------|--------------|-------------------------|--------------------------------|----------------------------|
| Encoder  | `(2, 1)`     | `parse_encoder`         | `encoder_data.feather`         | `ENCODER`                  |
| TTL      | `(1, 1)`     | `parse_mesoscope_frame` | `mesoscope_frame_data.feather` | `MESOSCOPE_FRAME`          |
| Brake    | `(3, 1)`     | `parse_brake`           | `brake_data.feather`           | `BRAKE`                    |
| Valve    | `(5, 1)`     | `parse_valve`           | `valve_data.feather`           | `VALVE`                    |
| Gas puff | `(5, 2)`     | `parse_gas_puff`        | `gas_puff_data.feather`        | `GAS_PUFF`                 |
| Lick     | `(4, 1)`     | `parse_lick`            | `lick_data.feather`            | `LICK`                     |
| Torque   | `(6, 1)`     | `parse_torque`          | `torque_data.feather`          | `TORQUE`                   |
| Screen   | `(7, 1)`     | `parse_screen`          | `screen_data.feather`          | `SCREEN`                   |

The water valve `(5, 1)` and the gas-puff valve `(5, 2)` share module type `5` and are distinguished only by
`module_id`. Every output file carries a `time_us` uint64 column as its first column, and the per-parser value columns
are documented below.

---

## Encoder, TTL, brake, screen state parsers

**Encoder `(2, 1)` -> `encoder_data.feather`.** Reads CCW rotation (event `51`) and CW rotation (event `52`)
displacement payloads as `float64`. CCW is positive displacement, CW is negative (the CW stream is negated
before merging). If one direction is entirely absent, a single synthetic zero-displacement entry is inserted
one microsecond after the first opposite-direction timestamp so both streams are non-empty. The merged
displacements are scaled by `cm_per_pulse` and integrated with `np.cumsum` into cumulative traveled distance;
results are rounded to 8 decimals and negative-zero entries are normalized to `0.0`. Columns: `time_us`,
`traveled_distance_cm`.

**TTL `(1, 1)` -> `mesoscope_frame_data.feather`.** Reads ON (event `51`) and OFF (event `52`) transitions as
state-only events. Both edge polarities are required, and the parser raises `ValueError` naming the session and the
output file when either set of timestamps is empty, because the downstream fluorescence assembly pairs every rising
edge with its falling edge. ON maps to `1` and OFF to `0` (`uint8`), and a terminal OFF (`0`) is appended one
microsecond after the last sample if the final value is not already `0`. Columns: `time_us`, `ttl_state`. These pulses
are the alignment reference consumed by `mesoscope:mesoscope-vr-fluorescence-alignment`.

**Brake `(3, 1)` -> `brake_data.feather`.** Engaged (event `51`) applies `maximum_brake_strength` and disengaged
(event `52`) applies `minimum_brake_strength` (the residual torque from mechanical coupling). Both values are read off
the session's `MesoscopeHardwareState` and cast to `float64`. Columns: `time_us`, `brake_torque_N_cm`.

**Screen `(7, 1)` -> `screen_data.feather`.** Tracks LED VR-screen on/off state by detecting toggle-pulse
rising edges (ON event `51`, OFF event `52`). The initial state is `screens_initially_on` coerced to `int`
(`True` -> `1`, `False`/`None` -> `0`). If there are no ON transitions, the parser writes a single row at the
first OFF timestamp holding the initial state. Otherwise the state starts at the initial value and flips at
each rising edge, with the first row carrying the initial state at the first recorded timestamp. Columns:
`time_us`, `screen_state` (`uint8`).

---

## Valve and gas-puff event parsers

**Valve `(5, 1)` -> `valve_data.feather`.** Open (event `51`) and close (event `52`) transitions are merged
into a `1`/`0` signal; rising and falling edges of that signal bound each dispense pulse. Pulse duration
(close minus open, as `float64`) is converted to dispensed volume by the calibrated power law
`scale_coefficient * duration ** nonlinearity_exponent`, where `scale_coefficient` is
`valve_scale_coefficient` and the exponent is `valve_nonlinearity_exponent`. Per-pulse volumes are summed
with `np.cumsum` (rounded to 8 decimals), with an initial `0.0` volume re-added at the first timestamp. If
the valve never opened, the parser writes a single zero-volume, zero-tone row at the first close timestamp.
Tone-buzzer state is read separately (ON event `54`, OFF event `55`), with a terminal OFF appended if the
last tone value is not `0`. Volume and tone are then interpolated onto the shared union of their timestamps
via discrete interpolation. Columns: `time_us`, `dispensed_water_volume_uL`, `tone_state`.

**Gas puff `(5, 2)` -> `gas_puff_data.feather`.** Open (event `51`) and close (event `52`) transitions are
merged into a `1`/`0` `puff_state` stream. The cumulative puff count is the running sum of falling edges
(open-to-closed `1 -> 0` transitions) as `uint32`. The hardware state is unused but required for uniform
dispatch. If no gas puffs were delivered, the parser writes a single zero-state, zero-count row at the first
close timestamp. Columns: `time_us`, `puff_state` (`uint8`), `cumulative_puff_count` (`uint32`).

---

## Lick and torque sensor parsers

**Lick `(4, 1)` -> `lick_data.feather`.** Reads voltage-change events (event `51` only) as `uint16`,
preserving the raw 12-bit ADC resolution. The parser raises `ValueError` if `lick_threshold` is `None` (this
indicates a mismatch between the eligibility filter and the registry, since eligibility should have skipped
the module). Timestamps and voltages are stably re-sorted, then binarized: a sample is a lick (`1`) when its
voltage is greater than or equal to `lick_threshold` (`uint16`), otherwise `0` (`uint8`). Columns: `time_us`,
`voltage_12_bit_adc` (`uint16`), `lick_state` (`uint8`).

**Torque `(6, 1)` -> `torque_data.feather`.** Reads CCW (event `51`, positive) and CW (event `52`, negative)
ADC payloads as `float64`, each scaled by `torque_per_adc_unit` before merging (the CW stream is negated). As
with the encoder, a missing direction is synthesized as a single zero entry one microsecond after the first
opposite-direction timestamp. Merged torques are rounded to 8 decimals, a terminal zero is appended if the
last value is not `0`, and negative-zero entries are normalized to `0.0`. Columns: `time_us`, `torque_N_cm`.

---

## Hardware-state eligibility fields per module

| Module   | Required fields (None-tested)                            | Usage flags (falsy skips the module) | Meaning                                                 |
|----------|----------------------------------------------------------|--------------------------------------|---------------------------------------------------------|
| Encoder  | `cm_per_pulse`                                           |                                      | Pulse-to-centimeter displacement scaling                |
| TTL      |                                                          | `recorded_mesoscope_ttl`             | Whether the session recorded mesoscope frame TTL pulses |
| Brake    | `maximum_brake_strength`, `minimum_brake_strength`       |                                      | Engaged and disengaged brake torque levels              |
| Valve    | `valve_scale_coefficient`, `valve_nonlinearity_exponent` |                                      | Power-law dispensed-volume calibration                  |
| Gas puff |                                                          | `delivered_gas_puffs`                | Whether the session delivered gas puffs                 |
| Lick     | `lick_threshold`                                         |                                      | ADC threshold for lick binarization                     |
| Torque   | `torque_per_adc_unit`                                    |                                      | ADC-to-N·cm torque scaling                              |
| Screen   | `screens_initially_on`                                   |                                      | Initial screen state before the first toggle            |

The schema, semantics, and authoring of these `MesoscopeHardwareState` fields are owned by
`assets:session-hardware-state` and `mesoscope:mesoscope-vr-session-schema`. This skill documents only which fields
gate each parser and how the parser uses them.

---

## Adding a new module parser

You MUST keep a new parser consistent with the existing contract. When adding support for a new hardware
module:

1. Write a public `parse_<module>(event_partition, output_directory, session)` entry point in
   `mesoscope_vr/microcontrollers.py` that resolves the hardware state from the session, returns early when the module
   is ineligible, and delegates to a private `_parse_<module>_data` helper. Read events only through the agnostic
   primitives (`get_event_timestamps`, `get_event_data`, `merge_event_streams`) from
   `forging:microcontroller-primitives`, and do not re-scan the DataFrame directly.
2. Write the output with `write_ipc(file=output_file, compression="uncompressed")` so the result is
   memory-mappable downstream, and lead the schema with a `time_us` uint64 column.
3. Add the output filename as a new `BehaviorDataFiles` member (owned by
   `mesoscope:mesoscope-vr-processing-schema`) and build the output path from that member inside the entry point,
   never hardcoding the filename string.
4. Register a new `_ModuleSpecification` under the module's `(module_type, module_id)` key in `_MODULE_REGISTRY`,
   listing every `MesoscopeHardwareState` field the parser reads in `required_fields`, every boolean recording whether
   the module was used at all in `usage_flags`, and every event code the parser reads in `event_codes`. Then register
   the entry point under its `(acquisition system, module_type, module_id)` triplet in `sollertia_forgery.registries`.
   If the parser reads a field, that field MUST appear in `required_fields`, and the lick parser's `ValueError` guard
   exists precisely to catch a parser/registry skew.
5. If the new column must reach the assembled session feather, coordinate with
   `mesoscope:mesoscope-vr-dataset-assembly` — assembly is out of scope here.

---

## Related skills

| Skill                                      | Relationship                                                                        |
|--------------------------------------------|-------------------------------------------------------------------------------------|
| `forging:microcontroller-primitives`       | Owns the agnostic discover / partition / extract / merge primitives this skill uses |
| `forging:data-processing-design`           | The platform-general processing doctrine this per-system parser instantiates        |
| `forging:behavior-results`                 | Output-discovery / verification reference; defers conversion detail here            |
| `mesoscope:mesoscope-vr-session-schema`    | Owns the `MesoscopeHardwareState` calibration-field schema and authoring            |
| `mesoscope:mesoscope-vr-processing-schema` | Owns the `BehaviorDataFiles` roster and the producer-to-directory mapping           |
| `forging:behavior-input-format`            | Owns the raw module-feather inputs this parser consumes                             |

---

## Verification checklist

```text
- [ ] Module dispatch uses the (module_type, module_id) tuple as the _MODULE_REGISTRY key (controller_id discarded)
- [ ] Output filenames sourced from BehaviorDataFiles members, never hardcoded strings
- [ ] Every event code, conversion, and required field cited matches mesoscope_vr/microcontrollers.py
- [ ] New parsers read events only via the forging:microcontroller-primitives primitives
- [ ] New parsers write uncompressed Arrow IPC and lead with a time_us uint64 column
- [ ] required_fields lists every hardware-state field the parser reads, usage_flags lists only usage booleans
- [ ] Public entry points take (event_partition, output_directory, session) and resolve the hardware state themselves
- [ ] Did not re-document the agnostic primitive API (deferred to forging:microcontroller-primitives)
- [ ] Did not re-document hardware-state schema or BehaviorDataFiles roster (deferred to the schema skills)
- [ ] Cross-references use exact plugin:skill syntax with no ataraxis@ prefix
```
