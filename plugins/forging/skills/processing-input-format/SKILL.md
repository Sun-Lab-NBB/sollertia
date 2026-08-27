---
name: processing-input-format
description: >-
  Documents the on-disk inputs each sollertia-forgery batch pipeline requires and why a session resolves fewer jobs than
  its universe declares. Covers the per-pipeline input roots, the manifests and archives discovery indexes, the registry
  seams that resolve every per-system input, and the cross-library handoff contract. Use when a prepared batch holds
  fewer jobs than expected, when a pipeline refuses a session for missing input, or when checking whether a session is
  ready for a given pipeline.
user-invocable: false
---

# Processing input format

Documents what must exist on disk before each of the six batch pipelines can run, the discovery rule that turns those
inputs into a job universe, and the reason a unit resolves fewer jobs than its universe declares. Owns no MCP tools.

---

## Scope

**Covers:**
- The input root each of the six batch pipelines reads, and the artifact discovery indexes there
- The manifests that fix a pipeline's job universe independently of which archives sit on disk
- The registry seams through which every per-system identifier, locator, and parser resolves
- The configuration files a pipeline writes for itself rather than reading from acquired data
- Why a job sits in a pipeline's universe without being possible for a given unit
- The admission gate a session clears before it joins a forged dataset
- The cross-library handoff contract naming each upstream artifact, its producer, and its tracker

**Does not cover:**
- Batch preparation, execution, monitoring, and output cleaning. Owned by `/batch-processing`.
- Job sizing, the resource model, and the project plan. Owned by `/job-planning`.
- The artifacts each pipeline writes and how to read them back. Owned by `/processing-results`.
- Dataset definition, dataset growth, and dataset state. Owned by `/dataset-definition`.
- The forging pipeline's own stage semantics. Owned by `/dataset-forging`.
- Session discovery and the `session_paths` lists a batch consumes. Owned by `assets:session-discovery`.
- The preprocessing that materializes a session's acquired data. Owned by `experiment:data-management`.
- Every acquisition-system file name, column, module, and session type. Owned by the `mesoscope:mesoscope-vr-*` skill
  family, whose per-pipeline members this skill names beside each seam.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## Input roots and discovery

Discovery reads names, never data. Each pipeline exposes one function that reads the manifests, indexes the filenames,
decodes nothing, and writes nothing, so enumerating a unit's jobs costs the same however often it is asked.

| Pipeline          | Unit    | Input root discovery reads                            | Discovery function                               |
|-------------------|---------|-------------------------------------------------------|--------------------------------------------------|
| `checksum`        | Session | `raw_data/`                                           | `managing.checksum.discover_checksum_jobs`       |
| `runtime`         | Session | `raw_data/behavior_data/`                             | `runtime.discover_runtime_jobs`                  |
| `microcontroller` | Session | `raw_data/behavior_data/`                             | `microcontrollers.discover_microcontroller_jobs` |
| `video`           | Session | `raw_data/behavior_data/` and `raw_data/camera_data/` | `video.discover_video_jobs`                      |
| `two_photon`      | Session | the raw imaging directory the registry locates        | `two_photon.discover_two_photon_jobs`            |
| `forging`         | Dataset | the dataset directory                                 | `forging.discover_forging_jobs`                  |

Every directory name above is owned by sollertia-shared-assets and reached only through `SessionData.raw_data`, so no
pipeline in this library invents a name for the acquired hierarchy.

Each discovery function returns a `(unit, universe, possible)` triple. The universe is every job the unit's
configuration declares, whatever sits on disk. The possible subset is the part this unit's data can currently support. A
job in the universe but not in the possible subset is input this session does not carry, which is an answer rather than
a failure.

| Pipeline          | A universe job is not possible when                                                                    |
|-------------------|--------------------------------------------------------------------------------------------------------|
| `checksum`        | The `raw_data` tree holds no file the digest covers                                                    |
| `runtime`         | The donated source identifier keys no archive in the behavior-data directory                           |
| `microcontroller` | A registered controller's archive name resolves to zero files, or to several                           |
| `video`           | A camera's archive resolves to zero files or to several, or the pose-prediction locator returns `None` |
| `two_photon`      | Never. Discovery returns the possible subset equal to the universe                                     |
| `forging`         | Never. Definition admits a session only once it carries the outputs the forging stages consume         |

---

## Checksum pipeline input

