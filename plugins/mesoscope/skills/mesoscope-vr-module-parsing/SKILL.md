---
name: mesoscope-vr-module-parsing
description: >-
  Documents Mesoscope-VR's concrete instance of the per-system module-parser contract: the eight-entry
  module registry keyed by (module_type, module_id), each parser's output feather filename, event codes,
  required MesoscopeHardwareState calibration fields, output column schema, and unit conversions. Use when
  interpreting or modifying a Mesoscope-VR microcontroller feather output, adding a new module parser, or
  mapping a hardware module to its calibrated processed output.
user-invocable: false
---

# Mesoscope-VR module parsing

Documents how the Mesoscope-VR processing pipeline converts pre-extracted microcontroller module feather
files into calibrated, domain-specific behavior feather files, one parser per hardware module.

This skill owns the concrete Mesoscope-VR instance of the module-parser contract. It is the single source
for the `(module_type, module_id)` to conversion mapping: which parser runs, what output filename and column
schema it produces, which event codes it reads, and which `MesoscopeHardwareState` calibration fields it
requires. It concretizes the system-agnostic feather-parsing primitives owned by
`forging:microcontroller-primitives`; this skill never re-documents those primitives, only how the
Mesoscope-VR registry consumes them.

All of the symbols below live in `sollertia_forgery.mesoscope_vr.microcontrollers`. The registry
(`_MODULE_REGISTRY`) and the eight `_parse_*` functions are forgery-internal; the module-level entry points
`process_microcontroller_data` and `is_module_eligible` are the only callers, and the batch orchestration
that invokes them is owned by `forging:behavior-processing`.

---

## Scope

**Covers:**
- The eight-parser module registry dispatch on `(module_type, module_id)`
- Per-module output filename, event codes, required `MesoscopeHardwareState` fields, and output column schema
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
- Upstream axci production of the raw module feathers (see `ataraxis@communication:log-processing`)
- The pipeline-wide `BehaviorDataFiles` filename roster definition (see
  `mesoscope:mesoscope-vr-processing-schema`)
- Assembly of these per-module outputs into the session `data.feather` (see
  `mesoscope:mesoscope-vr-dataset-assembly`)
- The batch orchestration that runs module jobs (see `forging:behavior-processing`)

---

## Module parsing contract (concretizes forging:microcontroller-primitives)

A single module feather is processed by `process_microcontroller_data(feather_path, output_directory,
hardware_state)`. The flow is fully data-driven off the input filename and the registry:

1. Recover the module identity from the filename via the agnostic `parse_module_feather_name` primitive,
   which returns the `(controller_id, module_type, module_id)` integer tuple. Only `(module_type, module_id)`
   is used as the registry key; the controller ID is discarded.
2. Look up `_MODULE_REGISTRY[(module_type, module_id)]`. A pair with no registered specification raises
   `ValueError` ("does not match any registered module specification").
3. Read the input feather with `pl.read_ipc(source=feather_path, memory_map=True)` — memory-mapping is
   valid because all module feather writes use uncompressed Arrow IPC — then partition it once with the
   agnostic `partition_events` primitive into an event-code-keyed dictionary.
4. Dispatch to the specification's `parse_function`, passing the event partition, the resolved output file
   path (`output_directory / specification.output_filename`), and the `MesoscopeHardwareState`.

Every parser reads its events out of the partition with the agnostic `get_event_timestamps` (state-only
events) or `get_event_data` (payload-carrying events) primitives, merges paired streams with
`merge_event_streams`, and writes its result with `write_ipc(file=..., compression="uncompressed")` so the
output is itself memory-mappable downstream. This skill documents the per-module specifics layered on top of
that shared primitive flow; the primitives themselves are documented in `forging:microcontroller-primitives`.

`_ModuleSpecification` is a frozen dataclass with three fields:

| Field             | Type                     | Meaning                                                              |
|-------------------|--------------------------|---------------------------------------------------------------------|
| `parse_function`  | `Callable[..., None]`    | Transforms the event partition into the domain-specific feather      |
| `output_filename` | `str`                    | The output feather name, sourced from a `BehaviorDataFiles` member   |
| `required_fields` | `tuple[str, ...]`        | The `MesoscopeHardwareState` field names that gate eligibility       |

