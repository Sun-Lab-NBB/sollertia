---
name: datasets
description: >-
  Discovers, inspects, reads, writes, and validates forged datasets and their column descriptions via the
  sollertia-shared-assets MCP server. Owns the DatasetData marker, the two-level dataset layout, the
  data_descriptions.feather contract, and the seven slsa dataset tools. Use when locating datasets under a data root,
  auditing a dataset's structural completeness, repairing a corrupted dataset.yaml, or resolving a forged session's or
  animal's artifact paths.
user-invocable: false
---

# Sollertia forged datasets

Reads, inspects, and validates the forged-dataset container: the `dataset.yaml` marker, the two-level
`<dataset>/<animal>/<session>` tree it describes, and the `data_descriptions.feather` companion that gives every
assembled column its meaning. Uses the `slsa mcp` MCP server. This skill is the **exclusive** owner of
`discover_datasets_tool`, `inspect_datasets_tool`, `read_dataset_data_tool`, `write_dataset_data_tool`,
`describe_dataset_data_schema_tool`, `read_dataset_column_descriptions_tool`, and `validate_dataset_descriptions_tool`.
No other skill in the marketplace may document or call these seven tools.

The dataset layer is system-agnostic. It records which sessions a dataset holds and where their assembled artifacts
live, and it treats the contents of those artifacts as opaque.

---

## Scope

**Covers:**
- Discovering datasets under a data root, optionally narrowed to one project (`discover_datasets_tool`)
- Auditing dataset structural completeness (`inspect_datasets_tool`), including per-session and per-animal artifact
  inventories and the `issues` list
- Reading and repairing the `dataset.yaml` marker (`read_dataset_data_tool`, `write_dataset_data_tool`) and its schema
  (`describe_dataset_data_schema_tool`)
- The `data_descriptions.feather` companion: its two-column `pl.String` schema, its one-directional verification
  contract, `read_dataset_column_descriptions_tool`, and `validate_dataset_descriptions_tool`
- The two-level `<dataset>/<animal>/<session>` layout and the per-session and per-animal artifact rosters
- Resolving forged-copy paths for downstream skills (`DatasetSession.data_path`, `.descriptor_path`,
  `.vr_configuration_path`, `.experiment_configuration_path`, `DatasetAnimal.surgery_path`)
- The library-level mutators no MCP tool exposes (`DatasetData.create()`, `add_sessions()`, `remove_animal()`), for
  agents writing downstream Python

**Does not cover:**
- Composing or growing a dataset on disk, which applies an admission policy this skill's write tool does not (see
  `forging:dataset-definition`, `define_forging_dataset_tool`)
- Forging job state and planning (see `forging:dataset-definition` and `forging:dataset-forging`)
- What the described columns actually contain (see `forging:dataset-forging` for the column-description contract
  and, for Mesoscope-VR, `mesoscope:mesoscope-vr-processing-schema`)
- Session-level markers, descriptors, and hardware state (see `/session-data`, `/session-descriptors`,
  `/session-hardware-state`)
- The project and animal hierarchy that holds datasets (see `/project-hierarchy`)

---

## Anatomy of a forged dataset

A forged dataset aggregates data acquisition sessions of the **same session type**, recorded by the **same acquisition
system**, across different animals. The sessions are chosen upstream by a filtering step, and the container stores the
assembled data alone, together with the frozen per-session and per-animal metadata each assembled session needs to be
interpretable on its own.

```text
<project>/
└── <dataset>/                                 # dataset root, a sibling of the project's animal directories
    ├── dataset.yaml                           # DatasetData marker (THIS SKILL)
    ├── data_descriptions.feather              # per-dataset column-description companion (THIS SKILL)
    └── <animal>/
        ├── surgery_metadata.yaml              # forged copy of the animal's surgery record (/data-assets)
        └── <session>/
            ├── data.feather                   # assembled per-session data (forging:dataset-forging)
            ├── session_descriptor.yaml        # forged copy of the session descriptor (/session-descriptors)
            ├── vr_configuration.yaml          # forged copy, conditional on the dataset's session type
            └── experiment_configuration.yaml  # forged copy, conditional on the session's experiment
```

Session nesting is exactly two levels below the dataset root, so a session's directory is always
`<dataset>/<animal>/<session>`. Dataset directories live inside the project hierarchy as siblings of the animal
directories, and the shared hierarchy walkers recognize, skip, or count them by the `dataset.yaml` marker name alone.