Every path this pipeline reads and writes lives under `raw_data`, so a checksum job runs safely beside any other
pipeline's job for the same session. `managing.checksum.run_checksum_processing_pipeline` hashes the whole tree through
`ataraxis_data_structures.calculate_directory_checksum` and compares the result against the digest stored in
`ax_checksum.txt` (`RawDataFiles.CHECKSUM`).

`managing.checksum._CHECKSUM_EXCLUDED_FILES` holds exactly three names, the checksum file itself,
`ProcessingTrackers.CHECKSUM`, and that tracker's lock filename, which is derived from `ProcessingTracker.lock_path` so
the two cannot disagree. Excluding them keeps the tracker's own presence from altering the value it records.

**The tracker sits under the acquired data, not beside an output.** `shared_assets.pipelines._SESSION_TRACKER_LOCATIONS`
binds this pipeline to `session.raw_data.checksum_tracker_path`, and its notes give the reason. This pipeline verifies
the acquired data in place, while every other pipeline records beside the output it produces. This pipeline is also the
one batch pipeline whose dispatch entry declares no output directory.

The stored digest is the single prerequisite an earlier run establishes. A verification run that finds no stored value
raises `FileNotFoundError` and names the remedy, regenerating the session's checksum to establish the value against
which a later verification compares. Regeneration rides on the `regenerate_checksum` option, whose mechanics
`/batch-processing` owns.

A digest mismatch is recorded as a job failure carrying both values rather than raised, so the last recorded verdict is
read off the tracker rather than off an exception.

---

## Runtime pipeline input

The pipeline reads exactly one archive per session. `runtime.pipeline.discover_runtime_jobs` calls
`registries.resolve_runtime_binding(system=...)`, which returns a `(source_id, parser)` pair, then indexes
`session.raw_data.behavior_data_path` with `ataraxis_data_structures.discover_log_archives`. That helper inspects the
directory's own entries alone, keying each `{source_id}_log.npz` file by its name with `LOG_ARCHIVE_SUFFIX` removed.

The universe always holds a single `runtime_processing` job whose specifier is the donated source identifier. The job is
possible only when that identifier keys a discovered archive. When it does not, the pipeline refuses with
`FileNotFoundError`:

```text
Unable to process runtime data for session '{session_name}'. No runtime log archive was found for source
'{source_id}' in '{log_directory}'. The runtime DataLogger writes exactly one archive per session under its fixed
source id.
```

The decode stage is agnostic. `runtime.pipeline._decode_archive` builds an `ataraxis_data_structures.LogArchiveReader`
and returns a two-column table, `time_us` as `pl.UInt64` and `payload` as `pl.Binary`, in archive order. Everything the
payload bytes mean is the donated parser's business.

Both halves of the runtime binding are donated per acquisition system, the fixed source identifier that names the
archive and the payload layout the parser reads.

The Mesoscope-VR source identifier and runtime parser that fill this seam are documented by
`mesoscope:mesoscope-vr-trial-decomposition`.

---

## Microcontroller pipeline input

### What the acquired data must carry

`microcontrollers.pipeline` resolves both of its stages from one call to
`ataraxis_communication_interface.resolve_jobs(log_directory=session.raw_data.behavior_data_path)`. That call walks the
tree for exactly one `microcontroller_manifest.yaml` and then indexes one `{source_id}_log.npz` archive name per
registered controller.

When no manifest resolves, `microcontrollers.pipeline._resolve_controllers` raises `FileNotFoundError` naming the log
directory, and the message states the manifest's purpose, that it enumerates the controllers and modules to extract and
confirms the archives were produced by ataraxis-communication-interface. A tree holding several manifests spans
several recordings and raises `ValueError` instead.

A controller whose archive name resolves to zero files, or to several, is left unresolved. Its universe entries survive
so a later run can fill them, and it contributes no requested job to this run.

A module contributes a `module_parsing` job only when four conditions hold at once. It is declared by a manifest
controller, it appears in the system's event-code mapping, it is eligible for this session, and it carries a registered
parser. Kernel extraction is never configured, because this pipeline consumes no kernel table.

### The extraction configuration is an output, not a prerequisite

`microcontrollers.pipeline._materialize_extraction_config` writes `extraction_configuration.yaml`
(`ataraxis_communication_interface.EXTRACTION_CONFIGURATION_FILENAME`) into
`session.processed_data.microcontroller_data_path`. It builds the document from the per-controller extraction targets
the pipeline derives, narrowing `registries.resolve_microcontroller_event_codes` by
`registries.resolve_eligible_microcontroller_modules` for the loaded session.

