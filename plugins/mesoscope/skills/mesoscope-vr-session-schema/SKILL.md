---
name: mesoscope-vr-session-schema
description: >-
  Documents Mesoscope-VR's concrete instance of the per-system session-record contract: the four session descriptors
  (LickTraining, RunTraining, MesoscopeExperiment, WindowChecking) and the MesoscopeHardwareState snapshot, with their
  field names, types, defaults, enums, and per-session-type applicability. Use when reading, amending, or validating a
  Mesoscope-VR session_descriptor.yaml or hardware_state.yaml, deciding which descriptor class a session type uses,
  checking which hardware-state fields a session type populates or leaves None, or grading a surgery_quality value.
user-invocable: false
---

# Mesoscope-VR session schema

Documents the field-level schema of Mesoscope-VR's session descriptors and hardware-state snapshot, Mesoscope-VR's
concrete instance of the universal per-system session-record contract.

Descriptors, hardware-state, trial-types, experiment-config, and raw-data are a **universal per-system contract**, and
every Sollertia acquisition system is expected to define its own concrete schema for each. The system-agnostic core,
meaning the generic create/write/validate/describe tools and the dispatch registries (`DESCRIPTOR_REGISTRY`,
`HARDWARE_STATE_REGISTRY`, `SYSTEM_SESSION_TYPES`, `SESSION_TYPES_USING_VR_TASK`), lives in the `assets` plugin. This
skill owns only Mesoscope-VR's **concrete instance** of that contract, meaning the actual fields, types, defaults, and
enums the Mesoscope-VR system writes to disk.

The dataclasses documented here live in `sollertia-shared-assets` (`mesoscope_vr/runtime_data.py`), and are registered
in `registries.py`. Both files are the authoritative source of truth. Read them if a field detail is in doubt, rather
than trusting a stale table here. Each class is importable as `from sollertia_shared_assets import <Name>` or from
`sollertia_shared_assets.mesoscope_vr`.

---

## Scope

**Covers:**
- The four-descriptor roster for Mesoscope-VR and which `SessionTypes` value maps to each
- The four per-session-type descriptor filenames the runtime writes into the animal's `persistent_data` cache
- Per-descriptor field tables (names, types, defaults) for `LickTrainingDescriptor`, `RunTrainingDescriptor`,
  `MesoscopeExperimentDescriptor`, `WindowCheckingDescriptor`
- The shared `experimenter` / `animal_weight_g` / `incomplete` / `experimenter_notes` contract and which descriptors
  carry which shared fields
- `WindowCheckingDescriptor.surgery_quality` (integer `0`-`3`) and its omission of `animal_weight_g`
- The `MesoscopeHardwareState` 11-field schema and the `None` = "module not used" convention
- Per-session-type hardware-state field population (which fields are set vs. `None` per session type)
- Per-session-type applicability (window-checking yields no `hardware_state.yaml`)
- The `delivered_gas_puffs` derivation from `MesoscopeGasPuffTrial` in `trial_structures`
- The workflow for widening `MesoscopeHardwareState` with a field for a newly added hardware module

**Does not cover:**
- The generic read/write/validate/describe MCP tools and registry dispatch. Owned by the `assets` plugin
  (`assets:session-descriptors`, `assets:session-hardware-state`)
- Runtime **behavior** that populates, seeds, or completes these records (state machine, descriptor consumption,
  threshold seeding mechanics). Owned by `/mesoscope-vr-runtime`
- The Zaber and mesoscope-objective position snapshots. Owned by `/mesoscope-vr-snapshots`
- The experiment configuration (`MesoscopeExperimentConfiguration`, `trial_structures`, `experiment_states`) beyond the
  `delivered_gas_puffs` derivation. Owned by `/mesoscope-vr-experiment-schema`
- Path resolution (`assets:project-hierarchy`), session discovery (`assets:session-discovery`), and the `SessionData`
  marker file (`assets:session-data`)

---

## The per-system session-record contract for Mesoscope-VR

Mesoscope-VR instantiates the per-system contract with four descriptor classes (one per session type) and one
hardware-state class (one per acquisition system). Inside `raw_data` the canonical on-disk filenames never vary. A
descriptor is always `session_descriptor.yaml` and a hardware-state snapshot is always `hardware_state.yaml`, and only
the parsing dataclass changes.

