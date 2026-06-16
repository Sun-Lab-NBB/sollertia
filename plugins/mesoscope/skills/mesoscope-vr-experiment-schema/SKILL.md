---
name: mesoscope-vr-experiment-schema
description: >-
  Documents Mesoscope-VR's concrete instance of the per-system experiment-configuration contract:
  the MesoscopeExperimentConfiguration field schema, its MesoscopeWaterRewardTrial and
  MesoscopeGasPuffTrial trial classes, the trigger-type-to-trial mapping, the REST/RUN system-state
  codes, and the from_task_template builder defaults. Use when reading or hand-authoring a
  Mesoscope-VR experiment configuration YAML, interpreting its trial or state fields, deciding which
  TriggerType maps to which trial class, or checking a per-trial or per-state default. For the
  generic create/write/validate/describe tooling and registry dispatch, defer to
  assets:experiment-configuration; for how the runtime loads and executes the config, see
  /mesoscope-vr-runtime.
user-invocable: false
---

# Mesoscope-VR experiment configuration schema

Documents `MesoscopeExperimentConfiguration` — Mesoscope-VR's concrete implementation of the
per-system experiment-configuration contract — at the field level: its three contract fields, its
two runtime trial classes, the trigger-type-to-trial mapping, the `REST`/`RUN` system-state codes,
and the `from_task_template` builder defaults.

Descriptors, hardware-state, trial-types, experiment-config, and raw-data are a **universal
per-system contract** — every Sollertia acquisition system is expected to define its own instance.
They are not Mesoscope-VR quirks. The system-agnostic core (the create/write/validate/describe
tools, the `EXPERIMENT_CONFIGURATION_REGISTRY` dispatch, and the platform `TriggerType` taxonomy)
lives in the `assets` plugin. This skill owns only Mesoscope-VR's concrete instance of that
contract: the actual field-level schema and semantics defined in
`sollertia-shared-assets`'s `mesoscope_vr/experiment_configuration.py`.

---

## Scope

**Covers:**
- The `MesoscopeExperimentConfiguration` field schema (the three contract fields and their types)
- The `MesoscopeWaterRewardTrial` and `MesoscopeGasPuffTrial` trial-class field schemas and defaults
- The `ExperimentState` field schema as composed into a Mesoscope-VR configuration
- Mesoscope-VR's `TriggerType` to trial-class mapping (which `TriggerType` members are mapped, and
  which raise `ValueError`)
- The `REST` (`1`) / `RUN` (`2`) `system_state_code` values Mesoscope-VR accepts
- The `from_task_template` classmethod signature, defaults, and seeding behavior

**Does not cover:**
- The generic `create_experiment_from_vr_template_tool` /
  `write_experiment_configuration_tool` / `validate_experiment_configuration_tool` /
  `describe_experiment_configuration_schema_tool` tool mechanics — owned by
  `assets:experiment-configuration`
- The `EXPERIMENT_CONFIGURATION_REGISTRY` dispatch, contract assertions, and the system-agnostic
  five-member `TriggerType` taxonomy — owned by `assets:experiment-configuration`
- Authoring the Unity VR task template that seeds the configuration — owned by
  `assets:task-templates`
- How the runtime loads and executes this configuration (state machine sequencing, guidance
  counters, trial decomposition) — owned by `/mesoscope-vr-runtime`
- The Mesoscope-VR hardware composition and `MesoscopeVRStates` enum source — see `/mesoscope-vr`

---

## The contract fields

`MesoscopeExperimentConfiguration` is a `YamlConfig` dataclass declaring exactly the three contract
fields every `<System>ExperimentConfiguration` must declare. Mesoscope-VR adds no further fields —
the contract is the whole schema for this system today.

| Field               | Type                                   | Required                | Meaning                                                                                                    |
|---------------------|----------------------------------------|-------------------------|------------------------------------------------------------------------------------------------------------|
| `trial_structures`  | `dict[str, MesoscopeWaterRewardTrial \ | MesoscopeGasPuffTrial]` | Yes | The trials the experiment runs, keyed by trial name; values are Mesoscope-VR's trial classes         |
| `experiment_states` | `dict[str, ExperimentState]`           | Yes                     | The experiment state machine, keyed by state name                                                          |
| `unity_scene_name`  | `str`                                  | Yes                     | The Unity scene (VR task) in the linear infinite corridor; identifies the paired template by filename stem |

`trial_structures` carries **only per-trial runtime parameters** — the matching spatial fields (cue
sequence, zones, `trigger_type`, occupancy duration) live on the paired `TaskTemplate`'s
`trial_structures[<same name>]` and are joined to this config by trial name at session init.

---

## The trial classes

Mesoscope-VR defines two frozen, slotted runtime trial dataclasses. Each carries only the runtime
stimulus parameters; the behavioral success/avoidance condition is defined by the task template, not
by these classes. Another acquisition system declares its own trial classes for whatever
`TriggerType` subset it supports.

