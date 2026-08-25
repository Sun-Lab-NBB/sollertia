---
name: camera-timestamp-extraction
description: >-
  Documents the system-agnostic, manifest-driven camera timestamp extraction stage in the sollertia-forgery
  cross_system layer: discovering raw VideoSystem logs, parsing the numeric source ID from a log filename, resolving
  per-source output names from the camera manifest, re-extracting timestamps into a frame_time_us feather, and the
  self-contained per-source stage orchestrator and tracker. Use when explaining how camera source IDs become named
  timestamp feathers, when adding camera support without hardcoding names, or when auditing the video stage.
user-invocable: false
---

# Camera timestamp extraction

Documents the system-agnostic camera timestamp extraction stage that converts each session's raw VideoSystem log
archives into canonical per-camera timestamp feathers, with every camera output name resolved from the
acquisition-time camera manifest rather than a hardcoded registry. This stage lives in
`sollertia_forgery.cross_system.video` and works for any acquisition system that records cameras.

This skill owns the manifest-driven extraction contract: how `{source_id}_log.npz` archives are discovered, how a
source ID maps to a `{name}_timestamps.feather` output, where the timestamps are read from and written to, and how
the per-source jobs are tracked. The concrete camera roles a given acquisition system records (for example, which
source ID is the face camera) are deferred to that system's hardware-composition skill, because those names enter
the pipeline as manifest data registered at acquisition time, not as a constant in this codebase.

---

## Scope

**Covers:**
- Non-recursive discovery of raw `*_log.npz` VideoSystem log archives via `find_camera_logs`
- Parsing the numeric camera source ID from a log filename via `extract_camera_source_id`
- Resolving each registered source ID to its `{name}_timestamps.feather` output name via
  `resolve_camera_output_names`, which reads the camera manifest through the ataraxis-video-system `CameraManifest`
- The source and sink boundary: raw archives are read from the session's raw `behavior_data_path` and the named
  timestamp feathers are written into the session's processed `video_data_path`
- Re-extraction of logged timestamps into an uncompressed-IPC feather with a `frame_time_us` `uint64` column via
  `process_camera_log`
- The self-contained per-source stage orchestrator `run_video_processing_pipeline` and the video
  `ProcessingTracker` at the session's `video_tracker_path`
- Why camera output names enter as manifest data rather than as a hardcoded registry

**Does not cover:**
- The concrete Mesoscope-VR camera roles and source IDs (the GenICam cameras and their assigned source IDs come
  from the manifest at acquisition time; the hardware-composition inventory is owned by `mesoscope:mesoscope-vr`)
- Upstream axvs camera log production and the naming the extraction binding gives its own parsed feathers (see
  `ataraxis@video:log-processing` and `ataraxis@video:log-input-format`)
- The producer-to-directory mapping covering every processed feather the pipeline writes (owned by
  `mesoscope:mesoscope-vr-processing-schema`, the single home of that table)
- The ataraxis-video-system `CameraManifest`, the `CAMERA_MANIFEST_FILENAME` constant, and the
  `extract_logged_camera_timestamps` internals — these are upstream axvs symbols re-used here (see
  `ataraxis@video:log-processing` and `ataraxis@video:log-input-format`)
- The prepare-then-execute batch model and the shared `JobExecutionState` orchestration primitives (see
  `forging:data-processing-design`)
- The separate behavior batch pipeline (runtime and microcontroller jobs only; it does not include camera jobs —
  the camera stage is run separately by the `process video` CLI command) (see `forging:behavior-processing`)
- Microcontroller module feather parsing (see `forging:microcontroller-primitives`)

---

## Discovering raw VideoSystem logs and source IDs

The stage consumes the raw camera logs a VideoSystem leaves in the session's raw behavior data directory,
`session.raw_data.behavior_data_path` (the on-disk `behavior_data` directory), which collects the NPZ archives
every DataLogger-backed source writes during acquisition. Each raw archive is named `{source_id}_log.npz`,
matching the DataLogger source-ID naming convention shared across the Sollertia stack. The separate raw
`camera_data_path` directory holds the camera recordings themselves rather than their log archives.