### Per-session and per-animal artifacts

`inspect_datasets_tool` reports every artifact below for every session and animal the marker names, each carrying a
`present` flag and a `required` flag. Only an artifact that is both required and absent becomes an entry in `issues`.

| Artifact                        | Level   | Required                                                            |
|---------------------------------|---------|---------------------------------------------------------------------|
| `data.feather`                  | session | Always                                                              |
| `session_descriptor.yaml`       | session | Always                                                              |
| `vr_configuration.yaml`         | session | When the dataset's session type runs the VR task, see below         |
| `experiment_configuration.yaml` | session | Never, because the marker does not carry the per-session experiment |
| `surgery_metadata.yaml`         | animal  | Never                                                               |

Whether `vr_configuration.yaml` is required is decided by registry membership, not by the file itself: the dataset's
`session_type` is tested against `SESSION_TYPES_USING_VR_TASK`, and only a session type inside that frozen set makes the
snapshot's absence a defect. The rule is generic and the membership is registry-owned, so hand off to
`mesoscope:mesoscope-vr-dataset-assembly` for which session types belong to it today.

`surgery_metadata.yaml` is reported at the animal level and is never required, because the forging pipeline skips the
surgery snapshot when the animal's latest source session carries none. Its absence therefore describes the source data
rather than a defect in the dataset. A missing animal directory likewise raises no issue, while a missing session
directory does.

---

## The dataset marker

`DatasetData` is the dataclass behind `dataset.yaml`. It declares six fields, five of which reach the file.

| Field                | Type                         | Default  | Written to `dataset.yaml`               |
|----------------------|------------------------------|----------|-----------------------------------------|
| `name`               | `str`                        | required | Yes                                     |
| `project`            | `str`                        | required | Yes                                     |
| `session_type`       | `str \| SessionTypes`        | required | Yes, as the enum's string value         |
| `acquisition_system` | `str \| AcquisitionSystems`  | required | Yes, as the enum's string value         |
| `sessions`           | `tuple[DatasetSession, ...]` | `()`     | Yes, each entry as `session` + `animal` |
| `dataset_data_path`  | `Path`                       | `Path()` | No, excluded from serialization         |

`__post_init__` converts `session_type` and `acquisition_system` through their enums unconditionally, raising
`ValueError` when either identifier falls outside the platform vocabulary. That conversion is the only vocabulary gate
applied to an instance rehydrated from disk, since no other method re-resolves either field.

### Session and animal records

`DatasetSession` carries `session`, `animal`, and `session_path`, and serializes only the first two, because
`session_path` is excluded from the marker the same way `dataset_data_path` is. `DatasetAnimal` carries `animal` and
`animal_path` and never reaches the marker at all, since the dataset derives its animals from the session list on demand
and returns them sorted by identifier. `DatasetData` is a plain mutable dataclass, while `DatasetSession` and
`DatasetAnimal` are frozen and slotted, so assigning to one of their fields raises `FrozenInstanceError`.

Both excluded path fields are re-derived by `DatasetData.load` from the marker's own on-disk location, which is what
makes a dataset portable: copying the tree to another machine changes every resolved path and nothing inside the file.
The payload `read_dataset_data_tool` returns consequently holds no path of any kind.

| Property                                       | Resolves to                                 |
|------------------------------------------------|---------------------------------------------|
| `DatasetSession.data_path`                     | `<dataset>/<animal>/<session>/data.feather` |
| `DatasetSession.descriptor_path`               | `.../session_descriptor.yaml`               |
| `DatasetSession.vr_configuration_path`         | `.../vr_configuration.yaml`                 |
| `DatasetSession.experiment_configuration_path` | `.../experiment_configuration.yaml`         |
| `DatasetAnimal.surgery_path`                   | `<dataset>/<animal>/surgery_metadata.yaml`  |
| `DatasetData.descriptions_path`                | `<dataset>/data_descriptions.feather`       |

Every property is a pure path join with no existence check, so a caller reading a conditional artifact tests
`.is_file()` first, or reads the `present` flag off an `inspect_datasets_tool` report.

---

## Column descriptions

`data_descriptions.feather` is written once at the dataset root, not per session and not per animal, because every
session in a dataset shares one data format. It is an Arrow IPC table of exactly two `pl.String` columns named `column`
and `description`, and it is the interpretation contract for every session the dataset holds.