The write is unconditional and lands before any dispatch, in both the local and the remote path, because a scheduler may
dispatch a single extraction job into a fresh process that has only the file to read. Two consequences follow directly.
Never ask an experimenter to supply one, and never read its presence as evidence that a run completed.

### Extraction is a tracked job, and the parse jobs depend on it

The universe carries `microcontroller_data_extraction` with the controller identifier as its specifier, and
`module_parsing` with `{controller_id}-{module_type}-{module_id}`.
`microcontrollers.pipeline.microcontroller_job_prerequisites` declares each parse job's prerequisite as the extraction
job of the controller its specifier names, so extraction is a first-class stage inside the tracker rather than a
preparation step outside it.

When no configured controller carries both a present archive and at least one eligible module, the pipeline raises
`ValueError` rather than running an empty batch.

The Mesoscope-VR event codes, module eligibility rules, and parser roster that fill this seam are documented by
`mesoscope:mesoscope-vr-module-parsing`.

---

## Video pipeline input

### The camera manifest defines the job universe

`video.pipeline._resolve_camera_jobs` delegates to `ataraxis_video_system.resolve_jobs`, which walks
`session.raw_data.behavior_data_path` for exactly one `camera_manifest.yaml`. The pipeline then declares one
`camera_timestamp_extraction` job and one `motion_energy` job per registered camera, plus a single
`camera_timestamp_rename` job and a single `pose_tracking` job, each of the latter two carrying an empty specifier.

The universe is therefore a property of the manifest alone. A camera the manifest registers holds its two jobs whether
or not its archive or its recording is on disk, and an archive no manifest registers contributes no job at all.

When no manifest resolves, `video.pipeline._resolve_camera_jobs` raises `FileNotFoundError` naming
`camera_manifest.yaml` and the behavior-data directory. A tree holding several manifests, or a manifest registering no
camera, raises `ValueError`.

The possible subset narrows that universe one job kind at a time.

- A timestamp job is possible only when its camera's `{source_id}_log.npz` resolves to exactly one file.
- The rename job is possible when at least one timestamp job is, because it publishes what those jobs wrote.
- A motion-energy job is always possible, because it reads only that camera's recording.
- The tracking job is possible only when `registries.resolve_pose_prediction_locator(system=...)(session=...)` returns a
  path.

### The manifest names are load-bearing

The colloquial name each camera carries in the manifest determines the canonical filenames the rename job publishes and
locates that camera's recording on disk, so the manifest is the only camera configuration the pipeline needs.
`video.pipeline._verify_canonical_names` refuses the rename job before any link is made when two cameras share one name,
or when one camera's canonical filename is another camera's parsed feather.

`video.motion_energy.resolve_camera_video` reconstructs `{session_name}_{camera_name}.mp4` exactly under
`session.raw_data.camera_data_path` and never suffix-matches, so a recording that does not carry that precise name is
invisible to the job. An absent recording is benign, the job completes with no output and announces the skip.

### The pose predictions are produced outside this library

The tracking pass reads an externally produced pose-prediction file that no stage of this library writes.
`registries.resolve_pose_prediction_locator` resolves it, because the naming of that file belongs to the system that
produces it, and job discovery consults the locator to decide whether the session supports a tracking job at all. The
donated tracking function locates its own predictions again and returns without writing when it finds none, so the job
is safe on every session.

The Mesoscope-VR pose-prediction locator and tracking function that fill this seam are documented by
`mesoscope:mesoscope-vr-video-tracking`.

---

## Two-photon pipeline input

`two_photon.pipeline._resolve_data_path` calls `registries.resolve_two_photon_data_locator(system=...)(session=...)` to
find the session's raw imaging directory, which is cindra's input. The directory's name and its place in the acquired
hierarchy are donated per system.

Two conditions gate the run, and each raises `FileNotFoundError` naming the resolved directory. The directory must
exist, otherwise the session has no calcium-imaging data to process. The tree beneath it must carry a
`cindra_parameters.json` file (`cindra.PARAMETERS_FILENAME`), found with `discover_marker_files`, because every system
that produces two-photon data writes it at acquisition time so the cindra pipeline can recover the recording's
acquisition metadata.

The session's cindra configuration is materialized by this pipeline rather than supplied. `_resolve_configuration` takes
the configuration the donated resolver returns, overrides exactly three fields on it, the data path, the output path,
and the progress flag, and writes `configuration.yaml` into `session.processed_data.cindra_data_path` when persistence
is requested. A local run persists it. A remotely dispatched job requires it to be there already, since such a job reads
the configuration its preparation step wrote, and refuses with `FileNotFoundError` when the preparation step never ran.

