---
name: data-processing-design
description: >-
  Documents the platform-general design pattern for sollertia-forgery data processing: agnostic cross_system
  primitives versus per-system specialization that enters as data, feather as cross-stage interchange, the
  prepare-then-execute batch model with trackers and budget-bounded concurrency, and the delegate-to-dependency
  seam. Use when designing a processing or forging pipeline, adding a stage, auditing the cross_system versus
  per-system split, or deciding whether a concern belongs in slf or an upstream dependency.
user-invocable: false
---

# Data processing design

Documents the platform-general design pattern for sollertia-forgery (slf) data processing at the orchestration
and primitive layer. A processing pipeline turns raw acquisition outputs into canonical interchange feathers,
then forges those feathers into analysis-ready datasets. The pattern is the same for every acquisition system;
what differs is the concrete schemas, module mappings, and session selectors — all of which enter the agnostic
machinery purely as data, never as branching code.

This skill is a **pattern skill** — it documents the conventions and contracts that every slf processing
pipeline shares, but does not document any single pipeline's concrete stage logic. For concrete pipelines, see
the per-stage skills (`forging:behavior-processing`, `forging:dataset-forging`, `forging:checksum-verification`)
and the per-system specialization skills (`mesoscope:mesoscope-vr-trial-decomposition` and siblings). It is the
processing-side analog of `experiment:acquisition-system-design`.

---

## Scope

**Covers:**
- The two-layer split: system-agnostic `cross_system` primitives and orchestration versus per-acquisition-system
  specialization modules
- How system specificity enters as **data only** — manifest source names, the `required_session_type` selector,
  and the `(controller_id, module_type, module_id)` per-module registry keyed off filenames
- Feather (uncompressed Arrow IPC) as the canonical cross-stage interchange format and why it is memory-mappable
- The prepare-then-execute batch model: prepare-tools align ProcessingTracker registries, execute-tools dispatch
  via JobExecutionState and a ProcessPoolExecutor, status-tools read trackers from disk without locking execution
- The PendingJob / ActiveJob / JobExecutionState contract and the `(tracker_path, job_id)` dispatch key
- Three concurrency variants under one worker-budget doctrine (plain, per-job-overhead, saturating-allocation)
- Cancellation semantics and the single-active-execution-session constraint per pipeline
- A second agnostic stage shape: the one-shot tool-with-tracker model that runs synchronously and does NOT use
  prepare-then-execute / JobExecutionState, contrasted against the batch model
- Cross-stage seams: which stages slf owns end-to-end versus which it delegates to an upstream dependency agent
- The standard tool surface every batch slf pipeline exposes, plus the agnostic feather-inspection helper

**Does not cover** (delegated):
- The agnostic feather-parsing primitive API surface — see `forging:microcontroller-primitives`.
- The agnostic manifest-driven camera-rename primitive — see `forging:camera-timestamp-extraction`.
- The concrete behavior-processing, dataset-forging, and checksum orchestration workflows (and the checksum
  saturating-allocation constants) — see `forging:behavior-processing`, `forging:dataset-forging`,
  `forging:checksum-verification`.
- Project-manifest generation specifics (columns, status-flag dependency chain, tool surface) — cited here only
  to illustrate the one-shot stage shape; owned by `forging:project-manifest`.
- Concrete Mesoscope-VR module schemas, trial decomposition, fluorescence alignment, and assembled-session
  schemas — see `mesoscope:mesoscope-vr-module-parsing`, `mesoscope:mesoscope-vr-trial-decomposition`,
  `mesoscope:mesoscope-vr-fluorescence-alignment`, `mesoscope:mesoscope-vr-dataset-assembly`.
- Upstream log processing internals — see `ataraxis@video:log-processing` and
  `ataraxis@communication:log-processing`.
- Session discovery, layout, and metadata — owned by the assets plugin.

---

## Two-layer architecture: cross_system primitives versus per-system specialization

An slf processing pipeline is composed of two layers. The dependency direction is strictly one-way: a
system-specific package may import from `cross_system`, but `cross_system` must never import from a
system-specific package.

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│  Layer 1: cross_system — system-agnostic primitives and orchestration               │
│  ──────────────────────────────────────────────────────────                        │
│  orchestration.py  ── PendingJob / ActiveJob / JobExecutionState, prepare_tracker,  │
│                       job_execution_manager, read/derive_tracker_status,            │
│                       analyze_feather_file, RESERVED_CORES                           │
│  microcontroller.py ── module-feather discovery + filename registry parsing         │
│  video.py          ── manifest-driven camera-timestamp extraction pipeline          │
│  dataset.py        ── resolve_dataset (the required_session_type selector)          │
│  checksum.py       ── checksum resolution stage                                     │
│  mcp_tools.py      ── cross-system batch tool surface + one-shot manifest tools     │
└────────────────────────────┬──────────────────────────────────────────────────────┘
                             │ composed and parameterized (data only) by