`DatasetData.create()` is its only author, and it writes the companion **before** the marker, so a dataset that
discovery can find always carries one. The `slsa mcp` server exposes no tool that writes or repairs the companion, and
`write_dataset_data_tool` writes the marker alone. A dataset whose companion is genuinely lost is rebuilt by the forging
pipeline rather than patched from here.

Verification is one-directional. A described column that no session emits passes, because columns are emitted
conditionally, while a column written into a session's `data.feather` with no matching description is a violation. The
check reads only the Arrow footer schema of each session's feather, so no session data is materialized, and it
aggregates every offending column and the sessions that emit it into a single message rather than aborting on the first.
Run it once the dataset is fully composed, meaning every session's `data.feather` is on disk.

The two description tools disagree deliberately about a missing companion, and the disagreement is the point:

| Tool                                    | Companion missing                    | Path does not resolve to a dataset |
|-----------------------------------------|--------------------------------------|------------------------------------|
| `read_dataset_column_descriptions_tool` | Error envelope                       | Error envelope                     |
| `validate_dataset_descriptions_tool`    | Success envelope with `valid: false` | Error envelope                     |

The validator treats a missing companion and missing session data as verification verdicts, so its `issues` list is
where a completeness answer belongs. The reader treats the same condition as a failed read, since it has nothing to
return.

---

## MCP tool surface

| Tool                                    | Input                                            | Purpose                                           |
|-----------------------------------------|--------------------------------------------------|---------------------------------------------------|
| `discover_datasets_tool`                | `root_directory`, optional `project`             | Walks a data root for markers and loads each one  |
| `inspect_datasets_tool`                 | `dataset_paths: list[str]`                       | Reports structural completeness per dataset       |
| `read_dataset_data_tool`                | `file_path` to the marker                        | Reads the marker through the `DatasetData` schema |
| `write_dataset_data_tool`               | `file_path`, `dataset_data_payload`, `overwrite` | Repairs a corrupted marker                        |
| `describe_dataset_data_schema_tool`     | none                                             | Returns the `DatasetData` schema                  |
| `read_dataset_column_descriptions_tool` | `dataset_path`, the root or the marker           | Returns the whole column-to-description mapping   |
| `validate_dataset_descriptions_tool`    | `dataset_path`, the root or the marker           | Verifies every emitted column is described        |

The response envelope every tool on this server returns is documented in the `## Response contract` section of
`/assets-mcp-environment-setup`. None of the seven takes a `host` parameter, so all of them read the local filesystem.

`read_dataset_data_tool` takes the marker **file** path. `read_dataset_column_descriptions_tool` and
`validate_dataset_descriptions_tool` take either the dataset root or the marker and resolve the root from both forms.
Neither resolver searches recursively, so a project root holding one dataset is rejected rather than walked.
`inspect_datasets_tool` accepts the same two forms in every element of its list.

### Discovery

`discover_datasets_tool` resolves the root, optionally narrows the scan to `<root>/<project>`, indexes every
`dataset.yaml` under it, and loads each marker. Its `counts` vocabulary is `{ok, error}` and reports whether each marker
**loads**, saying nothing about whether the dataset it describes is complete. Both keys are always present, including at
zero. A marker that fails to load becomes a four-key entry carrying `dataset_path`, `marker_path`, `status: "error"`,
and `error_detail`. A loaded marker carries the dataset's identity, its session and animal counts, its sorted animal
list, and a `has_descriptions` flag.

Failure modes: a `root_directory` that does not exist or is not a directory, a `project` filter naming a directory
absent from the root, and an unreadable directory encountered mid-scan. Each returns the error envelope.

`dataset_paths` is the sorted list of roots whose markers loaded, and it is built to be chained. Pass it straight into
`inspect_datasets_tool` here, or into sollertia-forgery's `generate_dataset_state_tool` and `plan_dataset_jobs_tool`
through `forging:dataset-definition`. This tool expands the per-project `dataset_count` aggregate that
`get_data_root_overview_tool` reports, which `/project-hierarchy` owns.

### Inspection

`inspect_datasets_tool` produces the structural report. Its `counts` vocabulary is `{complete, incomplete, error}` and
is not interchangeable with discovery's. `complete` and `incomplete` come from whether a loaded dataset's `issues` list
is empty, while `error` counts the paths that never resolved or never loaded. An entry in that third group holds the
**raw input string** rather than a resolved path, so a typo is visible in the report.

