---
name: mesoscope-vr-session-schema
description: >-
  Documents Mesoscope-VR's concrete instance of the per-system session-record contract: the four
  session descriptors (LickTraining, RunTraining, MesoscopeExperiment, WindowChecking) and the
  MesoscopeHardwareState snapshot, with exact field names, types, defaults, enums, and
  per-session-type applicability. Field-level schema reference only. Use when reading, amending,
  validating, or reasoning about the fields of a Mesoscope-VR session_descriptor.yaml or
  hardware_state.yaml, when deciding which descriptor class a session type uses, when checking
  which hardware-state fields a session type populates versus leaves None, or when grading a
  window-checking surgery_quality value.
user-invocable: false
---

# Mesoscope-VR session schema

Documents the field-level schema of Mesoscope-VR's session descriptors and hardware-state snapshot —
Mesoscope-VR's concrete instance of the universal per-system session-record contract.

Descriptors, hardware-state, trial-types, experiment-config, and raw-data are a **universal
per-system contract**: every Sollertia acquisition system is expected to define its own concrete
schema for each. They are not Mesoscope-VR quirks. The system-agnostic core — the generic
create/write/validate/describe tools and the dispatch registries (`DESCRIPTOR_REGISTRY`,
`HARDWARE_STATE_REGISTRY`, `SYSTEM_SESSION_TYPES`, `SESSION_TYPES_USING_VR_TASK`) — lives in the
`assets` plugin. This skill owns only Mesoscope-VR's **concrete instance** of that contract: the
actual fields, types, defaults, and enums the Mesoscope-VR system writes to disk.

The dataclasses documented here live in `sollertia-shared-assets`
(`mesoscope_vr/runtime_data.py`), and are registered in `registries.py`. Both files are the
authoritative source of truth — read them if a field detail is in doubt; do not trust a stale
table here.

---

## Scope

**Covers:**
- The four-descriptor roster for Mesoscope-VR and which `SessionTypes` value maps to each
- Per-descriptor field tables (names, types, defaults) for `LickTrainingDescriptor`,
  `RunTrainingDescriptor`, `MesoscopeExperimentDescriptor`, `WindowCheckingDescriptor`
- The shared `experimenter` / `animal_weight_g` / `incomplete` / `experimenter_notes` contract and
  which descriptors carry which shared fields
- `WindowCheckingDescriptor.surgery_quality` (integer `0`–`3`) and its omission of `animal_weight_g`
- The `MesoscopeHardwareState` 11-field schema and the `None` = "module not used" convention
- Per-session-type hardware-state field population (which fields are set vs. `None` per session type)
- Per-session-type applicability (window-checking yields no `hardware_state.yaml`)
- The `delivered_gas_puffs` derivation from `MesoscopeGasPuffTrial` in `trial_structures`

**Does not cover:**
- The generic read/write/validate/describe MCP tools and registry dispatch — owned by the `assets`
  plugin (`assets:session-descriptors`, `assets:session-hardware-state`)
- Runtime **behavior** that populates, seeds, or completes these records (state machine, descriptor
  consumption, threshold seeding mechanics) — owned by `/mesoscope-vr-runtime`
- The Zaber and mesoscope-objective position snapshots — owned by `/mesoscope-vr-snapshots`
- The experiment configuration (`MesoscopeExperimentConfiguration`, `trial_structures`,
  `experiment_states`) beyond the `delivered_gas_puffs` derivation
- Path resolution, session discovery, and the `SessionData` marker file

---

## The per-system session-record contract for Mesoscope-VR

Mesoscope-VR instantiates the per-system contract with four descriptor classes (one per session
type) and one hardware-state class (one per acquisition system). The canonical on-disk filenames
never vary — `session_descriptor.yaml` and `hardware_state.yaml` — only the parsing dataclass does.

| Contract slot      | Registry                  | Keyed by             | Mesoscope-VR class(es)                                                                                         |
|--------------------|---------------------------|----------------------|----------------------------------------------------------------------------------------------------------------|
| Session descriptor | `DESCRIPTOR_REGISTRY`     | `SessionTypes`       | `LickTrainingDescriptor`, `RunTrainingDescriptor`, `MesoscopeExperimentDescriptor`, `WindowCheckingDescriptor` |
| Hardware state     | `HARDWARE_STATE_REGISTRY` | `AcquisitionSystems` | `MesoscopeHardwareState`                                                                                       |