┌────────────────────────────▼──────────────────────────────────────────────────────┐
│  Layer 2: <system>_vr — per-acquisition-system specialization                       │
│  ─────────────────────────────────────────────                                     │
│  processing.py / microcontrollers.py / forging.py ── concrete stage logic           │
│  processing_mcp_tools.py / forging_mcp_tools.py   ── per-system batch tool surface   │
│      ├── concrete module schemas, conversions, trial decomposition                  │
│      ├── the required_session_type value passed into resolve_dataset                │
│      └── the per-module feather registry keyed off filenames                        │
└───────────────────────────────────────────────────────────────────────────────────┘
```

Layer 1 carries the durable doctrine documented here. Layer 2 supplies the concrete schemas and the data that
parameterizes Layer 1. The same `cross_system` primitives back every system; only the per-system specialization
modules differ between systems.

---

## How system specificity enters as data

The agnostic layer never branches on the acquisition system. System specificity enters through three data
channels, never through conditional code in `cross_system`:

| Channel                  | Mechanism                                                                                      | Source of truth                                              |
|--------------------------|------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Camera output names      | The acquisition-time camera manifest maps each source ID to a colloquial name                   | `resolve_camera_output_names` in `cross_system/video.py`    |
| Session-type selector    | The calling system passes `required_session_type` to restrict which sessions are forgeable      | `resolve_dataset` in `cross_system/dataset.py`              |
| Per-module registry      | The `(controller_id, module_type, module_id)` tuple is parsed from each module feather filename  | `parse_module_feather_name` in `cross_system/microcontroller.py` |

`resolve_camera_output_names` reads the camera manifest that every VideoSystem writes alongside its log archives
and projects each registered source into its canonical `{name}_timestamps.feather` output filename. The manifest
is the sole source of camera output names, so the pipeline requires no acquisition-system-specific configuration:
the colloquial source names recorded at acquisition time directly determine the output names.

`resolve_dataset` accepts a `required_session_type: SessionTypes | None` argument supplied by the calling
acquisition system. When provided, dataset creation is restricted to that system's single forgeable session
type and the first session's type must match it; when `None`, any session type is accepted (the cross-session
consistency check still applies). The agnostic helper holds no hard-coded session type — the system passes it.

`parse_module_feather_name` extracts three integers — controller ID, module type, and module ID — from a
filename following the `controller_{controller_id}_module_{module_type}_{module_id}.feather` convention. The
per-system parser maps each parsed tuple to its concrete handler; the agnostic layer only parses the tuple and
raises a ValueError when the filename does not match. The concrete tuple-to-handler registry is owned by
`forging:microcontroller-primitives` and the per-system parsing skills.

---

## Feather as the cross-stage interchange format

Every stage hands off to the next through **feather files** — uncompressed Apache Arrow IPC files written with
`compression="uncompressed"`. Uncompressed Arrow IPC is memory-mappable, so a downstream stage can read large
arrays without decompressing or copying them into process memory. `process_camera_log` in `cross_system/video.py`
writes its output via `frame.write_ipc(file=output_path, compression="uncompressed")` specifically so the
downstream forging pipeline can memory-map it.

Two feather schema conventions recur across the stack, both recognized by the agnostic inspection helper:

- **The axci five-column module feather schema**: `timestamp_us, command, event, dtype, data`, produced by
  ataraxis-communication-interface log processing and partitioned by `partition_events` (see
  `forging:microcontroller-primitives`).
- **Camera timestamp feathers**: a single `frame_time_us` column holding per-frame acquisition timestamps in
  microseconds since the UTC epoch, written by `process_camera_log`.

`analyze_feather_file` in `cross_system/orchestration.py` reads any feather via `pl.read_ipc` and computes
generic summary statistics — row count, column list, inter-row timing, and sample rows. It recognizes the
canonical time axis by probing, in priority order, the `_TIME_COLUMN_CANDIDATES` tuple
`("timestamp_us", "time_us", "frame_time_us")`, which covers raw axci module feathers (`timestamp_us`), forgery
runtime and microcontroller outputs (`time_us`), and axvs camera timestamp feathers (`frame_time_us`). It needs
at least `_MINIMUM_ROWS_FOR_INTERVALS` (2) rows to compute intervals, and replaces `polars.Binary` columns in
sample rows with a boolean `<column>_has_data` flag to keep the payload JSON-serializable.

---

## The prepare-then-execute batch model

Every batch slf pipeline splits orchestration into three tool roles operating on disk-backed ProcessingTracker
state. This decoupling lets status tools read progress without contending with the running execution session.

1. **Prepare** — resolves the work set (sessions, datasets), then aligns each ProcessingTracker's job registry
   to the expected job set via `prepare_tracker`, and returns enriched job descriptors. It does not start
   execution. It is idempotent.
2. **Execute** — takes the prepared descriptors, builds the package-specific PendingJob subclasses, constructs a
   JobExecutionState, and starts a single background `job_execution_manager` thread that owns one
   ProcessPoolExecutor. Returns immediately with a `started` flag.
3. **Status / timing / overview** — read ProcessingTracker YAML files from disk via `read_tracker_status` and
   `derive_tracker_status` without touching the execution state lock, so they can be polled at any time.

`prepare_tracker(tracker, jobs)` aligns a tracker to a list of `(job_name, specifier)` tuples. If the file does
not exist, it initializes from scratch. If the file contains foreign IDs (entries not in the expected set), it
warns about architectural drift, resets, and reinitializes. If the file holds a strict subset, it additively
registers the missing entries without clobbering existing state. If it already matches exactly, it is a no-op.
This regeneration strategy is the single point where stale job registries are detected and repaired.

---

## Job identity, trackers, and JobExecutionState

The orchestration primitives in `cross_system/orchestration.py` are generic over a `PendingJob` subclass and
carry no domain logic:

| Type                | Role                                                                                                    |
|---------------------|--------------------------------------------------------------------------------------------------------|
| `PendingJob`        | Base dataclass: `tracker_path` + `job_id`, with a `dispatch_key` property returning `(tracker_path, job_id)` |
| `ActiveJob`         | Pairs a pending job with its in-flight `Future` on the shared process pool                              |
| `JobExecutionState` | Holds the `worker` callable, `all_jobs`, `pending_queue`, `active_jobs`, `worker_budget`, `max_parallel_jobs`, `lock`, `manager_thread`, and `canceled` flag |

Job identity is the `(job_name, specifier)` tuple. ProcessingTracker derives a stable hexadecimal job ID from it
via `generate_job_id(job_name=..., specifier=...)`, and the dispatch key `(tracker_path, job_id)` uniquely
identifies a job across the entire batch. Each package subclasses `PendingJob` with the extra fields its worker
needs (session path, output name, resolved worker count). The `worker` callable MUST be a picklable
module-level function accepting a single pending-job argument, because `job_execution_manager` dispatches it via
`ProcessPoolExecutor.submit`.

The worker is responsible for transitioning its job through the tracker's `start_job` → `complete_job` /
`fail_job` lifecycle. The manager drains each completed future's result under `contextlib.suppress(Exception)`
so a worker exception never silently crashes the daemon thread; the tracker remains the authoritative record of
each job's terminal outcome.

---

## Worker budget and concurrency contract

All concurrency derives from one shared doctrine: a CPU budget resolved from the machine's core count minus
`RESERVED_CORES` (2, in `cross_system/orchestration.py`), via
`ataraxis_base_utilities.resolve_worker_count(requested_workers=..., reserved_cores=RESERVED_CORES)`. The manager
resolves its concurrency cap as `worker_budget` when `max_parallel_jobs <= 0`, otherwise
`min(worker_budget, max_parallel_jobs)`, and sizes a single ProcessPoolExecutor to that cap for the session.

Pipelines pick one of three variants depending on each job's internal CPU footprint:

| Variant               | Where each job's parallelism lives                    | How concurrency is bounded                                                  | Worked example                       |
|-----------------------|-------------------------------------------------------|-----------------------------------------------------------------------------|--------------------------------------|
| Plain budget          | One core per job (job is a single subprocess)         | `worker_budget` alone caps concurrent jobs; `max_parallel_jobs` left at -1  | behavior processing                  |
| Per-job overhead      | Each job spawns a small internal pool                 | `max_parallel_jobs` floored by `worker_budget // _CORES_PER_JOB`            | forging                              |
| Saturating allocation | Each job internally parallelizes over many cores      | Budget split across jobs at a preferred per-job worker count, with min/max  | checksum                             |