Each loaded report carries `dataset_path`, `marker_path`, an `identity` block, `status`, `session_count`,
`animal_count`, a `descriptions` block, the per-animal `animals` list, the per-session `sessions` list, and `issues`.
`issues` is composed in a fixed order: the description-companion problems first, then each missing session directory,
then each required session artifact that is absent.

### Reading, repairing, and describing the marker

`read_dataset_data_tool` returns the error envelope when the file is absent and when its content fails to load as
`DatasetData`, and it reports the five serialized keys alone.

`write_dataset_data_tool` is **repair only**. It applies no admission policy, creates none of the session directories
its payload names, and writes a payload that names a session the hierarchy does not hold, or that repeats an animal and
session pair, exactly as supplied. Its validation covers the session-type and acquisition-system vocabulary and nothing
else. Its `overwrite` parameter is keyword-only and defaults to `True`, inverting the underlying helper's default, so a
marker is replaced unless `overwrite=False` is passed deliberately. Run `inspect_datasets_tool` afterwards to confirm
the repaired marker matches the tree it describes.

What a `write_*` tool validates on this server, and why every amendment MUST be a read-mutate-write of the complete
record, is documented in the `## Response contract` section of `/assets-mcp-environment-setup`.

The payload shape `describe_dataset_data_schema_tool` returns is documented in that same section. Its one addition is a
`nested_classes` mapping, which resolves `DatasetSession` out of the `sessions` tuple annotation. `DatasetAnimal` is
absent from that mapping, since no `DatasetData` field annotation references it. Use `inspect_datasets_tool` for
animal-level facts.

---

## Workflows

### Locating datasets under a data root

1. **Verify prerequisites:** the MCP server is connected (else `/assets-mcp-environment-setup`) and the working
   directory is configured (else `/working-directory`).
2. **Scan the root**, narrowing to one project when the project is known:
   ```text
   discover_datasets_tool(root_directory="<absolute data root>", project="<project name>")
   ```
3. **Read `counts`** to learn how many markers loaded, and read `error_detail` on each failed entry before doing
   anything else with that dataset.
4. **Chain `dataset_paths`** into inspection or into the forging planning tools rather than reassembling paths by hand.

### Auditing a dataset's structural completeness

1. **Inspect one dataset or a batch:**
   ```text
   inspect_datasets_tool(dataset_paths=["<absolute dataset root or marker>"])
   ```
2. **Read `issues` first.** An empty list means every required artifact for every session the marker names is present.
3. **Check the description block** through `descriptions.present` and `descriptions.column_count`.
4. **Interpret a `vr_configuration.yaml` issue** against the dataset's session type before reporting it, since the
   requirement is registry-gated.
5. **Hand off** to `forging:dataset-definition` when sessions are missing from the container, or to
   `forging:dataset-forging` when the container is right and the assembled feathers are not yet produced.

### Repairing a corrupted marker

1. **Fetch the schema** with `describe_dataset_data_schema_tool()` and read `required` off each field.
2. **Recover the current content**, through `read_dataset_data_tool(file_path=…)` when the file still loads, or through
   `inspect_datasets_tool` and a directory listing when it does not.
3. **Write the complete record**, carrying every field:
   ```text
   write_dataset_data_tool(file_path="<absolute>/dataset.yaml", dataset_data_payload={...})
   ```
4. **Re-read and diff** the result against the payload you intended.
5. **Re-inspect** with `inspect_datasets_tool` to confirm the marker and the directory tree agree.

Adding sessions to a dataset is not a repair. Route it to `forging:dataset-definition`.

### Verifying the column descriptions

```text
validate_dataset_descriptions_tool(dataset_path="<absolute dataset root>")
read_dataset_column_descriptions_tool(dataset_path="<absolute dataset root>")
```

Call the validator for the verdict, since it reports a missing companion and missing session data as `valid: false` with
one aggregated message in `issues`. Call the reader when the mapping itself is what you need, and expect the error
envelope from it when the companion is absent. A pass returns `summary` with `session_count` and
`described_column_count`, and `summary` and `issues` never appear together.

### Resolving a forged session's or animal's artifacts

Take the `sessions[*].artifacts[*].path` and `animals[*].surgery_metadata.path` values from an `inspect_datasets_tool`
report, then hand the path to the skill that owns the file. `/session-descriptors` owns the descriptor and
`/data-assets` owns the surgery record. `/experiment-configuration` owns the experiment snapshot, and `/task-templates`
owns the VR configuration snapshot. Read the assembled `data.feather` itself through `forging:dataset-forging`.

