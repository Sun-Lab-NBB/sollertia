---
name: mesoscope-vr-imaging-configuration
description: >-
  Documents the Mesoscope-VR two-photon donations sollertia-forgery dispatches through its acquisition-system
  registries: the raw calcium-imaging directory locator, the cross-recording session-type frozenset, and the two cindra
  configuration resolvers with their genotype-driven calcium-indicator selection. Use when resolving a session's cindra
  configuration, when a genotype fails to map to an indicator, or when changing an indicator-tuned imaging parameter.
user-invocable: false
---

# Mesoscope-VR imaging configuration

Documents the assets `mesoscope_vr/two_photon.py` donates to the acquisition-system-agnostic two-photon and forging
worker packages of sollertia-forgery. The donations answer three questions about a loaded Mesoscope-VR session. Where
its raw calcium-imaging data lives, whether its animals are tracked across recordings, and which cindra configuration
its animal's calcium indicator earns.

Every explicit parameter value the two configuration builders write is catalogued in one reference file, loaded on
demand:

- [references/cindra-parameters.md](references/cindra-parameters.md) lists every cindra section, field, and value the
  single-recording and multi-recording builders set, and marks the fields that depend on the calcium indicator.

---

## Scope

**Covers:**
- The three registry seams `two_photon.py` fills, and the agnostic accessor that reads each one
- `locate_two_photon_data`, the raw calcium-imaging directory locator, and the path it resolves
- `MESOSCOPE_MULTI_RECORDING_SESSION_TYPES` and why the frozenset exists alongside the multi-recording resolver
- `resolve_single_recording_configuration` and `resolve_multi_recording_configuration`, including the `None` return
- `_CalciumIndicator`, its two members and their transgenic lines, and the `_IndicatorParameters` bundle each earns
- `_GENOTYPE_INDICATOR_REGISTRY`, its three recognized keys, the normalization rule, and the unresolved-genotype error
- The surgery-record field the genotype is read from, and the error raised when the surgery metadata file is missing
- `_assert_indicator_coverage`, the import-time check that every indicator declares its indicator-dependent parameters
- The explicit-parameter design rule, the deploy-time fields left at cindra defaults, and the worker-count contract

**Does not cover:**
- The meaning of any cindra configuration section or parameter. Owned by `cindra:single-recording-configuration` and
  `cindra:multi-recording-configuration`.
- Batch orchestration of the two-photon pipeline that runs a resolved single-recording configuration. Owned by
  `forging:batch-processing`.
- The forging pipeline's multi-recording stage and its per-animal fan-out. Owned by `forging:dataset-forging`.
- The registry model, the donation protocols, and the import-time donor-coverage check over every seam. Owned by
  `forging:data-processing-design`.
- Alignment of the cindra fluorescence outputs onto the mesoscope-frame TTL pulses. Owned by
  `/mesoscope-vr-fluorescence-alignment`.
- The `SurgeryData` model and the tools that read it. Owned by `assets:data-assets`.

---

## Registry seams

`two_photon.py` fills three of the thirteen seams in `sollertia_forgery/registries.py`, each keyed on
`AcquisitionSystems.MESOSCOPE_VR`:

| Registry                                 | Mesoscope-VR donation                                             | Agnostic accessor                                                                                   |
|------------------------------------------|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `_TWO_PHOTON_DATA_REGISTRY`              | `locate_two_photon_data`                                          | `resolve_two_photon_data_locator`                                                                   |
| `_CINDRA_CONFIGURATION_REGISTRY`         | `_CindraConfigurationAsset` bundling both configuration resolvers | `resolve_single_recording_configuration_resolver`, `resolve_multi_recording_configuration_resolver` |
| `_MULTI_RECORDING_SESSION_TYPE_REGISTRY` | `MESOSCOPE_MULTI_RECORDING_SESSION_TYPES`                         | `resolve_multi_recording_session_types`                                                             |

The agnostic pipelines never import this module. Each resolves the donation registered for the acquisition system its
session or dataset records, then calls it.

---

