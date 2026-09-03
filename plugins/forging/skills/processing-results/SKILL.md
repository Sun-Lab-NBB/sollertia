---
name: processing-results
description: >-
  Documents what each sollertia-forgery pipeline writes to disk, where its tracker lives, and how an agent verifies an
  output when the library ships no verification tool. Covers the processed-data tree, the project and dataset artifacts,
  per-pipeline output ownership, and the reading of a legitimately empty or absent output. Use when evaluating
  processing results, when the user asks whether a session or a dataset finished, or when a stage reports success while
  its output directory looks wrong.
user-invocable: false
---

# Processing results

Documents every artifact the pipelines write, the tracker each one records against, and the procedure that separates a
real success from a vacuous one. This skill owns no MCP tools.

**The server ships no output-verification tool and no feather-query tool.** Nothing on it opens a pipeline's output
feather, counts its rows, or checks its schema, and the read tools open only the stored record artifacts.
Verification runs through the breakdowns of `read_project_jobs_tool`,
owned by `/project-state`, and of `get_processing_status_tool`, owned by `/batch-processing`. The forging jobs the
project job artifact never carries are covered by `read_dataset_state_tool`, owned by `/dataset-definition`. Read the
record with those tools first, then read the bytes by hand.

---

## Scope

**Covers:**
- The `processed_data` tree a fully processed session carries, with each agnostic filename pattern and its writer
- The project-level tables, the forged-dataset hierarchy, and the directory each pipeline owns
- Per-pipeline tracker location, output ownership, and what a legitimately empty or absent output means
- How to tell a real success from a job that succeeded having written nothing
- Interpretation rules for the positional camera tables, atomic publication, and the status vocabularies

**Does not cover:**
- Signatures, filters, and paging of the read tools. Owned by `/project-state`, `/batch-processing`, and
  `/dataset-definition`.
- Preparing a batch, executing it, resetting jobs, and cleaning output. Owned by `/batch-processing`.
- What must exist on disk before a pipeline can run. Owned by `/processing-input-format`.
- The per-unit plan cache and the resource model behind `cores` and `memory_mb`. Owned by `/job-planning`.
- Every acquisition-system file name, column schema, and session type. Owned by the `mesoscope:mesoscope-vr-*` skill
  family, one member per pipeline, listed in the related-skills table.
- The upstream log archive formats. Owned by `video:log-input-format` and `communication:log-input-format`.
- The extracted-message schema of those archives. Owned by `video:log-processing-results` and
  `communication:log-processing-results`.
- The imaging library's array and image formats under the session's imaging output directory. Owned by
  `cindra:single-recording-results`.
- The preprocessing that materializes `raw_data`. Owned by `experiment:data-management`.

**Handoff rules:** a question about which sessions are ready goes to `/project-state`, and a question about why a job
failed while a batch is still installed goes to `/batch-processing`. Any question naming a concrete behavior column,
parsed table, or camera name goes to the acquisition system's own plugin.

---

## Agent requirements

You MUST read the tracker record before the filesystem. The library never re-derives success from the presence of
files, and `forging.admission.verify_session_admissibility` states the rule for the whole codebase, checking which
pipelines completed rather than counting sources.

You MUST NOT report a stage as finished because its output directory holds files, and you MUST NOT report it as
failed because a file you expected is absent. Both readings are settled by the tables in this skill.

You MUST NOT name an acquisition system's file, column, or session type when reporting a result from this plugin.
Report the agnostic artifact and defer the schema.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## Where verification actually happens

Four stored artifacts answer every completeness question, and each has exactly one read tool.

| Question                                        | Artifact                           | Tool                         | Owner                 |
|-------------------------------------------------|------------------------------------|------------------------------|-----------------------|
| Are the stored project tables current           | `manifest_processing_tracker.yaml` | `get_manifest_status_tool`   | `/project-state`      |
| Which sessions have finished which pipeline     | `{project}_manifest.feather`       | `read_project_manifest_tool` | `/project-state`      |
| How did each individual job fare                | `{project}_jobs.feather`           | `read_project_jobs_tool`     | `/project-state`      |
| How did this dataset's forging jobs fare        | `dataset_state.feather`            | `read_dataset_state_tool`    | `/dataset-definition` |
| What is the batch this process is running doing | live trackers                      | `get_processing_status_tool` | `/batch-processing`   |

