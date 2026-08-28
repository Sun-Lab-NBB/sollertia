# Extension guardrails

Documents the import-time checks that guard the `sollertia-shared-assets` registries, the verbatim text of every
failure they raise, the touch each message names, and the extension steps no check covers.

`registries.py` runs three assertions at the bottom of its module body: `_assert_registry_coverage()`,
`_assert_descriptor_contract()`, and `_assert_experiment_configuration_contract()`, in that order. The third folds in a
fourth check, `_experiment_builder_signature_gaps()`, which inspects the builder's signature and returns the gaps the
third reports. The package `__init__.py` imports `registries.py` directly, so a bare `import sollertia_shared_assets`
runs all of them and an unfinished extension fails before the `slsa mcp` server starts. Every failure is raised as a
`RuntimeError` through `ataraxis_base_utilities.console.error`.

---

## Message stems

Five distinct failure messages exist. Match a failure by its stem, then read the section below for the touch the
message names.

| Message stem                                                                                         | Raised by                                   | Touch it names                                                                         |
|------------------------------------------------------------------------------------------------------|---------------------------------------------|----------------------------------------------------------------------------------------|
| `<REGISTRY> is missing entries for`                                                                  | `_assert_registry_coverage`                 | The dispatch-registry entry for a new enum member                                      |
| `SYSTEM_SESSION_TYPES is missing entries for`                                                        | `_assert_registry_coverage`                 | The session-type frozenset of a new acquisition system                                 |
| `SYSTEM_SESSION_TYPES does not claim`                                                                | `_assert_registry_coverage`                 | The `SYSTEM_SESSION_TYPES` membership of a new session type                            |
| `DESCRIPTOR_REGISTRY descriptors are missing the required 'incomplete' field for`                    | `_assert_descriptor_contract`               | The `incomplete: bool = True` field on a new descriptor                                |
| `EXPERIMENT_CONFIGURATION_REGISTRY classes do not satisfy the experiment-configuration contract for` | `_assert_experiment_configuration_contract` | A contract field or the `from_task_template` builder on a new experiment configuration |

Every message names the offending enum **member name**, never its string value, and lists multiple offenders sorted
alphabetically by that name.

---

## `_assert_registry_coverage()`

Runs three separate checks and raises on the first one that fails.

### Dispatch-registry coverage

Pairs each dispatch registry with the enum that keys it and raises when `expected - actual` is non-empty:

| Registry                            | Expected key set                |
|-------------------------------------|---------------------------------|
| `DESCRIPTOR_REGISTRY`               | `frozenset(SessionTypes)`       |
| `HARDWARE_STATE_REGISTRY`           | `frozenset(AcquisitionSystems)` |
| `EXPERIMENT_CONFIGURATION_REGISTRY` | `frozenset(AcquisitionSystems)` |
| `SYSTEM_RAW_DATA_REGISTRY`          | `frozenset(AcquisitionSystems)` |
| `READ_ASSET_REGISTRY`               | `frozenset(ReadAssets)`         |
| `CREDENTIALS_FILE_REGISTRY`         | `frozenset(CredentialsTypes)`   |

Message, with the registry name and the comma-joined missing member names interpolated:

```text
{registry_name} is missing entries for {missing_names}. Every enum member must have a registered dispatch class. See the README's 'Adding New Session Types' / 'Adding New Acquisition Systems' / 'Adding a New Read Asset' sections for the full extension touch list.
```

Two properties of this check decide how far to trust it:

- It is one-directional. Only `expected - actual` is computed, so a stale registry key left behind by a removed enum
  member passes silently and only a missing key is reported.
- Its README pointer is fixed text shared by all six registries. A missing `CREDENTIALS_FILE_REGISTRY` entry therefore
  routes the reader to three README sections that do not cover credentials. Use the credentials recipe in
  [extension-recipes.md](extension-recipes.md) instead.

### Every acquisition system declares at least one session type

Triggers on a system absent from `SYSTEM_SESSION_TYPES` and on a system mapped to an empty frozenset, because an empty
set rejects every session type the system is asked to run:

```text
SYSTEM_SESSION_TYPES is missing entries for {missing_names}. Every acquisition system must declare the session types it can run. See the README's 'Adding New Acquisition Systems' section.
```

### Every session type is claimed by at least one system

Unions every frozenset in `SYSTEM_SESSION_TYPES` and reports the `SessionTypes` members no system claims, because an
unclaimed session type is rejected by every system:

```text
SYSTEM_SESSION_TYPES does not claim {orphan_names}. Every session type must be supported by at least one acquisition system. See the README's 'Adding New Session Types' section.
```

---

## `_assert_descriptor_contract()`

Reads `dataclasses.fields` on every value in `DESCRIPTOR_REGISTRY` and reports the session types whose descriptor
declares no field named `incomplete`. The check tests the field **name** only, so it accepts a descriptor that declares
`incomplete` with a different type or a different default. The `incomplete: bool = True` shape is therefore a convention
the recipes state rather than one the check enforces.

```text
DESCRIPTOR_REGISTRY descriptors are missing the required 'incomplete' field for {missing_names}. Every session descriptor must declare 'incomplete: bool = True'; the session-inspection tooling reads this field to decide whether a session's data is complete and eligible for unsupervised processing.
```