`find_camera_logs(data_directory)` discovers these archives with a **non-recursive** glob over the directory using
the `*_log.npz` pattern, then returns the matches sorted. It does not descend into subdirectories. If the directory
does not exist, or contains no matching archives, it returns an empty list rather than raising.

`extract_camera_source_id(log_path)` parses the numeric source ID out of the filename. It splits the filename stem
(for example, `51_log`) on underscores and requires exactly two components, the second of which must be the literal
`log` and the first of which must be all digits. When the filename does not satisfy that shape, it raises a
`ValueError` reporting the expected `{source_id}_log.npz` convention. Otherwise it returns the leading numeric
component as an `int`.

---

## Resolving output names from the camera manifest

`resolve_camera_output_names(data_directory)` is the contract that makes this stage system-agnostic. Every VideoSystem
writes a camera manifest alongside its log archives in the same raw behavior data directory. The function reads that
manifest and projects each registered source into its canonical output filename.

The function performs a deferred import of `CAMERA_MANIFEST_FILENAME` and `CameraManifest` from
ataraxis-video-system, joins `CAMERA_MANIFEST_FILENAME` onto the data directory to locate the manifest, and raises
`FileNotFoundError` when that manifest file is absent. It then loads the manifest with `CameraManifest.from_yaml`
and returns a dictionary mapping each manifest source's `id` to `{source.name}_timestamps.feather`. The colloquial
source name recorded at acquisition time directly determines the output name, so the pipeline needs no
acquisition-system-specific configuration.

The deferred import is intentional: importing `sollertia_forgery.cross_system.video` does not require
ataraxis-video-system to be installed; the acquisition binding is only resolved when a video extraction job
actually runs. `CAMERA_MANIFEST_FILENAME`, `CameraManifest`, and the manifest schema are upstream axvs symbols —
this skill documents how they are consumed, not their definitions.

---

## Source and sink directories

This stage has an explicit source/sink boundary, and both halves resolve as `SessionData` fields:

| Role   | Session path                             | On-disk directory           | Contents                                                            |
|--------|------------------------------------------|-----------------------------|---------------------------------------------------------------------|
| Source | `session.raw_data.behavior_data_path`    | `raw_data/behavior_data`    | Raw `{source_id}_log.npz` archives and the camera manifest          |
| Sink   | `session.processed_data.video_data_path` | `processed_data/video_data` | The named `{name}_timestamps.feather` outputs and the video tracker |

The stage **re-extracts** the timestamps from the raw logs in `behavior_data_path` and writes the named feathers
into `video_data_path`, which is the single processed directory this stage writes into. There is no separate
camera-timestamps landing directory anywhere in the processed tree, and `ProcessedData` carries no
`camera_timestamps_path` field. Note that `behavior_data` is a raw-side directory only: it is the DataLogger
archive directory under `raw_data`, and no processed feather is ever written into a `processed_data/behavior_data`
path. For the full mapping of every processed feather to the directory that receives it, defer to
`mesoscope:mesoscope-vr-processing-schema`.

The video `ProcessingTracker` lives at `session.processed_data.video_tracker_path`, which resolves to the
`video_processing_tracker.yaml` file inside that same `video_data` directory. The tracker and the timestamp
feathers therefore share one directory; there is no `camera_processing_tracker.yaml` and no separate tracker
location to reason about.

---

## Re-extracting timestamps into a frame_time_us feather

`process_camera_log(log_path, output_directory, output_name, *, workers=-1)` performs the extraction for a single
archive. It deferred-imports `extract_logged_camera_timestamps` from ataraxis-video-system and calls it as
`extract_logged_camera_timestamps(log_path=log_path, n_workers=workers)` to recover the per-frame acquisition
timestamps. The `workers` budget is passed straight through to the binding; setting it to `-1` lets the binding use
all available CPU cores (minus reserved cores), and the binding parallelizes the extraction of an individual
archive internally.