The manifest's pipeline columns are gross done indicators, so they answer which sessions are ready and nothing finer.
The job artifact carries one row per tracked job of the five per-session pipelines, and it holds no forging row at all,
so a `pipelines=["forging"]` filter is rejected rather than returning an empty listing. Timing comes only from
`detailed=True`, which appends `started_at` and `completed_at` to a job row on both the job read and the status read.

---

## Recommended query order

The order that establishes whether a batch succeeded, opening with the regeneration that stops the snapshot predating
the batch, is the `## Recommended query order` section of `/project-state`. The forging jobs it does not reach are read
through `read_dataset_state_tool`, owned by `/dataset-definition`, and the batch this server process is running through
`get_processing_status_tool`, owned by `/batch-processing`.

**Read the bytes by hand** only after that order reports `SUCCEEDED` and its counts look right. Every table this
library writes is uncompressed Arrow IPC, so `pl.read_ipc_schema` answers a column question from the footer alone and
`pl.read_ipc` memory-maps the rest.

---

## Output data reference

### The session processed-data tree

Entries below are the agnostic ones, stable under every acquisition system. Elided entries are named by the system's
donated parsers and workers, so their filenames and column schemas belong to that system.

```text
<project>/<animal>/<session>/
├── raw_data/                                           <- acquired data. This library writes exactly two files here
│   ├── ax_checksum.txt                                 written only by a regeneration run of the checksum pipeline
│   └── checksum_processing_tracker.yaml                written by managing.checksum
└── processed_data/                                     <- everything else this library writes for a session
    ├── job_plan.yaml                                   the session plan cache, orchestration.planning._save_plan
    ├── runtime_data/
    │   ├── runtime_processing_tracker.yaml             written by runtime.pipeline
    │   └── <the acquisition system's runtime tables>
    ├── microcontroller_data/
    │   ├── microcontroller_processing_tracker.yaml     written by microcontrollers.pipeline
    │   ├── extraction_configuration.yaml               _materialize_extraction_config, rewritten every invocation
    │   ├── controller_{cid}_module_{type}_{mid}.feather   raw extracted messages, one file per emitting module
    │   └── <the acquisition system's parsed tables>
    ├── video_data/
    │   ├── video_processing_tracker.yaml               written by video.pipeline
    │   ├── camera_{source_id}_timestamps.feather       the parse stage's output, one file per camera archive
    │   ├── {camera}_timestamps.feather                 the rename stage's hardlink, named from the camera manifest
    │   ├── {camera}_energy.feather                     video.motion_energy.compute_camera_motion_energy
    │   └── <the acquisition system's pose-tracking tables>
    └── cindra/                                         <- the imaging library owns everything below this point
        ├── single_recording_tracker.yaml               registered by two_photon.pipeline, recorded on by cindra
        ├── configuration.yaml                          the one file this library writes here, two_photon.pipeline
        ├── acquisition_parameters.yaml
        ├── combined_metadata.npz                       the combination stage's completion marker
        ├── plane_{n}/                                  per virtual plane, with registration and detection outputs
        └── multi_recording/{animal}_{dataset}/         lower-cased; the forging pipeline writes then reads these
```

Each tracker's `.lock` companion sits beside it, derived from the tracker path by `ProcessingTracker.lock_path`, so it
appears in every directory listing without being an output.

### The project tables

`managing.manifest.generate_project_manifest` writes the first three files below, publishing the job table before the
manifest deliberately, because the two renames are not simultaneous and readers take no lock. The plan table is written
separately by the planning pipeline, which `/job-planning` owns.