**Plain budget** (behavior processing): the execute tool resolves a single `worker_budget` and leaves
`max_parallel_jobs` at its default; the budget simultaneously caps the pool size and the maximum number of
concurrent jobs, since each job is one subprocess.

**Per-job overhead** (forging): a module-level `_CORES_PER_JOB` constant (3, in the forging tool: one subprocess
plus a two-thread assembly pool) divides the resolved budget to yield a core-bounded ceiling
`max(1, resolved_budget // _CORES_PER_JOB)`, and the effective parallelism is `min(requested_parallel, that
ceiling)`. This bounds per-session memory peaks independently of CPU allocation.

**Saturating allocation** (checksum): when both worker count and parallelism are automatic, the allocator
maximizes the number of concurrent jobs at a preferred per-job worker count, then reduces parallelism until each
job clears a minimum-worker floor, rounding worker counts down to a clean multiple and capping each job at a hard
ceiling. The concrete preferred / minimum / maximum constants belong to the checksum stage and are documented in
`forging:checksum-verification`; the inspection point in source is
`_resolve_checksum_saturating_allocation` in `cross_system/mcp_tools.py`.

---

## The one-shot tool-with-tracker stage shape

Not every agnostic stage uses the batch model. A second shape exists for operations that run synchronously to
completion in a single call and produce one artifact: the **one-shot tool-with-tracker** stage. Project-manifest
generation is the canonical instance.