### `MesoscopeWaterRewardTrial`

A reinforcing trial that delivers a water reward when the animal meets the trial's success
condition.

| Field                     | Type    | Default | Meaning                                                                   |
|---------------------------|---------|---------|---------------------------------------------------------------------------|
| `reward_size_ul`          | `float` | `5.0`   | Volume of water, in microliters, delivered on successful trial completion |
| `reward_tone_duration_ms` | `int`   | `300`   | Duration, in milliseconds, of the auditory tone sounded with the reward   |

### `MesoscopeGasPuffTrial`

An aversive trial that delivers a gas puff when the animal fails the trial's avoidance condition.

| Field              | Type  | Default | Meaning                                                                    |
|--------------------|-------|---------|----------------------------------------------------------------------------|
| `puff_duration_ms` | `int` | `100`   | Duration, in milliseconds, of the gas puff delivered when the animal fails |

---

## The trigger-type-to-trial mapping

Mesoscope-VR maps only the **subset** of the platform `TriggerType` taxonomy it supports. The
template carries each trial's `trigger_type`; `from_task_template` pairs it with a runtime trial
class as follows:

| Template `trigger_type` | Mesoscope-VR runtime trial class | Result                                           |
|-------------------------|----------------------------------|--------------------------------------------------|
| `interaction`           | `MesoscopeWaterRewardTrial`      | Mapped                                           |
| `occupancy_disarm`      | `MesoscopeGasPuffTrial`          | Mapped                                           |
| `collision`             | (none)                           | Raises `ValueError` — not mapped on Mesoscope-VR |
| `occupancy_arm`         | (none)                           | Raises `ValueError` — not mapped on Mesoscope-VR |
| `occupancy_trigger`     | (none)                           | Raises `ValueError` — not mapped on Mesoscope-VR |

`from_task_template` iterates the template's `trial_structures`; any trial whose `trigger_type` is
not `INTERACTION` or `OCCUPANCY_DISARM` raises a `ValueError` reporting that the trigger type "is not
mapped to a runtime trial class". The full five-member `TriggerType` taxonomy and its system-agnostic
semantics are owned by `assets:experiment-configuration`; this table records only Mesoscope-VR's
mapped subset.

---

## The experiment-state schema

`experiment_states` values are `ExperimentState` instances (a system-agnostic dataclass composed
into this configuration). It is a frozen, slotted dataclass with these fields:

| Field                                   | Type    | Default | Meaning                                                                                         |
|-----------------------------------------|---------|---------|-------------------------------------------------------------------------------------------------|
| `experiment_state_code`                 | `int`   | —       | Unique identifier code of the experiment state (the logical phase label)                        |
| `system_state_code`                     | `int`   | —       | The acquisition system's state-snapshot code for the phase (see below)                          |
| `state_duration_s`                      | `float` | —       | Seconds to maintain the state while executing the experiment                                    |
| `supports_trials`                       | `bool`  | `True`  | Whether trials are executed during this state; `False` means no trial-related processing occurs |
| `reinforcing_initial_guided_trials`     | `int`   | `0`     | Reinforcing trials after state onset that use guidance mode                                     |
| `reinforcing_recovery_failed_threshold` | `int`   | `0`     | Sequential reinforcing failures after which recovery guidance engages                           |
| `reinforcing_recovery_guided_trials`    | `int`   | `0`     | Guided reinforcing trials used in recovery guidance mode                                        |
| `aversive_initial_guided_trials`        | `int`   | `0`     | Aversive trials after state onset that use guidance mode                                        |
| `aversive_recovery_failed_threshold`    | `int`   | `0`     | Sequential aversive failures after which recovery guidance engages                              |
| `aversive_recovery_guided_trials`       | `int`   | `0`     | Guided aversive trials used in recovery guidance mode                                           |

`experiment_states` is a **dict**, not a list. Access states by string key
(`experiment_states["state_1"]`), never by integer index. `from_task_template` generates 1-indexed
keys (`state_1`, `state_2`, …), so the first state is `state_1`, not `state_0`.

### Mesoscope-VR `system_state_code` values

Mesoscope-VR accepts exactly two `system_state_code` hardware modes at runtime, drawn from its
`MesoscopeVRStates` enum (defined in `sollertia-experiment`):

| `system_state_code` | Mode   | Meaning                   |
|---------------------|--------|---------------------------|
| `1`                 | `REST` | The system's "rest" phase |
| `2`                 | `RUN`  | The system's "run" phase  |

Any other `system_state_code` (including the `0` that `from_task_template` seeds) is not a valid
Mesoscope-VR hardware mode and must be edited to `1` or `2` before the session can run. How the
runtime installs these modes is owned by `/mesoscope-vr-runtime`; this skill documents only the
accepted code values.

---

## The `from_task_template` builder

