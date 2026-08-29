# Template validation and the two-repo field mirror

What `ConfigLoader` rejects before `CreateTask` writes anything, and the field-by-field inventory the manual C#/Python
template mirror must match. Read this when adding a template field, extending the validation surface, or diagnosing a
`create_task` failure whose message names a template field rather than an asset.

`ConfigLoader.LoadTemplate` deserializes with `UnderscoredNamingConvention` and `IgnoreUnmatchedProperties`, so an
unrecognized YAML key is dropped in silence and surfaces only as the downstream validation failure of whichever field
went missing.

---

## `ConfigLoader.LoadTemplate` order of checks

1. **File existence**, reported as `FileNotFoundException: Template file not found: <path>`.
2. **YAML deserialization**. A malformed document fails here, before the filename check, so a badly named file with
   broken YAML reports the parse error rather than the filename error.
3. **Template filename stem**. `ConfigLoader.SegmentNameComponentPattern` (`^[A-Za-z0-9_]+$`) is applied to the YAML
   filename without its extension: `Template filename '<name>' is invalid. Template names must contain only ASCII
   letters, digits, and underscores, because the generated segment prefab filename joins the template name and the trial
   name with a hyphen.`
4. **`ValidateTemplate`**, in the order below.

### `ValidateTemplate`

**Structure:** non-null parsed template (`FormatException: Failed to parse template file.`), non-empty `cues`, non-null
`vr_environment` (which runs `ValidateVrEnvironment`), non-empty `trial_structures`.

**Per cue**, in order:

| Check                | Rejection message                                                                          |
|----------------------|--------------------------------------------------------------------------------------------|
| Non-null entry       | `The cue entry at index <i> is empty. Every entry of the cues list must define a cue.`     |
| `name` present       | `A cue entry is missing the required 'name' field.`                                        |
| `name` pattern       | `Cue name '<name>' is invalid. Cue names must contain only ASCII letters, digits, and ...` |
| `code` in `[0, 255]` | `Cue '<name>' has invalid code <n>. Must be 0-255.`                                        |
| `code` unique        | `Duplicate cue code <n> found.`                                                            |
| `name` unique        | `Duplicate cue name '<name>' found.`                                                       |
| `length_cm` positive | `Cue '<name>' has invalid length <n>. Must be positive and finite.`                        |
| `texture` present    | `Cue '<name>' is missing required 'texture' field.`                                        |
| Texture file exists  | `Cue '<name>' references texture '<t>' but no file found at <path>.`                       |

The texture-file-existence check resolves `<template dir>/../Textures/<cue.texture>` and is **C#-only**, because the
Python `Cue` requires a non-empty filename but cannot see the Unity project.

**Per trial**, in order:

1. Non-null entry.
2. Trial-key pattern (`^[A-Za-z0-9_]+$`).
3. Non-empty `cue_sequence`.
4. Every cue name in the sequence resolves against the cue catalog.
5. `trigger_type` present and one of the five literals.
6. `occupancy_duration_ms` present for the three occupancy literals.
7. `occupancy_duration_ms` positive and finite whenever a value is supplied at all. Zero is a real duration and an
   invalid one, so it is rejected on every trial whatever its trigger type. A `null` is how a template says the field is
   unused.

**Cross-trial:** no two trials may share an identical cue sequence. The signature is the trial's cue names joined by a
single space, and a collision raises `Trials '<a>' and '<b>' share an identical cue sequence.` Identical sequences are
indistinguishable to the host's cue-stream decomposer, which would silently merge them. Use distinct cue codes (textures
may be shared) to multiplex visually identical cues.

**Transitions:** every trial with a non-empty `transitions` dict is checked three ways. Every key must name a defined
trial (`Trial '<name>' has a transition to unknown trial '<target>'.`). Every weight must be finite and within `[0.0,
1.0]` (`... with invalid probability <p>. Must be between 0.0 and 1.0.`). The weights must sum to `1.0` within
`ProbabilitySumTolerance` (`0.001`), reported as `Trial '<name>' transition probabilities sum to <s>, must be 1.0.` The
sum accumulates in `double` so a long target list does not drift into the tolerance through single-precision rounding.
`TrialStructure.HasTransitions` treats null and empty identically, and either means the runtime samples the next trial
uniformly over all defined trial names.

### `ValidateVrEnvironment`

| Field                   | Bound                                             | Rejection message                                               |
|-------------------------|---------------------------------------------------|-----------------------------------------------------------------|
| `segments_per_corridor` | integer `>= 1`                                    | `Invalid segments_per_corridor <n>. Must be at least 1.`        |
| `cm_per_unity_unit`     | positive and finite                               | `Invalid cm_per_unity_unit <n>. Must be positive and finite.`   |
| `corridor_spacing_cm`   | positive and finite                               | `Invalid corridor_spacing_cm <n>. Must be positive and finite.` |
| `cue_offset_cm`         | finite (may be negative)                          | `Invalid cue_offset_cm <n>. Must be finite.`                    |
| `padding_prefab_name`   | unvalidated here, resolved by the asset preflight | (see `ValidateHandAuthoredAssets`)                              |