| Aspect            | Batch model                                                   | One-shot model                                              |
|-------------------|--------------------------------------------------------------|------------------------------------------------------------|
| Tools             | prepare → execute → status / timing / cancel / reset / clean  | generate → status → clean                                  |
| Execution         | Background daemon thread + ProcessPoolExecutor                | Synchronous, in the calling process                        |
| State object      | JobExecutionState with a pending/active queue                 | None — no JobExecutionState                                |
| Tracker           | Per-session/per-dataset tracker, one job per unit             | One project-level tracker recording a single outcome       |
| Cancellation      | Supported (clear pending queue)                               | Not applicable                                             |

The generate tool delegates to the pipeline function (`generate_project_manifest`), which owns its own
ProcessingTracker lifecycle and writes a single `{project_name}_manifest.feather`; the status tool reads that
tracker via `read_tracker_status`; the clean tool deletes the tracker, the feather, and their companion `.lock`
files. There is no prepare step and no execute/dispatch step. Use this shape when the work is a single
synchronous scan rather than a fan-out of independent per-unit jobs. The concrete manifest columns and status
chain are owned by `forging:project-manifest`.

---

## Cancellation and the single-active-execution-session constraint

Each batch pipeline keeps a single module-level execution-state variable, so only **one** execution session can
be active per pipeline at a time. The execute tool refuses to start a new session while the previous manager
thread is still alive, returning an error directing the caller to cancel first.

Cancellation is cooperative and atomic. The cancel tool acquires `state.lock`, sets `state.canceled = True`,
snapshots and clears `state.pending_queue` under the lock, and records the count of still-active jobs.
`job_execution_manager` consults the `canceled` flag each poll cycle: once set, it dispatches no new jobs but
lets already-running futures finish naturally. The tool then reads tracker files to tally terminal outcomes for
its final-state report. In-flight work is never killed mid-execution — only the pending queue is drained.

---

## Cross-stage seams: slf-owned stages versus delegated dependency stages

slf does not own every stage end-to-end. The pipeline crosses two seams where it consumes the feather outputs of
an upstream dependency agent instead of computing them itself:

| Stage                          | Owner                                            | slf's role                                                   |
|--------------------------------|--------------------------------------------------|-------------------------------------------------------------|
| Camera log → frame timestamps  | ataraxis-video-system (axvs)                      | Calls the binding in-process, renames output to a feather   |
| Microcontroller log → module feather | ataraxis-communication-interface (axci)     | Consumes the five-column module feathers                    |
| Camera-timestamp pipeline      | slf (`cross_system/video.py`)                     | Owns discovery, manifest resolution, tracker, feather write |
| Behavior processing / forging  | slf (per-system specialization)                   | Owns end-to-end                                             |
| Checksum / project manifest    | slf (`cross_system`)                              | Owns end-to-end                                             |