`MesoscopeExperimentConfiguration.from_task_template` is the contract builder
`create_experiment_from_vr_template_tool` dispatches to for Mesoscope-VR. It maps the template's
trial structures to runtime trial classes by trigger type, then seeds `state_count` default-valued
runtime states.

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

| Parameter                         | Type           | Default | Role                                                                                         |
|-----------------------------------|----------------|---------|----------------------------------------------------------------------------------------------|
| `template`                        | `TaskTemplate` | —       | The VR task template whose trial structures seed the configuration                           |
| `unity_scene_name`                | `str`          | —       | Stored verbatim; should match the template filename (caller's responsibility, not validated) |
| `state_count`                     | `int`          | `1`     | Number of default-valued runtime states to generate                                          |
| `default_reward_size_ul`          | `float`        | `5.0`   | `reward_size_ul` for every `interaction` (water-reward) trial                                |
| `default_reward_tone_duration_ms` | `int`          | `300`   | `reward_tone_duration_ms` for every `interaction` trial                                      |
| `default_puff_duration_ms`        | `int`          | `100`   | `puff_duration_ms` for every `occupancy_disarm` (gas-puff) trial                             |

`template` and `unity_scene_name` are positional/keyword; `state_count` and the three `default_*`
parameters all carry defaults so the generic creation tool can omit the system-specific generation
values (the registry contract assertion enforces this).

### State-seeding behavior

For each of the `state_count` states, the builder emits a `state_{n}` (1-indexed) `ExperimentState`
with:

- `experiment_state_code = n` (the 1-indexed state number)
- `system_state_code = 0` (a placeholder — **not** a valid Mesoscope-VR mode; must be set to `1` or
  `2` before running)
- `state_duration_s = 60` (the `_DEFAULT_STATE_DURATION_S` guidance constant)
- `supports_trials = True`

The guidance counters are populated **only for the trial classes that exist** in the seeded
`trial_structures`, mirroring the trial types present in the template:

| Guidance counter (reinforcing or aversive) | Seeded value when the matching trial type is present | Source constant                      |
|--------------------------------------------|------------------------------------------------------|--------------------------------------|
| `*_initial_guided_trials`                  | `3`                                                  | `_DEFAULT_INITIAL_GUIDED_TRIALS`     |
| `*_recovery_failed_threshold`              | `9`                                                  | `_DEFAULT_RECOVERY_FAILED_THRESHOLD` |
| `*_recovery_guided_trials`                 | `3`                                                  | `_DEFAULT_RECOVERY_GUIDED_TRIALS`    |

The `reinforcing_*` counters take these values only when a `MesoscopeWaterRewardTrial` is present in
`trial_structures` (otherwise `0`); the `aversive_*` counters only when a `MesoscopeGasPuffTrial` is
present (otherwise `0`). These constants are private to `experiment_configuration.py` and are not
overridable through the builder — only the per-trial `default_*` parameters above are.

---

## Related skills

| Skill                             | Relationship                                                                                                                                                                                                |
|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `assets:experiment-configuration` | Owns the generic create/write/validate/describe tooling, `EXPERIMENT_CONFIGURATION_REGISTRY` dispatch, and the system-agnostic `TriggerType` taxonomy; this skill is its Mesoscope-VR field-level companion |
| `assets:task-templates`           | Owns the `TaskTemplate` (with per-trial `trigger_type` and spatial `TrialStructure`) that seeds `from_task_template`                                                                                        |
| `/mesoscope-vr-runtime`           | Owns how the runtime loads and executes this configuration (state sequencing, guidance, trial decomposition, `system_state_code` installation)                                                              |
| `/mesoscope-vr`                   | Owns the Mesoscope-VR hardware composition and the `MesoscopeVRStates` enum that defines the `REST`/`RUN` codes                                                                                             |

---

## Verification checklist

You MUST verify any field, default, or enum cited from this skill against the
`sollertia-shared-assets` source before relying on it.

```text
- [ ] Field names, types, and defaults match mesoscope_vr/experiment_configuration.py exactly
- [ ] The three contract fields are reported as the whole schema (Mesoscope-VR adds no extra fields)
- [ ] Trial-class defaults cited as reward_size_ul=5.0, reward_tone_duration_ms=300, puff_duration_ms=100
- [ ] Only `interaction` and `occupancy_disarm` are described as mapped; the other three raise ValueError
- [ ] `system_state_code` values cited only as 1=REST and 2=RUN (0 is a placeholder, not valid)
- [ ] experiment_states described as a dict with 1-indexed string keys, never a list
- [ ] from_task_template defaults cited as state_count=1, ~60s duration, guidance 3/9/3
- [ ] Generic tool mechanics deferred to assets:experiment-configuration (not re-documented here)
- [ ] Runtime load/execute behavior deferred to /mesoscope-vr-runtime (not re-documented here)
- [ ] Template authoring deferred to assets:task-templates (not re-documented here)
```