Discovery returns the possible subset equal to the universe. Possibility here states what the recording can run rather
than what it has already produced, so a freshly acquired session reports its whole stage universe and the tracker
decides each stage's turn.

The Mesoscope-VR imaging-directory locator and cindra configuration resolvers that fill this seam are documented by
`mesoscope:mesoscope-vr-imaging-configuration`.

---

## Forging pipeline input

### The dataset hierarchy comes first

The forging pipeline's unit is a dataset directory, a sibling of the animal directories under the project root. That
hierarchy, its `dataset.yaml` marker and its `data_descriptions.feather` companion, must exist before the pipeline runs,
and `/dataset-definition` owns its creation.

`forging.discover_forging_jobs` reads nothing outside the dataset hierarchy, so a dataset whose source sessions have
moved off this machine still resolves its jobs. Assembly then fails on the absent source, which is the intended shape,
since a dataset is deliberately forged in passes.

### Admission is the gate, and it runs at definition time

`forging.admission.verify_session_admissibility` is called from `forging/dataset.py`, not from the pipeline. It reads
`registries.resolve_forging_admission_pipelines(system=...)` and, for the session's recorded type, requires every listed
pipeline to report that each of its jobs succeeded. A session type absent from a system's mapping joins no dataset at
all.

Admission checks trackers, not files, because every pipeline resolves its own job universe from the acquisition
manifests, so a completed tracker already accounts for every source the session recorded. A tracker that is absent, or
present with no jobs, reads as not started. The refusal names every outstanding pipeline and its state:

```text
Unable to admit session '{session_name}' into a forged dataset. A session joins a dataset only once every pipeline
required by its acquisition system has completed, but the following are outstanding: {outstanding}. Process the
session through them, then define the dataset again.
```

A session already inside a dataset therefore carries the processed outputs the assembly worker reads.
`forging.pipeline._forge_session` loads that source session from `project_root/{animal}/{session}`, so assembly reads
the original session in the project hierarchy rather than a copy under the dataset.

### The required raw assets and the described columns

`SessionData.required_raw_assets` is the single source of truth for which assets a session must carry, and assembly
enforces it over the three files it re-exports. The session descriptor is always required, the experiment configuration
when the session names an experiment, and the VR configuration when the session type uses a VR task. A required but
absent asset raises `FileNotFoundError` before any expensive work begins, naming the filename and the path where it was
expected.

Every column the donated assembler writes into a session's `data.feather` must have a matching description in the
dataset's `data_descriptions.feather`. The pipeline reads the written file's schema alone and raises `ValueError`
listing every undescribed column.

### Cross-recording input is materialized, not supplied

For an animal whose system tracks cells across recordings, `forging.pipeline.define_forging_dataset` materializes a
`multi_recording_configuration.yaml` into that animal's directory inside the dataset, binding its recording directories
to the cindra output directory of each of that animal's sessions. That file is a pipeline output too, so an animal
without one is either untracked or has its source sessions elsewhere.

Whether any animal needs one follows from `registries.resolve_multi_recording_session_types`, read off the dataset's own
recorded session type. A dataset whose type is untracked skips the call outright and reads no source data.

The Mesoscope-VR admission policy and assembly worker that fill these seams are documented by
`mesoscope:mesoscope-vr-dataset-assembly`, and its tracked session types by
`mesoscope:mesoscope-vr-imaging-configuration`.

---

## Cross-library handoff contract

Every upstream artifact below is produced outside this library. The tracker column names the tracker whose jobs go
unfilled when the artifact is missing.

| Upstream artifact                                        | Producing library                       | Owning skill                            | Tracker involved                          |
|----------------------------------------------------------|-----------------------------------------|-----------------------------------------|-------------------------------------------|
| The session `raw_data` tree and its stored digest        | sollertia-experiment                    | `experiment:data-management`            | `checksum_processing_tracker.yaml`        |
| The acquisition runtime log archive                      | the ataraxis-data-structures DataLogger | `experiment:acquisition-system-runtime` | `runtime_processing_tracker.yaml`         |
| The microcontroller manifest and its log archives        | ataraxis-communication-interface        | `communication:log-input-format`        | `microcontroller_processing_tracker.yaml` |
| The camera manifest and its log archives                 | ataraxis-video-system                   | `video:log-input-format`                | `video_processing_tracker.yaml`           |
| The camera recordings the energy jobs read               | ataraxis-video-system                   | `video:post-recording`                  | `video_processing_tracker.yaml`           |
| The externally produced pose predictions                 | the system's own inference tool         | `mesoscope:mesoscope-vr-video-tracking` | `video_processing_tracker.yaml`           |
| The cindra acquisition parameters file                   | cindra                                  | `cindra:acquisition-data-preparation`   | `single_recording_tracker.yaml`           |
| The per-session cindra outputs the multi-day stages read | cindra                                  | `cindra:single-recording-results`       | `forging_tracker.yaml`                    |
| The `session_paths` lists every batch consumes           | sollertia-shared-assets                 | `assets:session-discovery`              | none                                      |