| Path                               | Written by                                     | Row semantics                                     |
|------------------------------------|------------------------------------------------|---------------------------------------------------|
| `{project}_jobs.feather`           | `managing.jobs.write_project_jobs`             | one row per tracked job of a per-session pipeline |
| `{project}_manifest.feather`       | `managing.manifest.generate_project_manifest`  | one row per session with a non-empty `raw_data`   |
| `manifest_processing_tracker.yaml` | `managing.manifest`                            | the single `manifest_generation` job              |
| `{project}_plan.feather`           | `orchestration.planning.generate_project_plan` | one row per planned job, session and dataset      |

`animal` is a `String` column in all four artifacts, so the manifest, jobs, plan, and dataset-state tables join on
animal and session without a cast. The plan joins the job table on `job_id`.

### The forged-dataset hierarchy

The forging pipeline owns this whole tree, and `forging.dataset` materializes it before any assembly job runs.

```text
<project>/<dataset>/                                    <- a sibling of the animal directories, not a child
├── dataset.yaml                                        the marker, project_hierarchy.DATASET_MARKER_FILENAME
├── data_descriptions.feather                           two String columns, `column` and `description`
├── forging_tracker.yaml                                ProcessingTrackers.FORGING
├── dataset_state.feather                               forging.state.generate_dataset_state
├── job_plan.yaml                                       the dataset plan cache
└── <animal>/
    ├── surgery_metadata.yaml                           copied from the animal's most recent source session
    ├── multi_recording_configuration.yaml              only for an animal the system tracks across recordings
    └── <session>/
        ├── data.feather                                the assembled session table, DatasetFiles.DATA
        ├── session_descriptor.yaml                     always re-exported
        ├── vr_configuration.yaml                       re-exported only when the source session carries one
        └── experiment_configuration.yaml               re-exported only when the source session carries one
```

The existence and the path of `data.feather` are agnostic. Its column set is not, since the acquisition system donates
the assembler that fills it and the descriptions that document it.

### What the tracker document holds

Every tracker is an `ataraxis_data_structures.ProcessingTracker` serialized as YAML with one top-level `jobs` key,
mapping a hexadecimal `job_id` to a record carrying `job_name`, `specifier`, `status`, `executor_id`, `error_message`,
`started_at`, and `completed_at`. The `job_id` is `ProcessingTracker.generate_job_id`, an xxHash64 of the job name and
the specifier, so a job's identity is reproducible from its name and specifier alone.

---

## Per-pipeline output ownership

| Pipeline          | Tracker on disk                                                               | Output directory it owns                                    |
|-------------------|-------------------------------------------------------------------------------|-------------------------------------------------------------|
| `manifest`        | `<project>/manifest_processing_tracker.yaml`                                  | none, it writes two project tables                          |
| `checksum`        | `<session>/raw_data/checksum_processing_tracker.yaml`                         | none                                                        |
| `runtime`         | `processed_data/runtime_data/runtime_processing_tracker.yaml`                 | `processed_data/runtime_data`                               |
| `microcontroller` | `processed_data/microcontroller_data/microcontroller_processing_tracker.yaml` | `processed_data/microcontroller_data`                       |
| `video`           | `processed_data/video_data/video_processing_tracker.yaml`                     | `processed_data/video_data`                                 |
| `two_photon`      | `processed_data/cindra/single_recording_tracker.yaml`                         | `processed_data/cindra`                                     |
| `forging`         | `<dataset>/forging_tracker.yaml`                                              | the dataset tree, plus one directory in each source session |

| Pipeline          | Job names, in stage order                                                                  | Specifier of each, in the same order             |
|-------------------|--------------------------------------------------------------------------------------------|--------------------------------------------------|
| `manifest`        | `manifest_generation`                                                                      | the project directory stem                       |
| `checksum`        | `checksum_resolution`                                                                      | the session name                                 |
| `runtime`         | `runtime_processing`                                                                       | the runtime log source id the system declares    |
| `microcontroller` | `microcontroller_data_extraction`, `module_parsing`                                        | controller id, then `{cid}-{type}-{mid}`         |
| `video`           | `camera_timestamp_extraction`, `camera_timestamp_rename`, `pose_tracking`, `motion_energy` | camera source id, empty, empty, camera source id |
| `two_photon`      | `binarization`, `registration`, `processing`, `combination`                                | empty, `plane_{n}`, `plane_{n}`, empty           |
| `forging`         | `multiday_discovery`, `multiday_extraction`, `session_data_assembly`                       | animal, session, session                         |