`SYSTEM_SESSION_TYPES[MESOSCOPE_VR]` claims all four session types; `SESSION_TYPES_USING_VR_TASK`
contains only `MESOSCOPE_EXPERIMENT`, so only experiment sessions also write a
`vr_configuration.yaml` task-template snapshot. Registry dispatch mechanics are owned by the
`assets` plugin — this skill documents only the resolved Mesoscope-VR classes.

---

## Session descriptors

### Descriptor roster

Every descriptor file is named `session_descriptor.yaml` regardless of session type; the YAML
parses into a session-type-specific dataclass via `DESCRIPTOR_REGISTRY`:

| `SessionTypes` value   | Descriptor dataclass            |
|------------------------|---------------------------------|
| `lick training`        | `LickTrainingDescriptor`        |
| `run training`         | `RunTrainingDescriptor`         |
| `mesoscope experiment` | `MesoscopeExperimentDescriptor` |
| `window checking`      | `WindowCheckingDescriptor`      |

### Shared field contract

All four descriptors share three required-or-defaulted fields:

| Field                | Type   | Default                           | Meaning                                                                          |
|----------------------|--------|-----------------------------------|----------------------------------------------------------------------------------|
| `experimenter`       | `str`  | (required, no default)            | The ID of the experimenter running the session.                                  |
| `incomplete`         | `bool` | `True`                            | Whether the session's data is complete and eligible for unsupervised processing. |
| `experimenter_notes` | `str`  | `"Replace this with your notes."` | The experimenter's notes made during runtime.                                    |

`incomplete` is contract-enforced: `registries.py` asserts at import time that every descriptor in
`DESCRIPTOR_REGISTRY` declares an `incomplete` field, because the session-inspection tooling reads
it to decide processing eligibility.

The three **non-window-checking** descriptors (`LickTrainingDescriptor`, `RunTrainingDescriptor`,
`MesoscopeExperimentDescriptor`) additionally share:

| Field                                | Type    | Default                | Meaning                                                                               |
|--------------------------------------|---------|------------------------|---------------------------------------------------------------------------------------|
| `animal_weight_g`                    | `float` | (required, no default) | The animal's weight, in grams, at the beginning of the session.                       |
| `maximum_unconsumed_rewards`         | `int`   | `1`                    | Cap on consecutive delivered-but-unconsumed rewards before delivery is paused.        |
| `dispensed_water_volume_ml`          | `float` | `0.0`                  | Total water, in mL, dispensed during runtime (excludes the paused/idle state).        |
| `pause_dispensed_water_volume_ml`    | `float` | `0.0`                  | Total water, in mL, dispensed during the paused (idle) state.                         |
| `experimenter_given_water_volume_ml` | `float` | `0.0`                  | Additional water, in mL, administered manually by the experimenter after the session. |

`WindowCheckingDescriptor` carries **none** of these: no `animal_weight_g`,
`maximum_unconsumed_rewards`, or water totals. Its only session-type-specific field is
`surgery_quality`.

### Per-descriptor field tables

The tables below cover only the **session-type-specific** fields beyond the shared sets above.
Required fields (`experimenter`, and `animal_weight_g` for the non-window-checking types) come first
in the dataclass; defaulted fields follow.

#### `LickTrainingDescriptor`

| Field                       | Type    | Default | Meaning                                                               |
|-----------------------------|---------|---------|-----------------------------------------------------------------------|
| `minimum_reward_delay_s`    | `int`   | `6`     | Minimum delay, in seconds, between two consecutive water rewards.     |
| `maximum_reward_delay_s`    | `int`   | `18`    | Maximum delay, in seconds, between two consecutive water rewards.     |
| `maximum_water_volume_ml`   | `float` | `1.0`   | Maximum water volume, in mL, the system may dispense during training. |
| `maximum_training_time_min` | `int`   | `20`    | Maximum time, in minutes, the system may run the training.            |
| `water_reward_size_ul`      | `float` | `5.0`   | Water volume, in microliters, dispensed per reward.                   |
| `reward_tone_duration_ms`   | `int`   | `300`   | Duration, in milliseconds, of the reward auditory tone.               |

Plus the shared non-window-checking fields (`animal_weight_g`, `maximum_unconsumed_rewards`, the
three water-total floats) and the three universal fields (`experimenter`, `incomplete`,
`experimenter_notes`).

#### `RunTrainingDescriptor`