The recovered timestamps are written as a single-column feather:

- The column name is `frame_time_us`, holding per-frame acquisition timestamps in microseconds since the UTC epoch.
  This is the canonical camera-timestamp column recognized across the downstream forging and analysis pipelines.
- The column dtype is `uint64`: the timestamps are cast with `numpy` to `uint64` before the dataframe is built.
- The file is written via the polars Arrow IPC writer with `compression="uncompressed"`, so the resulting feather
  supports memory-mapped reads by the downstream forging pipeline.

The function creates `output_directory` (with parents) if needed, then writes the feather under `output_name` — the
canonical filename resolved from the manifest (for example, `face_camera_timestamps.feather`). Extraction and
renaming happen in a single pass.

---

## The self-contained per-source stage orchestrator and tracker

`run_video_processing_pipeline(session_path, job_id=None, *, workers=-1, display_progress=False)` is a
**self-contained** stage orchestrator: it performs its own discovery, validation, per-source job registry, and
execution in one call. It does not use the shared prepare-then-execute `JobExecutionState` batch model owned by
`forging:data-processing-design`.

The orchestrator proceeds as follows:

1. Loads the session via `SessionData.load(session_path=session_path)` and resolves the raw behavior data
   directory from `session.raw_data.behavior_data_path`.
2. Calls `resolve_camera_output_names` once to obtain the source-ID-to-output-name mapping from the manifest.
3. Discovers logs with `find_camera_logs`, parses each source ID with `extract_camera_source_id`, and registers a
   job only when that source ID is present in the resolved manifest mapping (archives whose source ID is not
   registered are skipped).
4. Builds a job registry keyed by the `(VIDEO_JOB_NAME, str(source_id))` tuple, where `VIDEO_JOB_NAME` is the
   module constant `"camera_timestamp_extraction"`.
5. Raises `ValueError` when no registered camera log archives are discovered.
6. Ensures the processed video data output directory exists, constructs a `ProcessingTracker` bound to
   `session.processed_data.video_tracker_path`, and aligns the tracker's job registry with the discovered job
   tuples via the shared `prepare_tracker` helper.

Each `(job_name, specifier)` tuple is the unit of work, and the specifier is the string form of the source ID. The
per-job state transitions are managed by a single private helper that calls `ProcessingTracker.generate_job_id` to
derive the job's unique hexadecimal identifier, then wraps the extraction in `start_job`, `complete_job`, and
`fail_job` calls so that a raised exception both records the failure on the tracker and re-raises.

---

## Local versus remote execution and how it differs from the batch model

`run_video_processing_pipeline` selects between two execution modes based on whether `job_id` is supplied:

| Mode   | Trigger              | Behavior                                                                                       |
|--------|----------------------|------------------------------------------------------------------------------------------------|
| Local  | `job_id is None`     | All discovered jobs run sequentially in the parent process; the optional progress bar advances per job |
| Remote | `job_id` is provided | Only the single job whose generated ID matches `job_id` is executed                             |

In local mode, the parent process iterates the job registry and runs each extraction in order; no additional worker
pool is layered on top, because the ataraxis-video-system binding already parallelizes each archive internally via
the `workers` budget. When `display_progress` is true, a progress bar covering the job count advances by one per
completed job.

In remote mode, the orchestrator builds a lookup from each generated job ID to its `(job_name, specifier)` tuple
using `ProcessingTracker.generate_job_id`, raises `ValueError` (reporting the valid job IDs) when the supplied
`job_id` matches none of them, and otherwise executes exactly that one job.

This is deliberately distinct from the shared prepare-then-execute batch model. The batch model (owned by
`forging:data-processing-design`) splits preparation from a separately-driven execution that dispatches jobs onto a
shared process pool via `JobExecutionState`. The camera stage instead owns its full lifecycle in a single function
call and relies on the binding's per-archive parallelism rather than a cross-job pool.