| Contract slot      | Registry                  | Keyed by             | Mesoscope-VR class(es)                                                                                         |
|--------------------|---------------------------|----------------------|----------------------------------------------------------------------------------------------------------------|
| Session descriptor | `DESCRIPTOR_REGISTRY`     | `SessionTypes`       | `LickTrainingDescriptor`, `RunTrainingDescriptor`, `MesoscopeExperimentDescriptor`, `WindowCheckingDescriptor` |
| Hardware state     | `HARDWARE_STATE_REGISTRY` | `AcquisitionSystems` | `MesoscopeHardwareState`                                                                                       |

`SYSTEM_SESSION_TYPES[MESOSCOPE_VR]` claims all four session types. `SESSION_TYPES_USING_VR_TASK` contains only
`MESOSCOPE_EXPERIMENT`, so only experiment sessions also write a `vr_configuration.yaml` task-template snapshot.
Registry dispatch mechanics are owned by the `assets` plugin, and this skill documents only the resolved Mesoscope-VR
classes.

This skill is also the worked reference an extender copies when authoring a new acquisition system's
`<system>/runtime_data.py`. The extension workflow itself is owned by `assets:library-extension`.

---

## Session descriptors

### Descriptor roster

| `SessionTypes` value   | Descriptor dataclass            | Persistent cache filename              |
|------------------------|---------------------------------|----------------------------------------|
| `lick training`        | `LickTrainingDescriptor`        | `lick_training_descriptor.yaml`        |
| `run training`         | `RunTrainingDescriptor`         | `run_training_descriptor.yaml`         |
| `mesoscope experiment` | `MesoscopeExperimentDescriptor` | `mesoscope_experiment_descriptor.yaml` |
| `window checking`      | `WindowCheckingDescriptor`      | `window_checking_descriptor.yaml`      |

The per-animal persistent cache at `<animal>/persistent_data/` keeps the most recent descriptor of each session type
under the third column's filename. The cached copy is therefore named after the session type instead of carrying the
flat `session_descriptor.yaml` name used inside `raw_data`. The Mesoscope-VR acquisition runtime hardcodes those four
names, and it recovers the previous session's animal weight and water intake from the newest of the three
non-window-checking caches. `assets:project-hierarchy` owns the `persistent_data` path property itself.

### Shared field contract

All four descriptors share three required-or-defaulted fields:

| Field                | Type   | Default                           | Meaning                                                                                                                                                                                                                              |
|----------------------|--------|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `experimenter`       | `str`  | (required, no default)            | The ID of the experimenter running the session.                                                                                                                                                                                      |
| `incomplete`         | `bool` | `True`                            | `True` marks the session as incomplete, meaning it ran past initialization but hit a runtime issue and may carry data gaps, so it is held back from unsupervised processing. The runtime flips it to `False` at a clean session end. |
| `experimenter_notes` | `str`  | `"Replace this with your notes."` | The experimenter's notes made during runtime.                                                                                                                                                                                        |

Every registered descriptor must declare `incomplete`, a platform contract enforced at import and owned by
`assets:library-extension`.

The three **non-window-checking** descriptors (`LickTrainingDescriptor`, `RunTrainingDescriptor`,
`MesoscopeExperimentDescriptor`) additionally share:

| Field                                | Type    | Default                | Meaning                                                                                                                                                                       |
|--------------------------------------|---------|------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `animal_weight_g`                    | `float` | (required, no default) | The animal's weight, in grams, at the beginning of the session.                                                                                                               |
| `maximum_unconsumed_rewards`         | `int`   | `1`                    | Cap on consecutive delivered-but-unconsumed rewards before delivery is paused. Setting it to `0` removes the limit entirely, so every delivered reward may remain unconsumed. |
| `dispensed_water_volume_ml`          | `float` | `0.0`                  | Total water, in mL, dispensed during runtime (excludes the paused/idle state).                                                                                                |
| `pause_dispensed_water_volume_ml`    | `float` | `0.0`                  | Total water, in mL, dispensed during the paused (idle) state.                                                                                                                 |
| `experimenter_given_water_volume_ml` | `float` | `0.0`                  | Additional water, in mL, administered manually by the experimenter after the session.                                                                                         |