## Raw calcium-imaging directory

`locate_two_photon_data(session)` returns `session.system_raw_data.mesoscope_data_path`, the session's
`raw_data/mesoscope_data` directory resolved by shared-assets' own builder. It checks nothing and raises nothing,
because a session that acquired no two-photon data still names the path it would have used. The two-photon pipeline
screens the directory itself and hands the resolved path to cindra as `file_io.data_path`.

---

## Cross-recording session types

`MESOSCOPE_MULTI_RECORDING_SESSION_TYPES` is `frozenset({SessionTypes.MESOSCOPE_EXPERIMENT})`, a single member.
Cross-recording cell tracking needs calcium imaging, which only an experiment session records, so every other session
type resolves no multi-recording configuration.

The frozenset and `resolve_multi_recording_configuration` answer the same question by different means. The resolver
needs a loaded session and therefore that session's source data, while the frozenset needs only a recorded session
type. The forging pipeline reads the answer from the dataset itself, through
`SessionTypes(dataset.session_type) in resolve_multi_recording_session_types(...)`, so a dataset keeps growing while
part of its source data lives elsewhere.

---

## Calcium indicators and their parameters

`_CalciumIndicator` enumerates the two indicators for which the Mesoscope-VR cindra configurations are tuned:

| Member     | Value      | Transgenic line                                                                                                                                |
|------------|------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| `GCAMP6F`  | `GCaMP6f`  | The Thy1-GCaMP6f transgenic line (GP5.17)                                                                                                      |
| `JGCAMP8S` | `jGCaMP8s` | The in-house cross of a jGCaMP8s reporter line and a CaMKII-Cre driver line (GCaMP8s x CamKIICre), the slow-decay member of the jGCaMP8 family |

`_IndicatorParameters` is a frozen, slotted dataclass carrying the three values that depend on the indicator. Each
field names the cindra field it fills:

| Field                   | cindra field                               | Read by                                                      |
|-------------------------|--------------------------------------------|--------------------------------------------------------------|
| `tau`                   | `main.tau`                                 | The single-recording builder. The OASIS AR(1) decay constant |
| `neuropil_coefficient`  | `spike_deconvolution.neuropil_coefficient` | Both builders                                                |
| `probability_threshold` | `roi_selection.probability_threshold`      | The multi-recording builder                                  |

`_INDICATOR_PARAMETERS` maps each indicator to its bundle:

| Indicator  | `tau` | `neuropil_coefficient` | `probability_threshold` |
|------------|-------|------------------------|-------------------------|
| `GCAMP6F`  | `0.4` | `0.7`                  | `0.85`                  |
| `JGCAMP8S` | `0.7` | `0.8`                  | `0.80`                  |

`tau` is in seconds. Every parameter outside this table is a fixed literal, identical for both indicators.

---

## Genotype registry and normalization

`_GENOTYPE_INDICATOR_REGISTRY` maps three normalized genotype strings to their indicator:

| Normalized key        | Indicator  |
|-----------------------|------------|
| `gp5.17`              | `GCAMP6F`  |
| `gp5.17 (hemi)`       | `GCAMP6F`  |
| `gcamp8s x camkiicre` | `JGCAMP8S` |

`_resolve_calcium_indicator(genotype)` normalizes before matching, with
`re.sub(pattern=r"\s+", repl=" ", string=genotype.strip().casefold())`. Normalization casefolds the string, strips
surrounding whitespace, and collapses every internal whitespace run to one space. The normalized string is then
matched exactly, so every jGCaMP8 variant needs its own registry key before it resolves.

Zygosity qualifiers such as the `(hemi)` in `GP5.17 (hemi)` are preserved and carry their own key, so a hemizygous
line resolves to the same indicator as the homozygous line while the surgery metadata keeps the distinction. Only the
two indicators used with the reference Mesoscope-VR system are recognized, so an unrecognized genotype raises
immediately and the caller resolves the mismatch before any imaging data is processed.

An unmatched genotype raises `ValueError`:

```text
Unable to resolve the calcium indicator for the genotype '{genotype}'. The genotype normalized to '{normalized_genotype}', which does not match a recognized indicator. The recognized genotypes are: {recognized_genotypes}.
```

`recognized_genotypes` is `", ".join(sorted(_GENOTYPE_INDICATOR_REGISTRY))`, which renders the three keys in sorted
order as `gcamp8s x camkiicre, gp5.17, gp5.17 (hemi)`.

---

## Where the genotype comes from

`_read_session_genotype(session)` reads `session.raw_data.surgery_metadata_path`, the session's
`raw_data/surgery_metadata.yaml` file (`RawDataFiles.SURGERY_METADATA`), and returns the `subject.genotype` field of
the `SurgeryData` record it loads through `SurgeryData.from_yaml`. The file is checked with `is_file()` before the
load, and its absence raises `FileNotFoundError`:

```text
Unable to resolve the cindra configuration for session '{session.session_name}'. No surgery metadata file was found at '{surgery_metadata_path}'. The animal's genotype is read from this file to select the cindra configuration.
```

---

## Import-time indicator coverage

`_assert_indicator_coverage()` runs at module import. It collects the sorted `name` of every `_CalciumIndicator`
member absent from `_INDICATOR_PARAMETERS` and raises `RuntimeError` naming them, so an indicator added without its
parameters fails at import rather than at the resolution of some later session:

```text
Unable to validate calcium-indicator coverage. Every _CalciumIndicator member must declare its indicator-dependent parameters in _INDICATOR_PARAMETERS, but the following members do not: {uncovered_members}.
```

Adding a third indicator therefore means three edits in one commit. A `_CalciumIndicator` member, its
`_INDICATOR_PARAMETERS` entry, and at least one `_GENOTYPE_INDICATOR_REGISTRY` key that resolves to it.

---

## Configuration construction

Both builders write every cindra parameter of every section they construct explicitly, so the configuration is
decoupled from cindra's evolving defaults. A cindra release that changes a default therefore changes nothing about how
a Mesoscope-VR session is processed.

| Builder                                 | cindra sections written                                                                                                                            |
|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| `_build_single_recording_configuration` | `main`, `file_io`, `registration`, `one_photon_registration`, `nonrigid_registration`, `roi_detection`, `signal_extraction`, `spike_deconvolution` |
| `_build_multi_recording_configuration`  | `recording_io`, `roi_selection`, `diffeomorphic_registration`, `roi_tracking`, `signal_extraction`, `spike_deconvolution`                          |

Only two fields per configuration depend on the indicator. `main.tau` and `spike_deconvolution.neuropil_coefficient` for
single-recording, `roi_selection.probability_threshold` and `spike_deconvolution.neuropil_coefficient` for
multi-recording. Every value both builders write is listed in
[references/cindra-parameters.md](references/cindra-parameters.md).

The deploy-time fields are the deliberate exception. They stay at their cindra defaults, because the pipeline that
runs the configuration overrides them with the locations it resolved:

| Configuration    | Field left at its cindra default     | Overridden by                                                                              |
|------------------|--------------------------------------|--------------------------------------------------------------------------------------------|
| Single-recording | `file_io.data_path`                  | The two-photon pipeline, with the path `locate_two_photon_data` resolved                   |
| Single-recording | `file_io.output_path`                | The two-photon pipeline, with `session.processed_data_path`                                |
| Multi-recording  | `recording_io.recording_directories` | The forging pipeline, with each dataset session's `processed_data.cindra_data_path`        |
| Multi-recording  | `recording_io.dataset_name`          | The forging pipeline, with `multi_recording_dataset_name(animal_id=..., dataset_name=...)` |
| Both             | The `runtime` settings               | The running pipeline, which sets `runtime.display_progress_bars` from its own preference   |

cindra takes the worker count as a call argument, so no configuration field carries it. Look for a worker setting in
the call that runs a stage, never in the materialized configuration file.

---

## Resolution workflow

