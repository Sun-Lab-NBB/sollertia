---
name: mesoscope-vr-experiment-schema
description: >-
  Documents Mesoscope-VR's concrete instance of the per-system experiment-configuration contract: the
  MesoscopeExperimentConfiguration field schema, its MesoscopeWaterRewardTrial and MesoscopeGasPuffTrial trial classes,
  the trigger-type-to-trial mapping, the REST/RUN system-state codes, and the from_task_template builder defaults. Use
  when reading or hand-authoring a Mesoscope-VR experiment configuration YAML, interpreting its trial or state fields,
  deciding which TriggerType maps to which trial class, or checking a per-trial or per-state default.
user-invocable: false
---

# Mesoscope-VR experiment configuration schema

Documents the field-level schema of `MesoscopeExperimentConfiguration`, Mesoscope-VR's concrete implementation of the
per-system experiment-configuration contract.

Hardware-state, experiment-configuration, and raw-data are a **universal per-system contract**. Each has a registry
keyed by `AcquisitionSystems` (`HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`, and
`SYSTEM_RAW_DATA_REGISTRY`), so every Sollertia acquisition system defines its own instance of all three. Session
descriptors are keyed per **session type** instead, because `DESCRIPTOR_REGISTRY` is keyed by `SessionTypes`, so a
system that needs its own descriptor mints a new `SessionTypes` member. The system-agnostic core (the
create/write/validate/describe tools, the `EXPERIMENT_CONFIGURATION_REGISTRY` dispatch, and the platform `TriggerType`
taxonomy) lives in the `assets` plugin. This skill owns only Mesoscope-VR's concrete instance of the
experiment-configuration contract, meaning the field-level schema and semantics defined in `sollertia-shared-assets`'s
`mesoscope_vr/experiment_configuration.py`.

An extender authoring a new system's `<system>/experiment_configuration.py` copies this skill's worked example,
following the procedure `assets:library-extension` owns.

---

## Scope

**Covers:**
- The `MesoscopeExperimentConfiguration` field schema (the three contract fields and their types)
- The `MesoscopeWaterRewardTrial` and `MesoscopeGasPuffTrial` trial-class field schemas and defaults
- The `TrialKind` discriminator enum and the `trial_kind` field both trial classes declare
- The trial-kind resolution rules a hand-authored configuration must satisfy to load
- Mesoscope-VR's `TriggerType` to trial-class mapping (which `TriggerType` members are mapped, and which raise
  `ValueError`)
- The `REST` (`1`) / `RUN` (`2`) `system_state_code` values an experiment configuration accepts
- The `from_task_template` classmethod signature, defaults, and seeding behavior

**Does not cover:**
- The generic `create_experiment_from_vr_template_tool` / `write_experiment_configuration_tool` /
  `validate_experiment_configuration_tool` / `describe_experiment_configuration_schema_tool` tool mechanics. Owned by
  `assets:experiment-configuration`
- The `EXPERIMENT_CONFIGURATION_REGISTRY` dispatch, contract assertions, and the system-agnostic five-member
  `TriggerType` taxonomy. Owned by `assets:experiment-configuration`
- Authoring the Unity VR task template that seeds the configuration. Owned by `assets:task-templates`
- The system-agnostic `ExperimentState` field schema composed into this configuration. Owned by
  `assets:experiment-configuration`
- How the runtime loads and executes this configuration (state machine sequencing, guidance counters, trial
  decomposition), and the `MesoscopeVRStates` enum with its state-machine semantics. Owned by `/mesoscope-vr-runtime`
- The Mesoscope-VR hardware composition. Owned by `/mesoscope-vr`

---

## The contract fields

`MesoscopeExperimentConfiguration` is a `YamlConfig` dataclass declaring exactly the three contract fields every
`<System>ExperimentConfiguration` must declare. Mesoscope-VR adds no further fields, so the contract is the whole schema
for this system today.

| Field               | Type                                                            | Required | Meaning                                                                                                    |
|---------------------|-----------------------------------------------------------------|----------|------------------------------------------------------------------------------------------------------------|
| `trial_structures`  | `dict[str, MesoscopeWaterRewardTrial \| MesoscopeGasPuffTrial]` | Yes      | The trials the experiment runs, keyed by trial name, with Mesoscope-VR's trial classes as values           |
| `experiment_states` | `dict[str, ExperimentState]`                                    | Yes      | The experiment state machine, keyed by state name                                                          |
| `unity_scene_name`  | `str`                                                           | Yes      | The Unity scene (VR task) in the linear infinite corridor. Identifies the paired template by filename stem |