| Field                              | Type    | Default | Meaning                                                                              |
|------------------------------------|---------|---------|--------------------------------------------------------------------------------------|
| `final_run_speed_threshold_cm_s`   | `float` | `1.5`   | Running speed threshold, in cm/s, at the end of training.                            |
| `final_run_duration_threshold_s`   | `float` | `1.5`   | Running duration threshold, in seconds, at the end of training.                      |
| `initial_run_speed_threshold_cm_s` | `float` | `0.8`   | Initial running speed threshold, in cm/s.                                            |
| `initial_run_duration_threshold_s` | `float` | `1.5`   | Initial running duration threshold, in seconds.                                      |
| `increase_threshold_ml`            | `float` | `0.1`   | Water volume, in mL, that triggers a threshold increase.                             |
| `run_speed_increase_step_cm_s`     | `float` | `0.05`  | Speed-threshold increment, in cm/s, applied per `increase_threshold_ml`.             |
| `run_duration_increase_step_s`     | `float` | `0.1`   | Duration-threshold increment, in seconds, applied per `increase_threshold_ml`.       |
| `maximum_water_volume_ml`          | `float` | `1.0`   | Maximum water volume, in mL, the system may dispense during training.                |
| `maximum_training_time_min`        | `int`   | `40`    | Maximum time, in minutes, the system may run the training.                           |
| `maximum_idle_time_s`              | `float` | `0.3`   | Max time, in seconds, the animal may dip below the speed threshold and still reward. |
| `water_reward_size_ul`             | `float` | `5.0`   | Water volume, in microliters, dispensed per reward.                                  |
| `reward_tone_duration_ms`          | `int`   | `300`   | Duration, in milliseconds, of the reward auditory tone.                              |

Plus the shared non-window-checking fields and the three universal fields. Note
`maximum_training_time_min` defaults to `40` here versus `20` for lick training.

#### `MesoscopeExperimentDescriptor`

Adds **no** session-type-specific fields beyond the shared non-window-checking set
(`animal_weight_g`, `maximum_unconsumed_rewards`, the three water-total floats) and the three
universal fields (`experimenter`, `incomplete`, `experimenter_notes`). Reward schedule and zone
parameters live on the experiment configuration, not the descriptor.

#### `WindowCheckingDescriptor`

| Field             | Type  | Default | Meaning                                                                                                 |
|-------------------|-------|---------|---------------------------------------------------------------------------------------------------------|
| `surgery_quality` | `int` | `0`     | Cranial window / surgery quality on a `0`–`3` inclusive scale: `0` non-usable to `3` publication-grade. |

Carries only `experimenter`, `surgery_quality`, `incomplete`, and `experimenter_notes` — no
`animal_weight_g`, no reward fields, no water totals.

---

## Hardware state

### `MesoscopeHardwareState` schema

`MesoscopeHardwareState` is the Mesoscope-VR instance of the per-system hardware-state contract,
keyed by `AcquisitionSystems.MESOSCOPE_VR` in `HARDWARE_STATE_REGISTRY` and written to
`hardware_state.yaml`. Every field defaults to `None`, and `None` means **"the corresponding
hardware module was not used by the executed runtime"** — not "missing data." That convention is
load-bearing for downstream pipelines and MUST be preserved on any amendment.

| Field                         | Type                     | Default | Meaning                                                                              |
|-------------------------------|--------------------------|---------|--------------------------------------------------------------------------------------|
| `cm_per_pulse`                | `float \| None`          | `None`  | Conversion factor translating encoder pulses into centimeters.                       |
| `maximum_brake_strength`      | `float \| None`          | `None`  | Braking torque, in N·cm, at the wheel edge when the brake is maximally engaged.      |
| `minimum_brake_strength`      | `float \| None`          | `None`  | Braking torque, in N·cm, at the wheel edge when the brake is fully disengaged.       |
| `lick_threshold`              | `int \| None`            | `None`  | Lick threshold, in 12-bit ADC units, above which the sensor signal counts as a lick. |
| `valve_scale_coefficient`     | `float \| None`          | `None`  | Scale coefficient of the valve open-time / dispensed-volume power law.               |
| `valve_nonlinearity_exponent` | `float \| None`          | `None`  | Nonlinearity exponent of the valve open-time / dispensed-volume power law.           |
| `torque_per_adc_unit`         | `float \| None`          | `None`  | Conversion factor translating torque sensor 12-bit ADC units into N·cm.              |
| `screens_initially_on`        | `bool \| None`           | `None`  | Initial state of the Virtual Reality screens at the start of the session's runtime.  |
| `recorded_mesoscope_ttl`      | `bool \| None`           | `None`  | Whether the session recorded brain activity data with the mesoscope.                 |
| `delivered_gas_puffs`         | `bool \| None`           | `None`  | Whether the session delivered any gas puffs to the animal.                           |
| `system_state_codes`          | `dict[str, int] \| None` | `None`  | Maps human-readable state names to integer system state-codes.                       |