`_ModuleSpecification.check_eligibility(hardware_state)` returns `False` if any required field is `None`, and
also returns `False` for a boolean required field whose value is `False` (so an unset `delivered_gas_puffs`
or `recorded_mesoscope_ttl` boolean disqualifies the module). `is_module_eligible(module_type, module_id,
hardware_state)` is the public wrapper: it returns `False` for an unregistered pair and otherwise delegates
to `check_eligibility`. Modules that are not eligible are skipped by the orchestration before
`process_microcontroller_data` is called.

---

## The eight-entry module registry

`_MODULE_REGISTRY` maps each `(module_type, module_id)` pair to its specification. The output filename is a
`BehaviorDataFiles` enum member; the resolved filename string is shown for reference.

| `(type, id)` | Parser                 | Output filename                | `BehaviorDataFiles` member | Required hardware-state fields                            |
|--------------|------------------------|--------------------------------|----------------------------|----------------------------------------------------------|
| `(2, 1)`     | `_parse_encoder_data`  | `encoder_data.feather`         | `ENCODER`                  | `cm_per_pulse`                                            |
| `(1, 1)`     | `_parse_ttl_data`      | `mesoscope_frame_data.feather` | `MESOSCOPE_FRAME`          | `recorded_mesoscope_ttl`                                 |
| `(3, 1)`     | `_parse_brake_data`    | `brake_data.feather`           | `BRAKE`                    | `maximum_brake_strength`, `minimum_brake_strength`       |
| `(5, 1)`     | `_parse_valve_data`    | `valve_data.feather`           | `VALVE`                    | `valve_scale_coefficient`, `valve_nonlinearity_exponent` |
| `(5, 2)`     | `_parse_gas_puff_data` | `gas_puff_data.feather`        | `GAS_PUFF`                 | `delivered_gas_puffs`                                    |
| `(4, 1)`     | `_parse_lick_data`     | `lick_data.feather`            | `LICK`                     | `lick_threshold`                                         |
| `(6, 1)`     | `_parse_torque_data`   | `torque_data.feather`          | `TORQUE`                   | `torque_per_adc_unit`                                    |
| `(7, 1)`     | `_parse_screen_data`   | `screen_data.feather`          | `SCREEN`                   | `screens_initially_on`                                   |

The water valve `(5, 1)` and the gas-puff valve `(5, 2)` share module type `5` and are distinguished only by
`module_id`. Every output file carries a `time_us` uint64 column as its first column; the per-parser value
columns are documented below.

---

## Encoder, TTL, brake, screen state parsers

**Encoder `(2, 1)` -> `encoder_data.feather`.** Reads CCW rotation (event `51`) and CW rotation (event `52`)
displacement payloads as `float64`. CCW is positive displacement, CW is negative (the CW stream is negated
before merging). If one direction is entirely absent, a single synthetic zero-displacement entry is inserted
one microsecond after the first opposite-direction timestamp so both streams are non-empty. The merged
displacements are scaled by `cm_per_pulse` and integrated with `np.cumsum` into cumulative traveled distance;
results are rounded to 8 decimals and negative-zero entries are normalized to `0.0`. Columns: `time_us`,
`traveled_distance_cm`.

**TTL `(1, 1)` -> `mesoscope_frame_data.feather`.** Reads ON (event `51`) and OFF (event `52`) transitions
as state-only events. The hardware state is unused here but is required for uniform dispatch. If either ON or
OFF timestamps are missing the parser returns without writing (rising edges cannot be detected). ON maps to
`1` and OFF to `0` (`uint8`); a terminal OFF (`0`) is appended one microsecond after the last sample if the
final value is not already `0`. Columns: `time_us`, `ttl_state`. These pulses are the alignment reference
consumed by `mesoscope:mesoscope-vr-fluorescence-alignment`.

**Brake `(3, 1)` -> `brake_data.feather`.** Engaged (event `51`) applies `maximum_brake_strength`;
disengaged (event `52`) applies `minimum_brake_strength` (the residual torque from mechanical coupling). The
parser reads `minimum_brake_strength` and falls back to the legacy misspelled field `minimum_break_strength`
when the canonical field is `None`. Both values are `float64`. Columns: `time_us`, `brake_torque_N_cm`.

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

## Required hardware-state calibration fields per module

A module is processed only when every field in its specification's `required_fields` is configured on the
session's `MesoscopeHardwareState`. The eligibility rule is uniform: a `None` field disqualifies the module,
and a boolean field set to `False` also disqualifies it.