`WindowCheckingDescriptor` carries **none** of these. Its only session-type-specific field is `surgery_quality`.

### Per-descriptor field tables

The tables below cover only the **session-type-specific** fields beyond the shared sets above. Required fields
(`experimenter`, and `animal_weight_g` for the non-window-checking types) come first in the dataclass, and defaulted
fields follow.

#### `LickTrainingDescriptor`

| Field                       | Type    | Default | Meaning                                                                                                      |
|-----------------------------|---------|---------|--------------------------------------------------------------------------------------------------------------|
| `minimum_reward_delay_s`    | `int`   | `6`     | Minimum delay, in seconds, between two consecutive water rewards.                                            |
| `maximum_reward_delay_s`    | `int`   | `18`    | Maximum delay, in seconds, between two consecutive water rewards.                                            |
| `maximum_water_volume_ml`   | `float` | `1.0`   | Maximum water volume, in mL, the system may dispense during training.                                        |
| `maximum_training_time_min` | `int`   | `20`    | Maximum time, in minutes, the system may run the training.                                                   |
| `water_reward_size_ul`      | `float` | `5.0`   | Water volume, in microliters, dispensed on each reward of the training's pseudorandom reward-delay sequence. |
| `reward_tone_duration_ms`   | `int`   | `300`   | Duration, in milliseconds, of the reward auditory tone.                                                      |

Plus the shared non-window-checking fields (`animal_weight_g`, `maximum_unconsumed_rewards`, the three water-total
floats) and the three universal fields (`experimenter`, `incomplete`, `experimenter_notes`).

#### `RunTrainingDescriptor`

| Field                              | Type    | Default | Meaning                                                                                                              |
|------------------------------------|---------|---------|----------------------------------------------------------------------------------------------------------------------|
| `final_run_speed_threshold_cm_s`   | `float` | `1.5`   | Running speed threshold, in cm/s, at the end of training.                                                            |
| `final_run_duration_threshold_s`   | `float` | `1.5`   | Running duration threshold, in seconds, at the end of training.                                                      |
| `initial_run_speed_threshold_cm_s` | `float` | `0.8`   | Initial running speed threshold, in cm/s.                                                                            |
| `initial_run_duration_threshold_s` | `float` | `1.5`   | Initial running duration threshold, in seconds.                                                                      |
| `increase_threshold_ml`            | `float` | `0.1`   | Water volume, in mL, that triggers a threshold increase.                                                             |
| `run_speed_increase_step_cm_s`     | `float` | `0.05`  | Speed-threshold increment, in cm/s, applied per `increase_threshold_ml`.                                             |
| `run_duration_increase_step_s`     | `float` | `0.1`   | Duration-threshold increment, in seconds, applied per `increase_threshold_ml`.                                       |
| `maximum_water_volume_ml`          | `float` | `1.0`   | Maximum water volume, in mL, the system may dispense during training.                                                |
| `maximum_training_time_min`        | `int`   | `40`    | Maximum time, in minutes, the system may run the training.                                                           |
| `maximum_idle_time_s`              | `float` | `0.3`   | Max time, in seconds, the animal may dip below the speed threshold and still reward.                                 |
| `water_reward_size_ul`             | `float` | `5.0`   | Water volume, in microliters, dispensed when the animal achieves the required running speed and duration thresholds. |
| `reward_tone_duration_ms`          | `int`   | `300`   | Duration, in milliseconds, of the reward auditory tone.                                                              |

Plus the shared non-window-checking fields and the three universal fields. Note `maximum_training_time_min` defaults to
`40` here versus `20` for lick training.

#### `MesoscopeExperimentDescriptor`

Adds **no** session-type-specific fields beyond the shared non-window-checking set (`animal_weight_g`,
`maximum_unconsumed_rewards`, the three water-total floats) and the three universal fields (`experimenter`,
`incomplete`, `experimenter_notes`). Reward size and tone duration live on the experiment configuration's trial classes,
owned by `/mesoscope-vr-experiment-schema`, and the trial zone geometry lives on the task template's `TrialStructure` in
`vr_configuration.yaml`, owned by `assets:task-templates`.

#### `WindowCheckingDescriptor`