`trial_structures` carries **only per-trial runtime parameters**. The matching spatial fields (cue sequence, zones,
`trigger_type`, occupancy duration) live on the paired `TaskTemplate`'s `trial_structures[<same name>]`, and are joined
to this config by trial name at session init.

---

## The trial classes

Mesoscope-VR defines two frozen, slotted runtime trial dataclasses. Each carries the runtime stimulus parameters plus
the `trial_kind` discriminator that identifies it on disk, and each raises `ValueError` from `__post_init__` when
`trial_kind` holds any member other than its own. The behavioral success or avoidance condition is defined by the task
template, not by these classes. Another acquisition system declares its own trial classes for whatever `TriggerType`
subset it supports.

### The trial-kind discriminator

`TrialKind` is Mesoscope-VR's own `StrEnum`, declared alongside the trial classes rather than in the platform enum
module, and it carries exactly two members. The field that stores them is named by the module-level
`_DISCRIMINATOR_FIELD = "trial_kind"` constant.

| Member            | Value     | Runtime trial class         |
|-------------------|-----------|-----------------------------|
| `TrialKind.WATER` | `"water"` | `MesoscopeWaterRewardTrial` |
| `TrialKind.PUFF`  | `"puff"`  | `MesoscopeGasPuffTrial`     |

`TrialKind` is public. It is exported from `sollertia_shared_assets.mesoscope_vr` and re-exported from the package root,
so `from sollertia_shared_assets import TrialKind` is valid. The discriminator is what routes a stored trial back to the
class that wrote it. Deserialization tries the members of the trial union in order and skips an arm whose initialization
raises.

### `MesoscopeWaterRewardTrial`

A reinforcing trial that delivers a water reward when the animal meets the trial's success condition.

| Field                     | Type        | Default           | Meaning                                                                    |
|---------------------------|-------------|-------------------|----------------------------------------------------------------------------|
| `reward_size_ul`          | `float`     | `5.0`             | Volume of water, in microliters, delivered on successful trial completion  |
| `reward_tone_duration_ms` | `int`       | `300`             | Duration, in milliseconds, of the auditory tone sounded with the reward    |
| `trial_kind`              | `TrialKind` | `TrialKind.WATER` | The discriminator that identifies the trial when it is read back from YAML |

### `MesoscopeGasPuffTrial`

An aversive trial that delivers a gas puff when the animal fails the trial's avoidance condition.

| Field              | Type        | Default          | Meaning                                                                    |
|--------------------|-------------|------------------|----------------------------------------------------------------------------|
| `puff_duration_ms` | `int`       | `100`            | Duration, in milliseconds, of the gas puff delivered when the animal fails |
| `trial_kind`       | `TrialKind` | `TrialKind.PUFF` | The discriminator that identifies the trial when it is read back from YAML |

---

## The trigger-type-to-trial mapping

Mesoscope-VR maps only the **subset** of the platform `TriggerType` taxonomy it supports. The template carries each
trial's `trigger_type`, and `from_task_template` pairs it with a runtime trial class as follows:

| Template `trigger_type` | Mesoscope-VR runtime trial class | Result                                          |
|-------------------------|----------------------------------|-------------------------------------------------|
| `interaction`           | `MesoscopeWaterRewardTrial`      | Mapped                                          |
| `occupancy_disarm`      | `MesoscopeGasPuffTrial`          | Mapped                                          |
| `collision`             | (none)                           | Raises `ValueError`, not mapped on Mesoscope-VR |
| `occupancy_arm`         | (none)                           | Raises `ValueError`, not mapped on Mesoscope-VR |
| `occupancy_trigger`     | (none)                           | Raises `ValueError`, not mapped on Mesoscope-VR |

`from_task_template` iterates the template's `trial_structures`. Any trial whose `trigger_type` is not `INTERACTION` or
`OCCUPANCY_DISARM` raises a `ValueError` reporting that the trigger type "is not mapped to a runtime trial class". The
full five-member `TriggerType` taxonomy and its system-agnostic semantics are owned by
`assets:experiment-configuration`, and this table records only Mesoscope-VR's mapped subset.

---

## The experiment-state schema

`experiment_states` values are `ExperimentState` instances, a system-agnostic dataclass composed into this
configuration. Its field schema, defaults, and access rules are owned by `assets:experiment-configuration`, and this
skill records only what Mesoscope-VR constrains on top of them.