The camera-timestamp stage is illustrative of the seam: `process_camera_log` defers the
`ataraxis_video_system` import to call time and invokes `extract_logged_camera_timestamps` in-process — slf does
not reimplement timestamp extraction, it calls the dependency binding and owns only the discovery, manifest-driven
renaming, tracker, and feather write around it. When a concern is already solved by an upstream dependency
pipeline, delegate to it and consume its feathers; reserve slf ownership for the discovery, orchestration,
renaming, and cross-stage assembly that the dependency does not provide. The upstream internals are owned by
`ataraxis@video:log-processing` and `ataraxis@communication:log-processing`.

---

## The standard batch pipeline tool surface

Every batch slf pipeline exposes the same tool roles, named per pipeline. Implemented across
`cross_system/mcp_tools.py` (checksum, project manifest) and the per-system `*_mcp_tools.py` modules:

| Role             | Purpose                                                                                          |
|------------------|------------------------------------------------------------------------------------------------|
| `prepare_*`      | Resolve the work set and align trackers via `prepare_tracker`; return enriched job descriptors  |
| `execute_*`      | Build PendingJobs, start the JobExecutionState manager thread; return a `started` flag           |
| `*_status`       | Read trackers from disk for per-job progress of the active session                              |
| `*_timing`       | Report per-job and aggregate timing from tracker state                                          |
| `cancel_*`       | Clear the pending queue under the state lock; let in-flight jobs finish                         |
| `reset_*_jobs`   | Reset selected or all tracker jobs to scheduled for re-runs                                     |
| `*_batch_status_overview` | Summarize tracker status across every session/dataset under a root directory           |
| `clean_*`        | Delete a named output subtree (and/or trackers and lock files)                                  |

In addition, the agnostic `analyze_feather_file` helper backs the per-pipeline inspection tools (the
`verify_*_output` and `query_*_data` tools), giving every pipeline a uniform way to summarize and sample its
feather outputs without bespoke parsing.

---

## Mesoscope-VR as a worked example

The Mesoscope-VR system is the current consumer of every pattern in this skill. It appears here only to make the
abstractions concrete; all of its specific schemas are deferred to the mesoscope plugin.

- **Plain-budget batch pipeline**: behavior processing in `mesoscope_vr/processing_mcp_tools.py`. The execute
  tool resolves a single `worker_budget` against `RESERVED_CORES` and leaves `max_parallel_jobs` at -1.
- **Per-job-overhead batch pipeline**: dataset forging in `mesoscope_vr/forging_mcp_tools.py`, with
  `_CORES_PER_JOB = 3` flooring concurrency at `worker_budget // _CORES_PER_JOB`. Its prepare tool passes
  `required_session_type=SessionTypes.MESOSCOPE_EXPERIMENT` into `resolve_dataset` — the concrete value of the
  agnostic session selector. The forging job name is `session_data_assembly`.
- **Saturating-allocation + one-shot pipelines**: checksum and project-manifest generation in
  `cross_system/mcp_tools.py`. Checksum uses `_resolve_checksum_saturating_allocation`; manifest generation is
  the one-shot generate / status / clean shape.
- **System specificity as data**: the camera manifest names, `SessionTypes.MESOSCOPE_EXPERIMENT`, and the parsed
  `(controller_id, module_type, module_id)` tuples are the only system-specific inputs the agnostic layer sees.

For the Mesoscope-VR-specific surface — concrete module schemas, trial decomposition, fluorescence alignment, and
assembled-session schemas — see `mesoscope:mesoscope-vr-module-parsing`,
`mesoscope:mesoscope-vr-trial-decomposition`, `mesoscope:mesoscope-vr-fluorescence-alignment`, and
`mesoscope:mesoscope-vr-dataset-assembly`.

---

## Maintenance contract

This skill documents durable design patterns. It is updated when:

- A new orchestration primitive or stage shape is added to `cross_system` (e.g., a third batch contract beyond
  prepare-then-execute and one-shot).
- A new concurrency variant is adopted across the platform.
- The PendingJob / JobExecutionState contract, the tracker-alignment doctrine, or the dispatch-key identity
  changes.
- A new cross-stage seam (owned versus delegated) is established or moved.

This skill is NOT updated when:

- A specific pipeline gains a new job type, column, or schema — that's the per-stage or per-system skill's domain.
- A specific worker's internal computation changes — the *pattern* it follows is what's documented here.