### Per-session-type applicability and field population

The hardware-state file exists only for session types whose runtime **exercises hardware modules**.
Window-checking sessions never produce a `hardware_state.yaml`; reading it for one fails with a
missing-file error by design, not corruption. Which fields are populated versus left `None` is
deterministic per session type (a runtime-side convention owned by `/mesoscope-vr-runtime`, the
producer; this table documents the resulting schema state):

| Session type           | Populated fields                                                                                                                                                                      | Left as `None`                                                                                                               |
|------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `mesoscope experiment` | All 11 fields. `recorded_mesoscope_ttl=True`. `delivered_gas_puffs` is computed from whether any `MesoscopeGasPuffTrial` exists in the experiment configuration's `trial_structures`. | None — every field is set.                                                                                                   |
| `lick training`        | `torque_per_adc_unit`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.                                 | `cm_per_pulse`, `maximum_brake_strength`, `minimum_brake_strength`, `screens_initially_on`, `recorded_mesoscope_ttl`.        |
| `run training`         | `cm_per_pulse`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.                                        | `maximum_brake_strength`, `minimum_brake_strength`, `torque_per_adc_unit`, `screens_initially_on`, `recorded_mesoscope_ttl`. |
| `window checking`      | (no file produced — see above)                                                                                                                                                        | n/a                                                                                                                          |

When amending a snapshot, set fields that the matching session type leaves as `None` to `None`.
A numeric value would imply a hardware module was used when it was not, misinforming the processing
pipeline's eligibility checks.

### The `delivered_gas_puffs` derivation

`delivered_gas_puffs` is **not** a direct measurement. For `mesoscope experiment` sessions it is
derived: `True` when at least one `MesoscopeGasPuffTrial` appears in the experiment configuration's
`trial_structures`, `False` otherwise. For lick-training and run-training it is fixed `False`
(those runtimes never deliver gas puffs). The `trial_structures` source and `MesoscopeGasPuffTrial`
shape belong to the experiment configuration (see the `assets` plugin); this skill documents only
the derived hardware-state value.

---

## Related skills

| Skill                           | Relationship                                                                                                     |
|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| `assets:session-descriptors`    | Generic owner of the descriptor read/write/validate/describe tools and `DESCRIPTOR_REGISTRY` dispatch.           |
| `assets:session-hardware-state` | Generic owner of the hardware-state read/write/validate/describe tools and `HARDWARE_STATE_REGISTRY` dispatch.   |
| `/mesoscope-vr-runtime`         | Owns the runtime behavior that populates, seeds, and completes these records (state machine, threshold seeding). |
| `/mesoscope-vr-snapshots`       | Sibling per-session records — the frozen Zaber and mesoscope-objective position snapshots.                       |
| `/mesoscope-vr`                 | Mesoscope-VR hardware composition and configuration that backs the populated hardware-state fields.              |

---

## Verification checklist

```text
- [ ] Field names, types, and defaults match sollertia-shared-assets/mesoscope_vr/runtime_data.py exactly
- [ ] Descriptor → SessionTypes mapping matches DESCRIPTOR_REGISTRY in registries.py
- [ ] WindowCheckingDescriptor documented as omitting animal_weight_g and carrying only surgery_quality (0–3)
- [ ] Shared experimenter / animal_weight_g / incomplete / experimenter_notes contract stated correctly
- [ ] MesoscopeHardwareState documented with all 11 fields and the None = "module not used" convention
- [ ] Per-session-type hardware-state population table matches the producer convention (window-checking = no file)
- [ ] delivered_gas_puffs documented as derived from MesoscopeGasPuffTrial in trial_structures (experiment only)
- [ ] Generic tool mechanics deferred to assets:session-descriptors and assets:session-hardware-state, not re-documented
- [ ] Runtime behavior deferred to /mesoscope-vr-runtime, not duplicated
- [ ] All lines ≤ 120 chars (tables may exceed for alignment); file under 500 lines
```