### Mesoscope-VR `system_state_code` values

`system_state_code` holds the integer value of a member of Mesoscope-VR's `MesoscopeVRStates` enum (defined in
`sollertia-experiment`), which carries five members:

| `MesoscopeVRStates` member | Code | Accepted in an experiment configuration |
|----------------------------|------|-----------------------------------------|
| `IDLE`                     | `0`  | No                                      |
| `REST`                     | `1`  | Yes                                     |
| `RUN`                      | `2`  | Yes                                     |
| `LICK_TRAINING`            | `3`  | No                                      |
| `RUN_TRAINING`             | `4`  | No                                      |

Session initialization walks every `experiment_states` value and raises `ValueError` on any code other than `1` or `2`.
The `0` that `from_task_template` seeds is therefore `IDLE`, a valid Mesoscope-VR system state that an experiment
configuration rejects, and it must be edited to `1` or `2` before the session can run. How the runtime installs these
states is owned by `/mesoscope-vr-runtime`, and this skill documents only which codes an experiment configuration
accepts.

---

## The `from_task_template` builder

`MesoscopeExperimentConfiguration.from_task_template` is the contract builder `create_experiment_from_vr_template_tool`
dispatches to for Mesoscope-VR. It maps the template's trial structures to runtime trial classes by trigger type, then
seeds `state_count` default-valued runtime states.

### Signature and defaults

```python
@classmethod
def from_task_template(
    cls,
    template: TaskTemplate,
    unity_scene_name: str,
    state_count: int = 1,
    default_reward_size_ul: float = 5.0,
    default_reward_tone_duration_ms: int = 300,
    default_puff_duration_ms: int = 100,
) -> MesoscopeExperimentConfiguration: ...
```