### The checksum pipeline

It owns no output directory, because it verifies the acquired data in place. Cleaning it removes the tracker and the
tracker's lock and leaves `ax_checksum.txt` where it is, so the unit keeps the baseline a later verification compares
against, and the unit contributes exactly one removed path rather than two.

A verification run against a session carrying no stored checksum raises `FileNotFoundError` rather than recording a
mismatch. A session whose `raw_data` holds only the excluded bookkeeping files is refused before the tracker is touched,
so the last recorded verdict survives.

**Real against vacuous.** A `SUCCEEDED` job under `regenerate_checksum=False` means the recomputed digest matched the
stored one. A `SUCCEEDED` job under `regenerate_checksum=True` proves only that the baseline was rewritten, since the
run stores the value it just computed and then compares against it. Read the mode before reading `integrity == 1` as
evidence of integrity. A mismatch does not raise. It records a job failure carrying the message
`Raw data integrity compromised: recomputed checksum '<computed>' does not match stored checksum '<stored>'.`, and this
is the one pipeline whose failure means the data is corrupt rather than the processing.

### The runtime pipeline

One job writes every table into `processed_data/runtime_data`. Which tables a successful run produces, which of them
are conditional on a recorded event, and what each column carries are all the acquisition system's contract, filled
through the runtime parser registry.

**Real against vacuous.** The tracker rolling up to `completed` is the whole record for this pipeline, since a single
job either produced the system's tables or failed. A directory holding only the tracker after a `SUCCEEDED` job means
the donated parser found no payload it routes, which is a question for the acquisition system's plugin.

### The microcontroller pipeline

Two stages record on one tracker. The extraction stage writes `controller_{cid}_module_{type}_{mid}.feather`, one file
per module that produced at least one message, each carrying the five fixed columns `timestamp_us`, `command`, `event`,
`dtype`, and `data`. The parse stage reads those raw tables and writes the acquisition system's parsed tables beside
them.

No `controller_{cid}_kernel.feather` is ever produced here, because `microcontrollers.pipeline._resolve_controllers`
builds every `ControllerExtractionConfig` with `kernel=None`.

`extraction_configuration.yaml` is rewritten on every invocation, before any job is dispatched, in local and remote
mode alike. Its presence is evidence that the pipeline was invoked, never that a run completed.

**Real against vacuous.** A module that emitted no message leaves no raw feather, and its parse job completes with no
output, echoing `No extracted data was found for module '<specifier>'. Completing its parse job with no output.` A
module the acquisition system's eligibility gate rejects writes no parsed table either. Both are correct outputs. The
pipeline never globs this directory. It derives each path from the module identity, so a directory listing is not the
job set. Count `module_parsing` rows in the job artifact against the raw tables rather than counting files.

### The video pipeline

The parse stage writes `camera_{source_id}_timestamps.feather`, one file per camera archive, with the single column
`frame_time_us` and one row per acquired frame in acquisition order. The rename stage then publishes each parsed
feather under `{camera}_timestamps.feather`, the name the acquisition-time camera manifest gives that camera. The
canonical name is a hardlink sharing the parsed feather's inode, so both names cost one copy of the bytes, both are
present after a successful rename, and the parsed feather is never deleted. A stale canonical path is unlinked first,
so the stage is re-runnable, and a host that cannot hardlink falls back to a copy.

The rename stage refuses the whole job before linking anything when two cameras carry the same manifest name, or when
one camera's canonical name is another camera's parsed feather. A camera whose manifest name is literally
`camera_{source_id}` already sits at its canonical path, so it is counted as published and left untouched.