| Field             | Type  | Default | Meaning                                                                                                                                                                                                                                                                                                                           |
|-------------------|-------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `surgery_quality` | `int` | `0`     | Cranial window / surgery quality on a `0`-`3` inclusive scale, `0` non-usable to `3` publication-grade. The range is a convention, not a constraint. `WindowCheckingDescriptor` declares no `__post_init__`, so `write_session_descriptor_tool` accepts an out-of-range integer without error. Validate the value before writing. |

Carries only `experimenter`, `surgery_quality`, `incomplete`, and `experimenter_notes`.

---

## Hardware state

### `MesoscopeHardwareState` schema

`MesoscopeHardwareState` is the Mesoscope-VR instance of the per-system hardware-state contract, registered in
`HARDWARE_STATE_REGISTRY` keyed by `AcquisitionSystems.MESOSCOPE_VR`, whose string value is `mesoscope`, the value every
`acquisition_system` tool argument expects. The member name is not accepted. The snapshot is written to
`hardware_state.yaml`. Every field defaults to `None`, and `None` means **"the corresponding hardware module was not
used by the executed runtime"** rather than "missing data". That convention is load-bearing for downstream pipelines and
MUST be preserved on any amendment. The convention is also unenforced, because `MesoscopeHardwareState` declares no
`__post_init__`, so `write_session_hardware_state_tool` performs a shape check only and accepts a value that contradicts
it. Validate the snapshot against the population table below before writing.

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

The hardware-state file exists only for session types whose runtime **exercises hardware modules**. Window-checking
sessions never produce a `hardware_state.yaml`, and reading it for one fails with a missing-file error by design rather
than by corruption. Which fields are populated versus left `None` is deterministic per session type, a runtime-side
convention owned by `/mesoscope-vr-runtime` as the producer. The table below records the resulting schema state.