---

## Library API

Three `DatasetData` members mutate a dataset on disk, and no MCP tool exposes any of them. Python code running where the
dataset lives reaches them directly.

| Member                        | Effect                                                                                   |
|-------------------------------|------------------------------------------------------------------------------------------|
| `DatasetData.create(...)`     | Builds the whole tree, writes the description companion, then writes the marker last     |
| `instance.add_sessions(...)`  | Creates the new session directories and rewrites the marker, rolling back on any failure |
| `instance.remove_animal(...)` | Deletes one animal's directory with everything under it and rewrites the marker          |

`create()` screens the name, the session type, the acquisition system, the session list, every animal and session
identifier, and every column-description entry before it creates a single directory, and it refuses a destination that
already exists. `add_sessions()` refuses a session the dataset already holds and a pair repeated inside one request, and
it enforces the structural invariants alone, leaving the question of whether a session belongs in the dataset to its
caller. `remove_animal()` unlinks an animal directory that is a symlink in place, so the tree it targets stays whole,
and it verifies the directory is gone before rewriting the marker.

The read-only helpers need no MCP round trip either: `animals`, `get_animal()`, `get_sessions_for_animal()`,
`get_session()`, `column_descriptions()`, `get_column_description()`, and `verify_data_descriptions()`.
`get_sessions_for_animal()` returns an empty tuple for an unknown animal, while `get_animal()` and `get_session()` raise
`ValueError`.

Composing or growing a dataset through these mutators skips the admission policy `forging:dataset-definition` applies,
so prefer `define_forging_dataset_tool` for any dataset a forging run will consume.

---

## Related skills

| Skill                                     | Relationship                                                                                         |
|-------------------------------------------|------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`           | Run first if the MCP server is not connected, and owns the response, write, and schema contracts     |
| `/working-directory` | Required prerequisite, bootstraps the working directory and data root that anchor path resolution |
| `/project-hierarchy`                      | Owns `get_data_root_overview_tool`, whose per-project `dataset_count` this skill's discovery expands |
| `/session-discovery` | Filters the candidate sessions from which a dataset is defined |
| `/session-data` | Owns the source session marker from which each forged session copy was produced |
| `/session-descriptors`                    | Owns the schema behind each forged `session_descriptor.yaml` copy                                    |
| `/data-assets`                            | Owns the surgery record behind each forged per-animal `surgery_metadata.yaml` copy                   |
| `/experiment-configuration`               | Owns the schema behind each forged `experiment_configuration.yaml` copy                              |
| `/task-templates`                         | Owns the VR task template behind each forged `vr_configuration.yaml` copy                            |
| `/library-extension` | Recipe for adding the `SessionTypes` and `AcquisitionSystems` members against which the marker validates |
| `forging:dataset-definition`              | Composes and grows datasets under an admission policy, and reports their forging job state           |
| `forging:dataset-forging`                 | Runs the per-session `data.feather` assembly whose output this container holds                       |
| `forging:processing-results`              | Owns the processed-data output layout and how a completed stage is verified                          |
| `mesoscope:mesoscope-vr-processing-schema` | Owns the Mesoscope-VR column roster and the current `SESSION_TYPES_USING_VR_TASK` membership        |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] discover_datasets_tool was called before inspection, and its dataset_paths list was chained into
      inspect_datasets_tool rather than reassembled by hand
- [ ] The counts vocabulary was read against the tool that produced it (ok/error for discovery,
      complete/incomplete/error for inspection)
- [ ] write_dataset_data_tool was only invoked to repair a corrupted marker, and its payload carried every field
- [ ] inspect_datasets_tool was called after every write_dataset_data_tool call, and the marker was confirmed to
      match the directory tree it describes
- [ ] Adding sessions to a dataset was routed to forging:dataset-definition rather than to write_dataset_data_tool
- [ ] A missing data_descriptions.feather was reported from validate_dataset_descriptions_tool's valid flag rather
      than from the reader's error envelope
- [ ] A vr_configuration.yaml issue was checked against the dataset's session type before being reported as a defect
- [ ] Questions about what an assembled column contains were handed off to the acquisition system's schema skill,
      which for Mesoscope-VR is mesoscope:mesoscope-vr-processing-schema
```