`motion_energy` and `pose_tracking` run independently of the parse and rename pair. The energy stage writes
`{camera}_energy.feather` with the two Float32 columns `motion_energy` and `frame_luminance`, one row per decoded
frame. The tracking stage's output is named by the acquisition system's donated tracking function.

**Real against vacuous.** The rename job reports `SUCCEEDED` having published zero names when no parsed feather exists
yet, echoing `Renamed {published} parsed camera timestamp feather(s) to their canonical names.` with a count of zero, so
read the count rather than the status. A camera whose recording is absent from the raw camera data directory completes
its energy job with no output, echoing that no recording was found for that camera and that its motion-energy
measurement is skipped. A rig that ran one of two cameras therefore leaves a clear tracker on the camera it did not run.
A camera's timestamp job that appears in the discovered universe but not in the possible set has no log archive, or has
a name that resolves to several archives. Its energy job stays possible because that stage reads only the recording.
### The two-photon pipeline

The imaging library writes the arrays and images under `processed_data/cindra` and records the four stages on
`single_recording_tracker.yaml`. This library writes two files there itself. It materializes `configuration.yaml` when
the run persists it, overriding three fields and leaving everything else as the acquisition system's resolver returned
it, and it registers the four stages on `single_recording_tracker.yaml` through the tracker's own alignment before any
stage runs, which is what creates that file. The tracker's presence is therefore evidence that the pipeline was
invoked, not that a stage completed.

Two files are completion markers worth more than a directory listing, because each is written atomically after the
arrays it describes. `combined_metadata.npz` marks the combination stage, and `tracking_template_masks.npz` under the
cross-recording directory marks multi-recording discovery. A `.binarizing` or `.registering` marker beside a plane
binary means the opposite, that the stage died mid-write and the binary must be rebuilt. That cross-recording
directory is the forging pipeline's output, and both cleans remove it: `forging` removes it as declared external
output, and `two_photon` removes it along with this pipeline's own arrays.

**Real against vacuous.** On a freshly acquired session the possible job set equals the universe by design, since
possibility here states what the session can run rather than what it has already produced. A full universe is
therefore not evidence of progress, and the tracker decides each stage's turn.

### The forging pipeline

It owns the whole dataset hierarchy, and it also owns a directory inside every source session that hierarchy names,
so cleaning it removes three things rather than two: every assembled feather in the dataset, the tracker beside them,
and each source session's cross-recording directory for this dataset. This is the most destructive operation the tool
set offers, it is the only clean that reaches outside the unit it names, and it is irreversible. Prefer resetting the
jobs, through `/batch-processing`, whenever the failure cause was external.

Two of its three job kinds write outside that hierarchy. `multiday_discovery` and `multiday_extraction` run the
imaging library against each source session's `processed_data/cindra` directory, so their output lands under every
source session, in the lower-cased `processed_data/cindra/multi_recording/{animal}_{dataset}`. The forging dispatch
entry declares those directories as its external output, and a clean resolves every one of them before it removes
anything, then removes each alongside the dataset hierarchy and the tracker. Cleaning `forging` therefore takes the
cross-recording output with it, and a later forging run rediscovers every job the dataset can run rather than skipping
work whose output is gone. The directory is named for this dataset alone, so a session belonging to several datasets
keeps the sibling directory each of the others owns, and a forging clean leaves that session's single-recording arrays
in place. A source session that no longer resolves is logged as a warning and passed over, keeping its directory, since
a clean cannot remove what it cannot locate.

The hazard runs the other way. `two_photon` owns the whole of `processed_data/cindra`, so cleaning it for a source
session takes that session's single-recording arrays, its `combined_metadata.npz`, and its
`single_recording_tracker.yaml` along with the cross-recording directory the dataset owns inside it, while the
dataset's own forging tracker is untouched and still records those jobs as `SUCCEEDED`. A later forging run therefore
skips them, and an assembly job reading that output fails against the missing directory. No clean regenerates it. Reset
the animal's forging jobs on the dataset's forging tracker instead, through `/batch-processing`, naming the `job_id`
that `dataset_state.feather` records for the animal's `multiday_discovery` row and for each of its sessions'
`multiday_extraction` and `forging` rows. Rebuilding the animal resets those same three job kinds on its behalf. That
session's own two-photon pipeline has to run again first, since the cross-recording stages read each source session's
`combined_metadata.npz` and fail against its absence rather than rebuilding it.