| Parser                 | Required fields                                          | Used for                                                      |
|------------------------|---------------------------------------------------------|--------------------------------------------------------------|
| `_parse_encoder_data`  | `cm_per_pulse`                                          | Pulse-to-centimeter displacement scaling                     |
| `_parse_ttl_data`      | `recorded_mesoscope_ttl`                               | Boolean gate; field value otherwise unused in the parser     |
| `_parse_brake_data`    | `maximum_brake_strength`, `minimum_brake_strength`     | Engaged / disengaged torque levels (legacy spelling fallback) |
| `_parse_valve_data`    | `valve_scale_coefficient`, `valve_nonlinearity_exponent` | Power-law volume calibration                               |
| `_parse_gas_puff_data` | `delivered_gas_puffs`                                  | Boolean gate; field value otherwise unused in the parser     |
| `_parse_lick_data`     | `lick_threshold`                                       | ADC threshold for lick binarization                          |
| `_parse_torque_data`   | `torque_per_adc_unit`                                  | ADC-to-N·cm torque scaling                                   |
| `_parse_screen_data`   | `screens_initially_on`                                 | Initial screen state before the first toggle                 |

The schema, semantics, and authoring of these `MesoscopeHardwareState` fields are owned by
`assets:session-hardware-state` and `mesoscope:mesoscope-vr-session-schema`; this skill documents only which
fields each parser requires and how it uses them.

---

## Adding a new module parser

You MUST keep a new parser consistent with the existing contract. When adding support for a new hardware
module:

1. Write a `_parse_<module>_data(event_partition, output_file, hardware_state)` function in
   `mesoscope_vr/microcontrollers.py`. Read events only through the agnostic primitives
   (`get_event_timestamps`, `get_event_data`, `merge_event_streams`) from
   `forging:microcontroller-primitives` — do not re-scan the DataFrame directly.
2. Write the output with `write_ipc(file=output_file, compression="uncompressed")` so the result is
   memory-mappable downstream, and lead the schema with a `time_us` uint64 column.
3. Add the output filename as a new `BehaviorDataFiles` member (owned by
   `mesoscope:mesoscope-vr-processing-schema`) and reference that member as the specification's
   `output_filename` — never hardcode the filename string in the registry.
4. Register a new `_ModuleSpecification` under the module's `(module_type, module_id)` key in
   `_MODULE_REGISTRY`, listing every `MesoscopeHardwareState` field the parser reads in `required_fields` so
   eligibility gating matches the parser's needs. If the parser reads a field, that field MUST appear in
   `required_fields`; the lick parser's `ValueError` guard exists precisely to catch a parser/registry skew.
5. If the new column must reach the assembled session feather, coordinate with
   `mesoscope:mesoscope-vr-dataset-assembly` — assembly is out of scope here.

---

## Related skills

| Skill                                  | Relationship                                                                       |
|----------------------------------------|------------------------------------------------------------------------------------|
| `forging:microcontroller-primitives`   | Owns the agnostic discover / partition / extract / merge primitives this skill uses |
| `forging:data-processing-design`       | The platform-general processing doctrine this per-system parser instantiates        |
| `forging:behavior-results`             | Output-discovery / verification reference; defers conversion detail here             |
| `mesoscope:mesoscope-vr-session-schema`| Owns the `MesoscopeHardwareState` calibration-field schema and authoring             |
| `mesoscope:mesoscope-vr-processing-schema` | Owns the `BehaviorDataFiles` filename roster the registry references            |
| `ataraxis@communication:log-processing`| Produces the raw axci module feathers consumed as parser input                      |

---

## Verification checklist

```text
- [ ] Module dispatch uses the (module_type, module_id) tuple as the _MODULE_REGISTRY key (controller_id discarded)
- [ ] Output filenames sourced from BehaviorDataFiles members, never hardcoded strings
- [ ] Every event code, conversion, and required field cited matches mesoscope_vr/microcontrollers.py
- [ ] New parsers read events only via the forging:microcontroller-primitives primitives
- [ ] New parsers write uncompressed Arrow IPC and lead with a time_us uint64 column
- [ ] required_fields lists every MesoscopeHardwareState field the parser reads (eligibility matches parser)
- [ ] Did not re-document the agnostic primitive API (deferred to forging:microcontroller-primitives)
- [ ] Did not re-document hardware-state schema or BehaviorDataFiles roster (deferred to the schema skills)
- [ ] Cross-references use plugin:skill syntax with ataraxis@ prefix for marketplace plugins
```