The consumer it protects is `read_descriptor_incomplete`, which the session-inspection tooling calls to decide whether
a session is complete and eligible for unsupervised processing.

---

## `_assert_experiment_configuration_contract()`

Walks `EXPERIMENT_CONFIGURATION_REGISTRY` and collects a gap list per registered class:

1. Each of the required fields `experiment_states`, `trial_structures`, and `unity_scene_name` that `dataclasses.fields`
   does not report on the class, listed by field name.
2. The literal gap `from_task_template builder` when the attribute is absent or is not callable.
3. Otherwise, every gap `_experiment_builder_signature_gaps()` returns for the builder.

Each offender is rendered as `{system.name} (missing {gaps})` with its gaps comma-joined, and the offenders are sorted
and joined with a semicolon before the message is raised:

```text
EXPERIMENT_CONFIGURATION_REGISTRY classes do not satisfy the experiment-configuration contract for {offender_names}. Every experiment configuration must declare the 'experiment_states', 'trial_structures', and 'unity_scene_name' fields and provide a 'from_task_template' classmethod that create_experiment_from_vr_template_tool can call with the 'template', 'unity_scene_name', and 'state_count' keyword arguments. See the README's 'Adding New Acquisition Systems' section.
```

---

## `_experiment_builder_signature_gaps()`

Inspects `from_task_template` against the contract parameters `template`, `unity_scene_name`, and `state_count`, which
are the exact keyword arguments `create_experiment_from_vr_template_tool` supplies. It raises nothing itself and
returns the gap strings the contract check reports, in this order:

| Gap string                                                       | Rule it enforces                                                                                                                      |
|------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `the '<name>' parameter on from_task_template`                   | The builder declares every contract parameter. A builder accepting `**kwargs` satisfies this rule for a parameter it does not name    |
| `keyword access to the '<name>' parameter on from_task_template` | No contract parameter is declared `POSITIONAL_ONLY`, since the tool passes all three by keyword                                       |
| `a default for the '<name>' parameter on from_task_template`     | Every parameter that is not a contract parameter carries a default, so the tool is able to omit the system-specific generation values |

Variadic parameters are exempt from the default rule, and contract parameters are exempt from it as well, which is why
a builder may declare `template` and `unity_scene_name` without defaults.

---

## What the checks do not catch

The import-time checks cover the six dispatch registries, the `SYSTEM_SESSION_TYPES` association, and the two
contract shapes above. Everything below passes a bare import and fails later, so each one is covered by a test, or by
an explicit tool call where no test package reaches it, rather than by a guardrail.

| Uncovered touch point                                                  | How the omission surfaces                                                                                                                                                                           | Where to cover it                                                               |
|------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| The trial-kind discriminator, all four edits                           | The trial class serializes but cannot be deserialized, and every configuration containing it raises at load                                                                                         | The experiment-configuration tests                                              |
| The `trial_structures` type union                                      | The class never surfaces in `list_supported_trial_types_tool`, which derives the vocabulary from that annotation                                                                                    | The experiment-configuration tests                                              |
| The trigger-to-trial mapping                                           | `from_task_template` raises "not mapped to a runtime trial class" for the unmapped trigger, which is also its intended unsupported-on-this-system signal                                            | The experiment-configuration tests                                              |
| The `occupancy_types` tuple                                            | `occupancy_duration_ms` is never required for the new mode, so a template that omits it loads and the trial reaches the acquisition runtime with `occupancy_duration_ms=None`                       | `tests/configuration/vr_configuration_test.py`                                  |
| The `_validate_zone_positions` trigger classification                  | Every legitimate template that uses a collision-style member raises a spurious trigger-zone or ordering geometry error                                                                              | `tests/configuration/vr_configuration_test.py`                                  |
| The `_SystemRawDataBuilder.build` contract                             | A bare `AttributeError` inside `SessionData._build_sub_dataclasses` at session create or load time                                                                                                  | A `SessionData.create()` smoke test for the system                              |
| `SESSION_TYPES_USING_VR_TASK` membership, the required-asset policy    | `required_raw_assets` omits `vr_configuration.yaml`, so session inspection passes a session whose VR snapshot is missing                                                                            | `tests/data_hierarchy/session_data_test.py`                                     |
| `SESSION_TYPES_USING_VR_TASK` membership, the creation gate            | `SessionData.create()` rejects every session of a listed type created without an `experiment_name`, because the VR task template is resolved from the experiment configuration's `unity_scene_name` | `tests/data_hierarchy/session_data_test.py`                                     |
| The session-record field through which a new session artifact resolves | The artifact has no session-resolved location, so its producer invents a path literal instead of reading `session_data.raw_data.<field>` or `session_data.processed_data.<field>`                   | `tests/data_hierarchy/session_data_test.py`                                     |
| The `list_processing_trackers_tool` description entry                  | `KeyError` on the first call of the tool, because its entry comprehension iterates every `ProcessingTrackers` member and indexes a `descriptions` dictionary from which the new member is absent    | A `list_processing_trackers_tool` call, since `interfaces/` has no test package |
| A stale registry key for a removed enum member                         | Nothing. The coverage check computes only `expected - actual`                                                                                                                                       | The registry tests                                                              |