| Parameter                         | Type           | Default    | Role                                                                                         |
|-----------------------------------|----------------|------------|----------------------------------------------------------------------------------------------|
| `template`                        | `TaskTemplate` | (required) | The VR task template whose trial structures seed the configuration                           |
| `unity_scene_name`                | `str`          | (required) | Stored verbatim. Should match the template filename (caller's responsibility, not validated) |
| `state_count`                     | `int`          | `1`        | Number of default-valued runtime states to generate. Must be at least 1                      |
| `default_reward_size_ul`          | `float`        | `5.0`      | `reward_size_ul` for every `interaction` (water-reward) trial                                |
| `default_reward_tone_duration_ms` | `int`          | `300`      | `reward_tone_duration_ms` for every `interaction` trial                                      |
| `default_puff_duration_ms`        | `int`          | `100`      | `puff_duration_ms` for every `occupancy_disarm` (gas-puff) trial                             |

The generic creation tool always passes `template`, `unity_scene_name`, and `state_count` by keyword. The three
`default_*` parameters carry defaults so the tool can omit the system-specific generation values, which is what the
registry contract assertion checks.

`from_task_template` raises `ValueError` when `state_count` is less than `1`, checked before any trial is mapped.

### State-seeding behavior

For each of the `state_count` states, the builder emits a `state_{n}` (1-indexed) `ExperimentState` with:

- `experiment_state_code = n` (the 1-indexed state number)
- `system_state_code = 0` (`MesoscopeVRStates.IDLE`)
- `state_duration_s = 60` (the `_DEFAULT_STATE_DURATION_S` state-duration constant)
- `supports_trials = True`

The guidance counters are populated **only for the trial classes that exist** in the seeded `trial_structures`,
mirroring the trial types present in the template, and a counter whose trial class is absent holds `0`:

| Guidance counter (reinforcing or aversive) | Seeded value when the matching trial type is present | Source constant                      |
|--------------------------------------------|------------------------------------------------------|--------------------------------------|
| `*_initial_guided_trials`                  | `3`                                                  | `_DEFAULT_INITIAL_GUIDED_TRIALS`     |
| `*_recovery_failed_threshold`              | `9`                                                  | `_DEFAULT_RECOVERY_FAILED_THRESHOLD` |
| `*_recovery_guided_trials`                 | `3`                                                  | `_DEFAULT_RECOVERY_GUIDED_TRIALS`    |

These constants are private to `experiment_configuration.py`. Only the per-trial `default_*` parameters above are
overridable through the builder.

---

## Reading a configuration back

`MesoscopeExperimentConfiguration.restore_excluded_fields` is the deserialization hook `YamlConfig.from_yaml` calls on
the loaded mapping before it becomes a dataclass. It returns the mapping unchanged unless `trial_structures` is a dict,
in which case it delegates every stored trial to `_restore_trial_kind`. That helper is driven by `_TRIAL_CLASSES` and
`_unique_trial_fields`. `_TRIAL_CLASSES` pairs each `TrialKind` member with the runtime trial class that declares it as
its default. `_unique_trial_fields` computes from `dataclasses.fields` the field names one class declares and its
sibling does not, which are `reward_size_ul` and `reward_tone_duration_ms` for the water class and `puff_duration_ms`
for the puff class.

Four rules govern a hand-authored trial:

| Stored trial                                               | Outcome                                                                                                                  |
|------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Declares fields unique to two classes                      | Raises `ValueError`, naming the kinds identified and asking for the foreign fields to be removed                         |
| Omits `trial_kind`                                         | Takes the kind its unique fields identify, falling back to `water` when it declares none                                 |
| Declares an unrecognized `trial_kind`                      | Raises `ValueError`, naming the valid values `water, puff`                                                               |
| Declares a `trial_kind` that contradicts its unique fields | Raises `ValueError`, because the discriminator alone decides the class and the fields it disagrees with would be dropped |

A field belonging to neither class is ignored during resolution, matching how the loader treats a key it does not
recognize.

`MesoscopeExperimentConfiguration.__post_init__` runs after the mapping is converted, on every constructed instance,
including every instance `from_task_template` returns. It raises `ValueError` listing, sorted, every trial in
`trial_structures` that did not resolve to a `MesoscopeWaterRewardTrial` or a `MesoscopeGasPuffTrial`. The loader runs
with per-field type checking disabled, so an unmatched trial would otherwise survive as a raw mapping and fail in the
acquisition runtime instead.

---

## Related skills

| Skill                             | Relationship                                                                                                                                                                                                                              |
|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/mesoscope-vr-cli-reference`     | Owns the `sle mesoscope configure experiment` command that writes this configuration                                                                                                                                                      |
| `assets:experiment-configuration` | Owns the generic create/write/validate/describe tooling, `EXPERIMENT_CONFIGURATION_REGISTRY` dispatch, the system-agnostic `TriggerType` taxonomy, and the `ExperimentState` schema. This skill is its Mesoscope-VR field-level companion |
| `assets:task-templates`           | Owns the `TaskTemplate` (with per-trial `trigger_type` and spatial `TrialStructure`) that seeds `from_task_template`                                                                                                                      |
| `assets:library-extension`        | Owns the procedure for adding a new acquisition system, for which this skill is the worked experiment-configuration example                                                                                                               |
| `/mesoscope-vr-runtime`           | Owns how the runtime loads and executes this configuration (state sequencing, guidance, trial decomposition, `system_state_code` installation) and the `MesoscopeVRStates` enum                                                           |
| `/mesoscope-vr`                   | Owns the Mesoscope-VR hardware composition                                                                                                                                                                                                |

---

## Verification checklist

You MUST verify any field, default, or enum cited from this skill against the `sollertia-shared-assets` source before
relying on it.

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Experiment configuration schema:
- [ ] Field names, types, and defaults match mesoscope_vr/experiment_configuration.py exactly
- [ ] The three contract fields are reported as the whole schema (Mesoscope-VR adds no extra fields)
- [ ] Trial-class fields cited as reward_size_ul=5.0, reward_tone_duration_ms=300, puff_duration_ms=100, and trial_kind
- [ ] Trial-kind rules: two-class clash raises, omission falls back to water, unknown or contradicting kind raises
- [ ] Only `interaction` and `occupancy_disarm` are described as mapped, and the other three raise ValueError
- [ ] system_state_code cited only as 1=REST and 2=RUN accepted (0 is IDLE, valid for the system, rejected here)
- [ ] The ExperimentState field schema deferred to assets:experiment-configuration (not re-documented here)
- [ ] from_task_template defaults cited as state_count=1, state_duration_s=60, guidance 3/9/3
- [ ] Generic tool mechanics deferred to assets:experiment-configuration (not re-documented here)
- [ ] Runtime load/execute behavior deferred to /mesoscope-vr-runtime (not re-documented here)
- [ ] Template authoring deferred to assets:task-templates (not re-documented here)
- [ ] Every cross-reference uses the bare /skill-name or plugin:skill-name form, with no marketplace prefix
```