---

## How per-system camera names enter as manifest data

The defining property of this stage is that camera output names are **data**, not code. The mapping from source ID
to output filename is produced entirely by `resolve_camera_output_names` reading the camera manifest that the
VideoSystem registered at acquisition time. To add support for another camera, register it in the manifest at
acquisition — no change to `sollertia_forgery.cross_system.video` is required, and there is no source-ID-to-role
registry to extend in this codebase.

Because of this, the concrete camera roles and their assigned source IDs for any specific acquisition system are
out of scope here. For the Mesoscope-VR camera inventory (which source IDs are recorded and what each camera is),
defer to `mesoscope:mesoscope-vr`, the hardware-composition skill that owns that inventory.

---

## Related skills

| Skill                                      | Relationship                                                                                                                                                                                      |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `forging:data-processing-design`           | Owns the shared prepare-then-execute `JobExecutionState` batch model this stage deliberately differs from                                                                                         |
| `forging:behavior-processing`              | Sibling behavior batch pipeline (runtime/microcontroller jobs); the camera stage is run separately by the `process video` CLI command in `interfaces/mesoscope_vr.py`, not by behavior processing |
| `forging:behavior-input-format`            | Defers its camera log archive input description to this skill                                                                                                                                     |
| `forging:behavior-results`                 | Defers its camera timestamp feather output description to this skill                                                                                                                              |
| `forging:microcontroller-primitives`       | Sibling cross_system contract for microcontroller module feather parsing                                                                                                                          |
| `mesoscope:mesoscope-vr`                   | Owns the per-system camera inventory (roles and source IDs) deferred from this skill                                                                                                              |
| `mesoscope:mesoscope-vr-processing-schema` | Owns the producer-to-directory mapping for the processed feathers, referenced here rather than duplicated                                                                                         |
| `ataraxis@video:log-processing`            | Upstream axvs stage that produces the raw camera logs and owns the `CameraManifest` and extraction binding                                                                                        |
| `ataraxis@video:log-input-format`          | Upstream axvs reference for the raw camera log archive format, source-ID semantics, and `CameraManifest`                                                                                          |
| `ataraxis@video:camera-interface`          | Upstream axvs reference for the VideoSystem acquisition surface and system-ID allocation                                                                                                          |

---

## Verification checklist

You MUST verify your work against this checklist before submitting.

```text
- [ ] Source/sink boundary described correctly: raw logs read from session.raw_data.behavior_data_path, named
      feathers written into session.processed_data.video_data_path (processed_data/video_data)
- [ ] No processed_data/behavior_data claim anywhere (behavior_data is the raw-side DataLogger archive directory)
- [ ] No camera_timestamps directory, no camera_timestamps_path field, and no camera_processing_tracker.yaml
- [ ] Discovery is non-recursive glob over *_log.npz; empty list on missing or empty directory
- [ ] Output names come from resolve_camera_output_names reading the camera manifest via CameraManifest, not from a
      hardcoded source-ID registry
- [ ] Output column is frame_time_us, dtype uint64, written as uncompressed Arrow IPC
- [ ] VIDEO_JOB_NAME is "camera_timestamp_extraction" and the job tuple is (VIDEO_JOB_NAME, str(source_id))
- [ ] Tracker is the video ProcessingTracker at session.processed_data.video_tracker_path, resolving to
      video_processing_tracker.yaml inside the same video_data directory that receives the feathers
- [ ] Local mode (job_id is None) runs all jobs sequentially; remote mode runs only the matching job_id
- [ ] Stage is described as self-contained, distinct from the JobExecutionState batch model owned by
      forging:data-processing-design
- [ ] Concrete per-system camera roles / source IDs deferred to mesoscope:mesoscope-vr
- [ ] Upstream axvs symbols (CAMERA_MANIFEST_FILENAME, CameraManifest, extract_logged_camera_timestamps)
      attributed to ataraxis@video skills, not claimed as owned here
```