When unsure whether a change belongs here or in a per-stage/per-system skill, ask: "Does this apply to every slf
processing pipeline, or only this one?" The pattern skill answers "every"; per-stage and per-system skills answer
"only this one."

---

## Related skills

| Skill                                          | Relationship                                                                                  |
|------------------------------------------------|----------------------------------------------------------------------------------------------|
| `forging:microcontroller-primitives`           | The agnostic module-feather parsing primitives and the filename-tuple registry contract.       |
| `forging:camera-timestamp-extraction`          | The agnostic manifest-driven camera-rename stage; worked example of the delegate-to-axvs seam.  |
| `forging:behavior-processing`                  | Concrete plain-budget batch pipeline that applies this doctrine.                               |
| `forging:dataset-forging`                      | Concrete per-job-overhead batch pipeline that applies this doctrine.                           |
| `forging:checksum-verification`                | Concrete saturating-allocation pipeline; owns its allocation constants.                        |
| `forging:project-manifest`                     | Concrete one-shot tool-with-tracker stage; owns the manifest columns and status chain.          |
| `mesoscope:mesoscope-vr-module-parsing`        | Concrete Mesoscope-VR module schemas and conversions.                                          |
| `mesoscope:mesoscope-vr-trial-decomposition`   | Concrete Mesoscope-VR runtime decoding and trial decomposition.                                |
| `mesoscope:mesoscope-vr-fluorescence-alignment`| Concrete Mesoscope-VR fluorescence / ScanImage alignment.                                      |
| `mesoscope:mesoscope-vr-dataset-assembly`      | Concrete Mesoscope-VR assembled-session schemas.                                               |
| `ataraxis@video:log-processing`                | Upstream dependency stage producing camera timestamp feathers consumed here.                   |
| `ataraxis@communication:log-processing`        | Upstream dependency stage producing microcontroller module feathers consumed here.             |
| `experiment:acquisition-system-design`         | The acquisition-side pattern skill this one mirrors; processing is its static-composition analog. |

---

## Verification checklist

```text
When designing a new processing pipeline or auditing an existing one:

Two-layer split:
- [ ] Agnostic primitives and orchestration live in cross_system; concrete logic lives in the per-system package
- [ ] cross_system never imports from a system-specific package (dependency direction is one-way)
- [ ] System specificity enters as data only (manifest names, required_session_type, filename-parsed registry),
      never as a branch on the acquisition system inside cross_system

Interchange format:
- [ ] Cross-stage handoffs are uncompressed Arrow IPC feathers (compression="uncompressed") for memory-mapping
- [ ] The time axis column matches a recognized candidate (timestamp_us / time_us / frame_time_us)

Batch model:
- [ ] prepare aligns the tracker via prepare_tracker and does NOT start execution; it is idempotent
- [ ] execute builds a PendingJob subclass, constructs JobExecutionState, starts one job_execution_manager thread
- [ ] The worker callable is a picklable module-level function taking a single pending-job argument
- [ ] The worker drives its tracker through start_job -> complete_job / fail_job
- [ ] status / timing / overview read trackers from disk without touching the execution-state lock
- [ ] Job identity is (job_name, specifier); the dispatch key is (tracker_path, job_id)

Concurrency:
- [ ] Budget is resolved from cores minus RESERVED_CORES via resolve_worker_count
- [ ] One concurrency variant is chosen deliberately (plain / per-job-overhead / saturating-allocation)
- [ ] Per-job-overhead pipelines floor parallelism by worker_budget // <cores-per-job>
- [ ] Saturating-allocation constants (preferred/minimum/maximum) live in the owning stage, not this pattern

Cancellation and sessions:
- [ ] A single module-level execution state enforces one active session per pipeline
- [ ] execute refuses to start while the prior manager thread is alive
- [ ] cancel clears the pending queue under state.lock and lets in-flight jobs finish

One-shot stages:
- [ ] One-shot stages run synchronously, use a single project-level tracker, and have NO JobExecutionState
- [ ] One-shot stages expose generate / status / clean, not prepare / execute

Cross-stage seams:
- [ ] Stages already owned by an upstream dependency are delegated (binding called in-process), not reimplemented
- [ ] slf owns only the discovery, manifest resolution, tracker, renaming, and assembly around delegated stages

Tool surface:
- [ ] Batch pipeline exposes prepare / execute / status / timing / cancel / reset / batch-overview / clean
- [ ] Feather inspection reuses analyze_feather_file rather than bespoke parsing
```