Forging jobs never reach `{project}_jobs.feather`, which carries the five per-session pipelines only. They live in the
dataset's own `dataset_state.feather`, one row per tracked forging job, carrying `scope` so a reader resolves a row's
subject instead of assuming the specifier names a session. `multiday_discovery` is animal-scoped and exists only where
the acquisition system resolves a cross-recording configuration, while both other job names are session-scoped.

After the worker writes a session's `data.feather`, `forging.pipeline._forge_session` re-reads the schema through
`pl.read_ipc_schema` and raises when a written column has no matching row in the dataset's `data_descriptions.feather`.
The contract is one-directional, so a described column that no session emits is permitted.

**Real against vacuous.** A `dataset_state.feather` with zero rows is a valid empty artifact, meaning the dataset has
no forging tracker or a tracker holding no jobs, not an error. A row with a null `animal` means a rebuild dropped that
session while the tracker still holds its outstanding rows, which is deliberate so a state read stays available across
that window. An animal directory carrying no cross-recording configuration means either that the dataset's session
type is not tracked across recordings, or that the animal's source sessions have moved off this machine, since a
dataset is forged in passes. An animal directory carrying no `surgery_metadata.yaml` means its latest source session
carried none, which is optional by design and logged as a warning.

---

## Interpretation guidance

### Every camera table is positional

The timestamp feather, the energy feather, and any per-frame table a system donates carry one row per acquired frame in
acquisition order, with no index column and no timestamp column of their own. Position is the frame index, and
alignment to the acquisition clock is left to dataset assembly. The consequence is a hard equality. A camera's value
feather must have exactly the height of that camera's timestamp feather, and the acquisition system's assembler rejects
a mismatch rather than padding. Compare the two heights before reporting either table as good.

### An absent output is a reading, not a failure

| Observation                                           | What it legitimately means                                                                                                                                 |
|-------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| A tracker file is absent, or holds no jobs            | Never run. The manifest and admission readers both report `not_started`, because a pipeline registers every job it resolved before dispatching any of them |
| A session carries no row in the project manifest      | Its `raw_data` directory is empty, so the session was aborted or never acquired                                                                            |
| A unit contributes no row to `{project}_plan.feather` | That unit carries no `job_plan.yaml`. Read the job as unplanned, never as free                                                                             |
| A module has no raw extracted feather                 | It emitted no message, and its parse job completed with no output                                                                                          |
| A camera has no energy feather                        | Its recording is absent from the raw camera directory, and the energy job completed                                                                        |
| `dataset_state.feather` holds zero rows               | The dataset has no forging tracker, or its tracker holds no jobs                                                                                           |

### The status vocabularies never mix

A tracker job records one of `SCHEDULED`, `RUNNING`, `SUCCEEDED`, and `FAILED`. The job artifact and the dataset state
carry those names verbatim, so their `status` filters take the uppercase form. The local status read lowercases them.
A tracker's rolled-up label is a different vocabulary entirely, `failed`, `completed`, `processing`, `not_started`, and
`in_progress`, and only `completed` means every tracked job succeeded. Never filter one vocabulary with the other's
value.

### A blocked job is not a status

A job is reported blocked when the run could neither queue its upstream stage nor confirm that the stage already
succeeded. Blocked jobs were never dispatched, so their tracker record still reads `SCHEDULED` and the status response
carries a count rather than a listing. List them by filtering to the scheduled status, and read the upstream stage's
failure as the cause.

### Publication is atomic, so a torn file is not a diagnosis