`resolve_single_recording_configuration(session)` returns a `SingleRecordingConfiguration`. It calls
`_read_session_genotype`, then `_build_single_recording_configuration`, which resolves the indicator and looks up its
parameter bundle. The two-photon pipeline reaches it through
`resolve_single_recording_configuration_resolver(system=session.acquisition_system)`, applies the two deploy-time
paths and the progress-bar preference, and materializes the result into the session's cindra directory.

`resolve_multi_recording_configuration(session)` returns `MultiRecordingConfiguration | None`. It returns `None`
before any file is read whenever `session.session_type` is not `SessionTypes.MESOSCOPE_EXPERIMENT`, so a training
session never touches its surgery metadata and never raises for a genotype the registry does not recognize. The
forging pipeline calls the resolver once per animal, with `animal_sessions[0]`, the animal's first dataset session,
and skips the whole animal when the call returns `None`.

Both resolvers raise the same two errors when they do read. `FileNotFoundError` for a missing surgery metadata file,
and `ValueError` for a genotype that no registry key matches. Both surface through `console.error`.

---

## Related skills

The `cindra:` entries below resolve through the cindra marketplace. Every other entry resolves inside the sollertia
marketplace.

| Skill                                   | Relationship                                                                               |
|-----------------------------------------|--------------------------------------------------------------------------------------------|
| `cindra:single-recording-configuration` | Owns the `SingleRecordingConfiguration` sections and parameter semantics these values fill |
| `cindra:multi-recording-configuration`  | Owns the `MultiRecordingConfiguration` sections and parameter semantics these values fill  |
| `forging:batch-processing`              | Runs the two-photon pipeline that materializes the single-recording configuration          |
| `forging:dataset-forging`               | Runs the multi-recording stage that materializes the per-animal configuration              |
| `forging:data-processing-design`        | Owns the registry model these three seams plug into                                        |
| `forging:processing-input-format`       | Owns the raw-data prerequisites a session meets before the two-photon pipeline runs        |
| `/mesoscope-vr-fluorescence-alignment`  | Consumes the cindra outputs these configurations produce                                   |
| `/mesoscope-vr-dataset-assembly`        | Owns the admission policy that gates a session into the dataset forging tracks             |
| `assets:data-assets`                    | Owns the `SurgeryData` record whose `subject.genotype` field selects the indicator         |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Registry seams:
- [ ] locate_two_photon_data resolves raw_data/mesoscope_data and reaches callers through _TWO_PHOTON_DATA_REGISTRY
- [ ] Both configuration resolvers reach the agnostic pipelines through _CINDRA_CONFIGURATION_REGISTRY
- [ ] MESOSCOPE_MULTI_RECORDING_SESSION_TYPES holds SessionTypes.MESOSCOPE_EXPERIMENT and nothing else
- [ ] resolve_multi_recording_configuration returns None for every session type other than mesoscope experiment

Indicator resolution:
- [ ] The genotype was read from the subject.genotype field of the session's raw_data/surgery_metadata.yaml
- [ ] The genotype was casefolded, stripped, and whitespace-collapsed before an exact registry match
- [ ] The three recognized keys were quoted exactly: gp5.17, gp5.17 (hemi), gcamp8s x camkiicre
- [ ] GCAMP6F carries tau 0.4, neuropil_coefficient 0.7, probability_threshold 0.85
- [ ] JGCAMP8S carries tau 0.7, neuropil_coefficient 0.8, probability_threshold 0.80
- [ ] An unrecognized genotype raises ValueError naming the normalized string and the recognized genotypes
- [ ] A missing surgery metadata file raises FileNotFoundError naming the session and the resolved path
- [ ] A new indicator was added as an enum member, an _INDICATOR_PARAMETERS entry, and a genotype key together

Configuration construction:
- [ ] Every cindra parameter quoted was read from references/cindra-parameters.md or the builder itself
- [ ] Only the deploy-time fields and the runtime settings were described as left at cindra defaults
- [ ] The worker count was sought in the call that runs a stage, not in a configuration field
- [ ] allow_overlap is False for single-recording extraction and True for multi-recording extraction
```