Python's `VREnvironment` additionally rejects a non-`int` `segments_per_corridor` (`type(...) is not int`), a constraint
C# cannot express because the field is already typed `int`.

---

## The two-repo field mirror

A template field is a **two-repo mirror**. The YAML deserializer maps each underscored YAML key to the camelCase C#
member, so the two class definitions must stay in lockstep. Otherwise the field is silently dropped, or fails to parse,
at `create_task` time, far from the edit that caused it, in the other repo. There is no automated parity check.

| YAML key                           | C# member (`SL.Config`)                        | Python field (`vr_configuration`)      | Default / optionality                  |
|------------------------------------|------------------------------------------------|----------------------------------------|----------------------------------------|
| `cues`                             | `TaskTemplate.cues`                            | `TaskTemplate.cues`                    | Required, non-empty                    |
| `vr_environment`                   | `TaskTemplate.vrEnvironment`                   | `TaskTemplate.vr_environment`          | Required                               |
| `trial_structures`                 | `TaskTemplate.trialStructures`                 | `TaskTemplate.trial_structures`        | Required, non-empty in C# only         |
| (none, filename stem)              | `TaskTemplate.templateName`                    | (none)                                 | C#-only, derived from the filename     |
| `name`                             | `Cue.name`                                     | `Cue.name`                             | Required                               |
| `code`                             | `Cue.code`                                     | `Cue.code`                             | Required, uint8                        |
| `length_cm`                        | `Cue.lengthCm`                                 | `Cue.length_cm`                        | Required, positive finite              |
| `texture`                          | `Cue.texture`                                  | `Cue.texture`                          | Required                               |
| `corridor_spacing_cm`              | `VREnvironment.corridorSpacingCm`              | `VREnvironment.corridor_spacing_cm`    | `20.0` on both sides                   |
| `segments_per_corridor`            | `VREnvironment.segmentsPerCorridor`            | `VREnvironment.segments_per_corridor`  | `3` on both sides                      |
| `padding_prefab_name`              | `VREnvironment.paddingPrefabName`              | `VREnvironment.padding_prefab_name`    | `"Padding"` on both sides              |
| `cm_per_unity_unit`                | `VREnvironment.cmPerUnityUnit`                 | `VREnvironment.cm_per_unity_unit`      | `10.0` on both sides                   |
| `cue_offset_cm`                    | `VREnvironment.cueOffsetCm`                    | `VREnvironment.cue_offset_cm`          | `0.0` on both sides                    |
| `cue_sequence`                     | `TrialStructure.cueSequence`                   | `TrialStructure.cue_sequence`          | Required, non-empty                    |
| `stimulus_trigger_zone_start_cm`   | `TrialStructure.stimulusTriggerZoneStartCm`    | same, underscored                      | Required                               |
| `stimulus_trigger_zone_end_cm`     | `TrialStructure.stimulusTriggerZoneEndCm`      | same, underscored                      | Required                               |
| `stimulus_location_cm`             | `TrialStructure.stimulusLocationCm`            | same, underscored                      | Required                               |
| `show_stimulus_collision_boundary` | `TrialStructure.showStimulusCollisionBoundary` | same, underscored                      | `false` in C#, required in Python      |
| `trigger_type`                     | `TrialStructure.triggerType`                   | `TrialStructure.trigger_type`          | Required, one of five literals         |
| `occupancy_duration_ms`            | `TrialStructure.occupancyDurationMs`           | `TrialStructure.occupancy_duration_ms` | `null` / `None`, required on occupancy |
| `transitions`                      | `TrialStructure.transitions`                   | `TrialStructure.transitions`           | `null` / `None`                        |

Python's `TaskTemplate.__post_init__` carries no emptiness guard on `cues` or `trial_structures`. An empty `cues` list
is still rejected whenever a trial exists, indirectly, because every `cue_sequence` is required non-empty and each name
it holds must resolve against the cue catalog. An empty `trial_structures` dict passes `write_template_tool` and
`validate_template_tool` unchallenged and fails only later in Unity, with `No trial structures defined in template.`

`VREnvironment` also exposes two C#-only conversion accessors, `CorridorSpacingUnity` and `CueOffsetUnity`, that divide
their cm field by `cmPerUnityUnit`. They are not template fields and have no Python counterpart.

Mode-aware zone, boundary, and ordering validation lives on the Python side alone, in
`TaskTemplate._validate_zone_positions` (`sollertia-shared-assets`, `configuration/vr_configuration.py`). There,
`collision` validates only `stimulus_location`, and `occupancy_trigger` validates only the trigger zone. `interaction`,
`occupancy_disarm`, and `occupancy_arm` validate the zone, the boundary, and their ordering.