---

## Related skills

The `video:`, `communication:`, and `cindra:` entries below resolve through the ataraxis and cindra marketplaces. Every
other entry resolves inside the sollertia marketplace.

| Skill                                          | Relationship                                                                |
|------------------------------------------------|-----------------------------------------------------------------------------|
| `/batch-processing`                            | Downstream: prepares and runs the jobs whose inputs this skill locates      |
| `/job-planning`                                | Downstream: sizes each job from the inputs this skill locates               |
| `/processing-results`                          | Downstream: the artifacts each pipeline writes from these inputs            |
| `/dataset-definition`                          | Upstream: creates the dataset hierarchy the forging pipeline reads          |
| `/dataset-forging`                             | Peer: the forging pipeline's own stage semantics                            |
| `/data-processing-design`                      | Context: the registry seams through which every per-system input resolves   |
| `/pipeline`                                    | Context: the end-to-end phase map this skill sits inside                    |
| `/forging-mcp-environment-setup`               | Prerequisite: server connectivity and the response contract                 |
| `assets:session-discovery`                     | Upstream: the exclusive producer of the session path lists a batch consumes |
| `experiment:data-management`                   | Upstream: the preprocessing that materializes a session's acquired data     |
| `mesoscope:mesoscope-vr-processing-schema`     | Reference: the Mesoscope-VR file name and column rosters the seams produce  |
| `mesoscope:mesoscope-vr-module-parsing`        | Reference: the Mesoscope-VR event codes, eligibility rules, and parsers     |
| `mesoscope:mesoscope-vr-trial-decomposition`   | Reference: the Mesoscope-VR runtime source identifier and its parser        |
| `mesoscope:mesoscope-vr-video-tracking`        | Reference: the Mesoscope-VR pose-prediction locator and tracking pass       |
| `mesoscope:mesoscope-vr-imaging-configuration` | Reference: the Mesoscope-VR imaging locator and cindra resolvers            |
| `mesoscope:mesoscope-vr-dataset-assembly`      | Reference: the Mesoscope-VR admission policy and assembly worker            |
| `communication:log-input-format`               | Upstream: the microcontroller manifest and log archive format               |
| `video:log-input-format`                       | Upstream: the camera manifest and log archive format                        |
| `video:post-recording`                         | Upstream: verifying the camera recordings before the energy jobs read them  |
| `cindra:acquisition-data-preparation`          | Upstream: the acquisition parameters the two-photon pipeline requires       |
| `cindra:single-recording-results`              | Upstream: the per-session outputs the cross-recording stages read           |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Input readiness (checked per pipeline before a batch is prepared):
- [ ] No acquisition-system-specific file name, column, module, or session type appears in this skill
- [ ] checksum: the raw_data tree holds at least one file the digest covers
- [ ] checksum: a stored ax_checksum.txt exists, unless the run is establishing it
- [ ] runtime: the behavior-data directory holds one archive under the source id resolve_runtime_binding returns
- [ ] microcontroller: the behavior-data tree holds exactly one microcontroller_manifest.yaml
- [ ] microcontroller: every controller to be extracted resolves exactly one log archive
- [ ] microcontroller: extraction_configuration.yaml was NOT requested from the experimenter
- [ ] video: the behavior-data tree holds exactly one camera_manifest.yaml
- [ ] video: every registered camera carries a name distinct from every other camera's name
- [ ] video: every camera whose timestamps are needed resolves exactly one log archive
- [ ] video: every camera whose motion energy is needed carries its recording under camera_data
- [ ] two_photon: the located raw imaging directory exists and its tree carries cindra_parameters.json
- [ ] forging: every session in the dataset cleared verify_session_admissibility for its recorded type
- [ ] forging: every animal tracked across recordings has its source sessions on this machine
- [ ] A job in a universe but absent from the possible subset was read as unsupported input, not as a failure
```