The project manifest, the project job table, the project plan, and the dataset state are all published through
`ataraxis_data_structures.atomic_write`, a temporary file renamed over the destination, so a memory-mapping reader
never observes a partially written one. Treat a table that fails to decode as a foreign file or a damaged filesystem
rather than as partial output, and do not add a re-run step on the theory that a write was cut short.

### Rows are naturally sorted

Every project and dataset table is ordered with `shared_assets.utilities.natural_sort`, which ranks then sorts
integers, so animal `2` precedes animal `10`. A reader comparing two listings by position gets the same order on both.

---

## Related skills

The `video:`, `communication:`, and `cindra:` entries below resolve through the ataraxis and cindra marketplaces. Every
other entry resolves inside the sollertia marketplace.

| Skill                                           | Relationship                                                                      |
|-------------------------------------------------|-----------------------------------------------------------------------------------|
| `/project-state`                                | Owns the manifest, job, and manifest-status read tools this skill points at       |
| `/batch-processing`                             | Owns the status, reset, and clean tools, and the batch that produced the output   |
| `/dataset-definition`                           | Owns the dataset state read tool and the dataset marker                           |
| `/job-planning`                                 | Owns the per-unit plan cache and the project plan table                           |
| `/processing-input-format`                      | Reference: what must exist on disk before any of these outputs can be written     |
| `/pipeline`                                     | Context: routes an end-to-end run through the skills that produce these artifacts |
| `/forging-mcp-environment-setup`                | Prerequisite: server connectivity and the response contract                       |
| `/cli-reference`                                | Reference: the `slf` commands that write the same artifacts without the server    |
| `mesoscope:mesoscope-vr-processing-schema`      | Reference: the file name roster every elided entry above resolves to              |
| `mesoscope:mesoscope-vr-dataset-assembly`       | Reference: the column universe of an assembled session table                      |
| `mesoscope:mesoscope-vr-module-parsing`         | Reference: the elided parsed tables under `microcontroller_data`                  |
| `mesoscope:mesoscope-vr-trial-decomposition`    | Reference: the elided tables under `runtime_data`                                 |
| `mesoscope:mesoscope-vr-video-tracking`         | Reference: the elided tracking tables under `video_data`                          |
| `mesoscope:mesoscope-vr-imaging-configuration`  | Reference: the resolver behind the persisted cindra `configuration.yaml`          |
| `mesoscope:mesoscope-vr-fluorescence-alignment` | Reference: the fluorescence columns the forged dataset carries                    |
| `assets:session-discovery`                      | Upstream: the exclusive producer of the session path lists a batch consumes       |
| `experiment:data-management`                    | Upstream: the preprocessing that materializes `raw_data`                          |
| `video:log-input-format`                        | Reference: the upstream camera log archive format this pipeline consumes          |
| `communication:log-input-format`                | Reference: the upstream microcontroller log archive format it consumes            |
| `video:log-processing-results`                  | Reference: the upstream camera archive extraction this pipeline dispatches        |
| `communication:log-processing-results`          | Reference: the extracted-message schema of the raw controller tables              |
| `cindra:single-recording-results`               | Reference: the imaging library's own array and image outputs                      |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Result verification, tool-settled (run read_project_manifest_tool, read_project_jobs_tool):
- [ ] sollertia-forgery MCP server is connected
- [ ] The query order of /project-state was followed, so the stored project tables were regenerated after the batch's
      last job retired
- [ ] Every pipeline column of every session under review reads 1 in read_project_manifest_tool
- [ ] read_project_jobs_tool with status FAILED and detailed True returns no row, or every returned error_message is
      accounted for
- [ ] read_dataset_state_tool reports SUCCEEDED for every session_data_assembly job of each dataset under review

Result verification, reader-judged:
- [ ] The checksum verdict under review came from a verification run, not from a regeneration run
- [ ] Each absent output was matched against the absent-output table before being reported as a problem
- [ ] Each camera value feather's height was compared against that camera's timestamp feather height
- [ ] A rename job reporting success was read by its published count, not by its status alone
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