| Session type           | Populated fields                                                                                                                                      | Left as `None`                                                                                                               |
|------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `mesoscope experiment` | All 11 fields. `recorded_mesoscope_ttl=True`, and `delivered_gas_puffs` is derived.                                                                   | None, every field is set.                                                                                                    |
| `lick training`        | `torque_per_adc_unit`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`. | `cm_per_pulse`, `maximum_brake_strength`, `minimum_brake_strength`, `screens_initially_on`, `recorded_mesoscope_ttl`.        |
| `run training`         | `cm_per_pulse`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.        | `maximum_brake_strength`, `minimum_brake_strength`, `torque_per_adc_unit`, `screens_initially_on`, `recorded_mesoscope_ttl`. |
| `window checking`      | (no file produced, see above)                                                                                                                         | n/a                                                                                                                          |

When amending a snapshot, set fields that the matching session type leaves as `None` to `None`. A numeric value would
imply a hardware module was used when it was not, misinforming the processing pipeline's eligibility checks.

### The `delivered_gas_puffs` derivation

`delivered_gas_puffs` is **not** a direct measurement. For `mesoscope experiment` sessions it is derived, holding `True`
when at least one `MesoscopeGasPuffTrial` appears in the experiment configuration's `trial_structures` and `False`
otherwise. For lick-training and run-training it is fixed `False`, because those runtimes never deliver gas puffs. The
`trial_structures` source and `MesoscopeGasPuffTrial` shape belong to the experiment configuration (see
`/mesoscope-vr-experiment-schema`), and this skill documents only the derived hardware-state value.

---

## Adding a hardware-state field for a new module

A hardware module added to the acquisition runtime reaches the processing side only through a `MesoscopeHardwareState`
field, because every Mesoscope-VR module parser is gated on one. The surrounding add-a-module chain is owned by
`/mesoscope-vr`, under "Add a new module to an existing microcontroller" in its `references/modification-workflows.md`,
and this section owns the field itself.

1. **Append the field to `MesoscopeHardwareState`** in `mesoscope_vr/runtime_data.py`, carrying the `| None` union, the
   `None` default, and the unit-bearing docstring every existing field carries. Appending rather than inserting keeps
   the YAML key order of already written snapshots aligned with the class.
2. **Decide which eligibility group the field joins.** A calibration constant or a recorded state value belongs in the
   parser's `required_fields`, where only `None` disqualifies the module, so a legitimate `False` such as
   `screens_initially_on` still parses. A boolean recording whether the module was used at all belongs in
   `usage_flags`, where any falsy value skips the module. `/mesoscope-vr-module-parsing` owns both tuples.
3. **Set the field in every session type that drives the module**, and leave it `None` in the rest, then extend the
   population table above with the result. The runtime writes the snapshot from three hand-written per-session-type
   branches, and `/mesoscope-vr`'s modification workflow names that edit site.
4. **Regenerate the checked-in stub** with `tox -e stubs` in `sollertia-shared-assets`, which refreshes
   `mesoscope_vr/runtime_data.pyi` next to the source module. A stale stub ships with the release.
5. **Bump the `sollertia-shared-assets` version** in its `pyproject.toml`, then raise the pin in both consumers, the
   `sollertia-experiment` pin behind the runtime that writes the snapshot and the `sollertia-forgery` pin behind the
   parsers that read it. A consumer resolved below the bump loads a `MesoscopeHardwareState` that lacks the field, the
   eligibility check reads the absent attribute as `None`, and the module is skipped rather than reported.

---

## Related skills

| Skill                             | Relationship                                                                                                                                              |
|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/mesoscope-vr-cli-reference`     | Owns the `sle mesoscope run` commands whose runtime populates these records                                                                               |
| `assets:session-descriptors`      | Generic owner of the descriptor read/write/validate/describe tools and `DESCRIPTOR_REGISTRY` dispatch.                                                    |
| `assets:session-hardware-state`   | Generic owner of the hardware-state read/write/validate/describe tools and `HARDWARE_STATE_REGISTRY` dispatch.                                            |
| `assets:library-extension`        | Owns the platform contract that requires `incomplete`, and the workflow for authoring a new system's `runtime_data.py`.                                   |
| `/mesoscope-vr-experiment-schema` | Owns the trial classes and `trial_structures` behind the `delivered_gas_puffs` derivation, and the `REST`/`RUN` codes stored in `system_state_codes`.     |
| `/mesoscope-vr-module-parsing`    | Consumes the null convention through `check_eligibility` when selecting module parsers.                                                                   |
| `/mesoscope-vr-runtime`           | Owns the runtime behavior that populates, seeds, and completes these records (state machine, threshold seeding).                                          |
| `/mesoscope-vr-snapshots`         | Sibling per-session records, the frozen Zaber and mesoscope-objective position snapshots, and routes descriptor and hardware-state schema questions here. |
| `/mesoscope-vr`                   | Mesoscope-VR hardware composition and configuration that backs the populated hardware-state fields.                                                       |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Session schema:
- [ ] Field names, types, and defaults match sollertia-shared-assets/mesoscope_vr/runtime_data.py exactly
- [ ] Descriptor to SessionTypes mapping matches DESCRIPTOR_REGISTRY in registries.py
- [ ] Persistent-cache filenames listed for all four session types and marked as runtime-hardcoded
- [ ] WindowCheckingDescriptor documented as omitting animal_weight_g and carrying only surgery_quality (0-3)
- [ ] surgery_quality and the hardware-state None rule both documented as unenforced conventions
- [ ] incomplete documented with True meaning incomplete and held back from unsupervised processing
- [ ] Shared experimenter / animal_weight_g / incomplete / experimenter_notes contract stated correctly
- [ ] MesoscopeHardwareState documented with all 11 fields and the None = "module not used" convention
- [ ] AcquisitionSystems.MESOSCOPE_VR documented as the string value "mesoscope"
- [ ] Per-session-type hardware-state population table matches the producer convention (window-checking = no file)
- [ ] delivered_gas_puffs documented as derived from MesoscopeGasPuffTrial in trial_structures (experiment only)
- [ ] A new hardware-state field carries a None default, joins required_fields or usage_flags deliberately, and is
      reflected in the per-session-type population table
- [ ] A new hardware-state field was followed by tox -e stubs, a sollertia-shared-assets version bump, and raised
      pins in sollertia-experiment and sollertia-forgery
- [ ] Generic tool mechanics deferred to assets:session-descriptors and assets:session-hardware-state, not re-documented
- [ ] Runtime behavior deferred to /mesoscope-vr-runtime, not duplicated
- [ ] Every cross-reference uses the bare /skill-name or plugin:skill-name form, with no marketplace prefix
```
