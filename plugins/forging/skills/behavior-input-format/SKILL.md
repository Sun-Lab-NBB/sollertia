---
name: behavior-input-format
description: >-
  Documents the behavior-processing input artifacts consumed by the sollertia-forgery pipeline:
  the Mesoscope-VR runtime NPZ archive, the camera-timestamp feathers from the cross-system video
  stage, and the raw microcontroller log archives the pipeline extracts in-process via the
  ataraxis-communication-interface binding. Use when the user asks about raw inputs, module
  eligibility, or why a behavior job is missing on disk.
user-invocable: false
---

# Behavior input format

Authoritative reference for the **behavior-processing-specific** input artifacts: the runtime NPZ
archive, the camera timestamp feathers (re-extracted from raw `ataraxis-video-system` camera logs by a
separate cross-system stage), and the raw microcontroller log archives that the behavior pipeline extracts
in-process into module feathers via the `ataraxis-communication-interface` binding. This skill only covers
artifacts that are unique to the behavior subsystem. Session metadata, directory layout, hardware state, and
experiment configuration are owned by the assets plugin and are referenced, not redocumented.

---

## Scope

**Covers:**
- Runtime NPZ archive format, source ID, message codes, and the runtime job contract
- Camera timestamp feather handoff from axvs (renaming mechanism deferred to `forging:camera-timestamp-extraction`)
- In-process microcontroller log extraction via the axci binding and the resulting module feathers
  (discovery/partition primitives deferred to `forging:microcontroller-primitives`)
- Module-to-hardware-state-field eligibility table and the `mesoscope_vr/microcontrollers.py` module registry
- Cross-library handoff contract (raw inputs consumed from axvs / axci by forgery)

**Does not cover:**
- Session directory layout, `raw_data/` / `processed_data/` hierarchy (see `assets:session-data` and
  `assets:project-hierarchy`)
- Session marker (`session_data.yaml`), `SessionData`, and `SessionTypes` (see `assets:session-data`)
- Hardware state YAML (`MesoscopeHardwareState`) authoring and validation (see `assets:session-hardware-state`)
- Experiment configuration YAML (`MesoscopeExperimentConfiguration`) authoring and validation
  (see `assets:experiment-configuration`)
- Session descriptor YAMLs (see `assets:session-descriptors`)
- Session discovery and filtering (see `assets:session-discovery`)
- Output formats, verification, and data querying (see `forging:behavior-results`)
- Batch orchestration workflow and the processing doctrine (see `forging:behavior-processing` and
  `forging:data-processing-design`)
- The agnostic feather discovery, naming, and event-partition primitives (see `forging:microcontroller-primitives`)
- The manifest-driven camera timestamp re-extraction and renaming stage (see `forging:camera-timestamp-extraction`)
- Per-module conversions, output column schemas, and calibration fields (see `mesoscope:mesoscope-vr-module-parsing`)
- The axci log-processing binding internals and the axvs acquisition protocol (see
  `ataraxis@communication:log-processing` and `ataraxis@video:log-processing`)

**Note:** `ataraxis@video:*` and `ataraxis@communication:*` refer to the **video** and **communication**
plugins from the [ataraxis marketplace](https://github.com/Sun-Lab-NBB/ataraxis).

---

## Session eligibility

A session is eligible for behavior processing only if its `session_type` is in
`PROCESSABLE_SESSION_TYPES`:

| Session type           | Eligible | Notes                                                     |
|------------------------|----------|-----------------------------------------------------------|
| `LICK_TRAINING`        | yes      | Runtime and microcontroller jobs                          |
| `RUN_TRAINING`         | yes      | Runtime and microcontroller jobs                          |
| `MESOSCOPE_EXPERIMENT` | yes      | Same as training plus experiment-specific runtime outputs |
| (anything else)        | no       | Returned with `eligible=False` and no error               |

Use `assets:session-discovery` with
`session_types=["lick_training", "run_training", "mesoscope_experiment"]` to discover eligible
sessions. Eligibility is re-validated at prepare time by `prepare_behavior_processing_batch_tool`,
which calls `discover_behavior_jobs`: it re-loads `SessionData`, checks the type against
`PROCESSABLE_SESSION_TYPES`, loads the hardware state from the canonical
`session.raw_data.hardware_state_path` (raising `FileNotFoundError` if absent), and reports the
session with `"No processable behavior jobs discovered for this session."` if neither a runtime
archive nor an eligible microcontroller module feather is found on disk.

Only `MESOSCOPE_EXPERIMENT` sessions produce the experiment-specific outputs
(`reinforcing_guidance_state_data.feather`, `aversive_guidance_state_data.feather`,
`vr_cue_data.feather`, `vr_trigger_zone_data.feather`, `trial_data.feather`). Training sessions
produce only the state feathers (`system_state_data.feather`, `runtime_state_data.feather`) plus
any eligible microcontroller and camera outputs.

---

## Where behavior inputs live

Behavior processing reads three distinct artifact classes from a session directory. The directory
hierarchy itself is documented in `assets:session-data` and `assets:project-hierarchy`; this skill only captures
the subset the behavior pipeline touches:

```text
{session_root}/
├── raw_data/
│   ├── session_data.yaml                          ← see assets:session-data (not consumed here)
│   ├── hardware_state YAML                        ← see assets:session-hardware-state (gates module jobs)
│   ├── experiment_configuration YAML              ← see assets:experiment-configuration (experiments only)
│   ├── behavior_data/
│   │   ├── 1_log.npz                              ← runtime NPZ archive (this skill)
│   │   ├── {controller_id}_log.npz               ← raw microcontroller log archives (behavior input)
│   │   └── extraction_configuration.yaml          ← axci extraction config (behavior input)
│   └── camera_data/
│       └── {source_id}_log.npz                    ← raw VideoSystem camera logs (see camera-timestamp-extraction)
└── processed_data/
    ├── behavior_data/
    │   └── {name}_timestamps.feather              ← re-extracted camera feathers (camera-timestamp-extraction)
    └── microcontroller_data/
        └── controller_{cid}_module_{type}_{id}.feather  ← module feathers produced in-process by the axci extraction stage; discovery/parse owned by forging:microcontroller-primitives
```

The pipeline resolves each artifact class from a fixed canonical subdirectory exposed by `SessionData`. The
runtime archive is resolved at a fixed source ID directly under `raw_data/behavior_data/`, alongside the raw
microcontroller `{controller_id}_log.npz` archives and the `extraction_configuration.yaml` that the behavior
pipeline's in-process axci extraction stage consumes. The raw camera logs live under `raw_data/camera_data/`.
The extracted module feathers are discovered directly under `processed_data/microcontroller_data/` with a
non-recursive glob (not `rglob`) after the behavior pipeline writes them there. The non-recursive discovery and
naming contract for module feathers is owned by `forging:microcontroller-primitives`; the camera re-extraction
stage is owned by `forging:camera-timestamp-extraction`.

---

## Runtime log archive

### Producer

The Mesoscope-VR acquisition runtime owns the runtime `DataLogger` for every Mesoscope-VR session and persists
every message written to the logger's input queue as an individual `.npy` file. Runtime messages use a reserved
source ID of `1` — this ID is reserved for the runtime itself and MUST NOT collide with any camera or
microcontroller source ID on the same DataLogger. Cameras and microcontrollers managed by the same acquisition
runtime share this DataLogger but use higher IDs.

After the session ends, the acquisition-side preprocessing runs `assemble_log_archives()` from
`ataraxis-data-structures`, which groups per-source `.npy` files into one uncompressed `.npz` archive per source
ID. The runtime archive is the only archive with source ID `1`.

### Required location

The forging pipeline discovers the runtime archive at the fixed canonical path
`session.raw_data.behavior_data_path / {RUNTIME_SOURCE_ID}_log.npz` (source ID `1`), resolved by
`find_log_archive` in `sollertia_forgery.mesoscope_vr.runtime`. Because the runtime DataLogger always writes to a
fixed source ID, at most one archive is expected per session, so discovery is a direct path lookup rather than a
recursive search.

### Format

#### Archive file

Each session produces exactly one uncompressed NPZ log archive at `1_log.npz`. The archive is
built in two phases by classes defined in
`ataraxis_data_structures.data_loggers`:

1. **Runtime logging** (`DataLogger` + `LogPackage` in `serialized_data_logger.py`). The runtime
   feeds the logger by putting `LogPackage(source_id, acquisition_time, serialized_data)` instances
   onto its multiprocessing input queue, where:

   - `source_id: np.uint8` — the producing system's identifier (fixed at `1` for the runtime, see
     Producer section above).
   - `acquisition_time: np.uint64` — microseconds elapsed since the session's onset; the onset
     message itself uses `0` as a sentinel.
   - `serialized_data: NDArray[np.uint8]` — the domain-specific payload (for runtime messages, the
     code-prefixed state arrays and cue sequences documented in the subsections below).

   `LogPackage.data` concatenates the three fields verbatim into a single 1-D `uint8` buffer:

   ```text
   [source_id: 1 byte][acquisition_time: 8 bytes (uint64, host-native)][serialized_data: N bytes]
   ```

   Total length is `9 + N` bytes per message. The 8 acquisition-time bytes are written via
   `np.frombuffer(buffer=self.acquisition_time, dtype=np.uint8)` — a raw byte reinterpret of the
   `uint64`, so the endianness is whatever the host uses (little-endian on every supported
   platform). `DataLogger` then saves the buffer as a standalone `.npy` file under the logger's
   output directory, naming it `{source_id:03d}_{acquisition_time:020d}.npy` (source ID
   zero-padded to 3 digits — enough to cover the full `uint8` range — and acquisition time
   zero-padded to 20 digits — the maximum width of a decimal `uint64`).

2. **Post-runtime assembly** (`assemble_log_archives()`, also from `ataraxis-data-structures`).
   After the session ends, this helper groups every `.npy` file in the logger's output directory
   by source ID and packages each group into one uncompressed NPZ archive (effectively
   `np.savez`, not `np.savez_compressed`). NPZ is a ZIP container holding one `.npy` per entry;
   each entry's key is the source `.npy` filename with the `.npy` suffix stripped, so the on-disk
   archive looks like:

   ```text
   1_log.npz
   ├── 001_00000000000000000000   ← onset message (acquisition_time = 0)
   ├── 001_00000000000000034821   ← first elapsed-time message
   ├── 001_00000000000000127544
   │   ...
   └── 001_{acquisition_time:020d}
   ```

   The entries are stored (not deflated), so random access and `mmap` over the ZIP payload are
   cheap — which is exactly how `LogArchiveReader` reads them. Because both header fields are
   fixed-width zero-padded, lexicographic key order matches chronological message order.

The read side lives in `LogArchiveReader` (`log_archive_reader.py`). Its `iter_messages()`
generator opens the NPZ with `np.load(..., mmap_mode="r")`, slices every non-onset entry into its
three components, strips the leading 9-byte `[source_id][acquisition_time]` header, and yields a
frozen `LogMessage(timestamp_us, payload)` dataclass where:

- `timestamp_us = onset_timestamp_us + acquisition_time`, giving the message's absolute UTC
  timestamp in microseconds since epoch (see the Onset message subsection for how
  `onset_timestamp_us` is discovered).
- `payload` is a `uint8` copy of byte 9 onward of the entry (`iter_messages` returns
  `message[9:].copy()`) — in other words, the original `serialized_data` array handed to `LogPackage`,
  bit-for-bit.

The forging runtime processor (`sollertia_forgery.mesoscope_vr.runtime.process_runtime_data`) only
ever talks to `LogArchiveReader` — it never opens the NPZ directly and never parses the 9-byte
header. Every nuance in the Message payloads and Cue sequence messages subsections below describes
the shape of `payload` only, not the header.

#### Onset message

The first message written to the runtime log has `acquisition_time=0` and carries the session's
absolute UTC onset timestamp as its payload. `LogArchiveReader` reads this payload as a 64-bit
integer (`onset = np.uint64(entry[9:].view(np.int64).item())`) during onset discovery and adds it to
every subsequent message's elapsed time to produce the absolute UTC timestamp exposed as
`message.timestamp_us`. The exact acquisition-side serialization of the onset payload is an
acquisition-runtime detail not verifiable from slf; only the reader's interpretation is documented
here. Without the onset the reader cannot resolve absolute time, so archives with a missing or
corrupt onset are unreadable.

#### Message payloads

After the onset, every runtime message is a single `np.uint8` array written as
`LogPackage.serialized_data`. The reader returns this array verbatim as `message.payload`.
Payloads have **two disjoint shapes**:

1. **Code-prefixed state messages** — short arrays (≤ 9 bytes) whose first byte is a message code
   in the range 1-5.
2. **Cue sequence messages** — long arrays (> 500 bytes) with no leading code byte.

The forging runtime reader dispatches on payload length first (cue sequence detection), then on
`payload[0]` for the state-message codes.

The runtime message code enum is owned by `mesoscope:mesoscope-vr-runtime`, and the forging runtime
reader mirrors every code as a module-level constant (`_SYSTEM_STATE_CODE`, `_RUNTIME_STATE_CODE`,
`_REINFORCING_GUIDANCE_STATE_CODE`, `_AVERSIVE_GUIDANCE_STATE_CODE`, `_DISTANCE_SNAPSHOT_CODE`) in
`sollertia_forgery.mesoscope_vr.runtime`. These two definitions MUST stay in lock-step — any new
writer code requires a matching reader constant, or forgery will silently drop unrecognized
messages.

**Code-prefixed payload layouts:**

| Code | Name (sollertia-experiment enum) | Payload layout                                         | Bytes |
|------|----------------------------------|--------------------------------------------------------|-------|
| 1    | `SYSTEM_STATE`                   | `[1: uint8][state_code: uint8]`                        | 2     |
| 2    | `RUNTIME_STATE`                  | `[2: uint8][state_code: uint8]`                        | 2     |
| 3    | `REINFORCING_GUIDANCE_STATE`     | `[3: uint8][enabled: uint8 (0 or 1)]`                  | 2     |
| 4    | `AVERSIVE_GUIDANCE_STATE`        | `[4: uint8][enabled: uint8 (0 or 1)]`                  | 2     |
| 5    | `DISTANCE_SNAPSHOT`              | `[5: uint8][traveled_distance: 8 bytes little-endian]` | 9     |

**SYSTEM_STATE / RUNTIME_STATE:** The trailing `state_code` byte is the numeric code of the
destination state. State enums live in sollertia-experiment and are not interpreted by forgery — codes
are stored verbatim in `system_state_data.feather` / `runtime_state_data.feather` for downstream
consumers to resolve.

**REINFORCING_GUIDANCE_STATE / AVERSIVE_GUIDANCE_STATE:** Emitted only during experiment sessions
when the corresponding Unity-side guidance mode toggles. The trailing byte is a boolean
(`0` = disabled, `1` = enabled). Training sessions never emit these codes, and forgery also gates
on the `experiment_configuration` before writing the guidance feathers, so non-experiment sessions
skip them even if the archive somehow contained them.

**DISTANCE_SNAPSHOT:** Logged when the VR wall cue sequence changes. The 8-byte tail is the
cumulative traveled distance in centimeters as a little-endian float64; the forging reader parses it
via `payload[1:9].view(dtype="<f8")[0]` and appends it to a list of distance snapshots. These
snapshots are consumed as distance breakpoints by the cue-sequence decomposer to stitch consecutive
cue sequences together — one breakpoint per sequence boundary, so the number of breakpoints must
equal the number of cue sequences minus one (the decomposer raises `ValueError` otherwise).

#### Cue sequence messages

Cue sequence messages are emitted only during experiment sessions, when the runtime receives a new
sequence from Unity over MQTT:

```python
self._unity_state.cue_sequence = np.array(
    json.loads(data[1].decode("utf-8"))["cue_sequence"], dtype=np.uint8
)
self._logger.input_queue.put(
    LogPackage(
        source_id=self._source_id,
        acquisition_time=np.uint64(self._timestamp_timer.elapsed),
        serialized_data=self._unity_state.cue_sequence,
    )
)
```

The payload is a raw `np.uint8` array where each byte encodes a VR wall cue (one byte per spatial
unit along the corridor). Cue sequences are typically several thousand bytes long, so forgery
uses a **payload-length threshold** to distinguish them from state messages:

```python
_CUE_SEQUENCE_MIN_LENGTH = 500  # bytes

if len(payload) > _CUE_SEQUENCE_MIN_LENGTH and experiment_configuration is not None:
    # treat as cue sequence
```

Any payload longer than 500 bytes is assumed to be a cue sequence. State messages (codes 1-5) are
always ≤ 9 bytes, so the threshold is safe with a huge margin. Training sessions never emit cue
sequences, and the reader also gates on `experiment_configuration is not None`, so non-experiment
sessions skip cue sequence handling entirely.

#### Timestamp resolution

`LogArchiveReader` returns each message's `timestamp_us` as absolute UTC microseconds, computed as
`onset_utc_us + elapsed_us`, where `onset_utc_us` is read from the onset message's payload (see
the Onset message subsection above for how the reader discovers it). Forgery's runtime processor
stores these timestamps verbatim in the `time_us` column of every output feather, so downstream
analysis can cross-reference runtime events against microcontroller and camera timestamps without
any additional alignment step.

### Job produced

`(runtime_processing, "1")` — exactly one per session whenever `1_log.npz` exists.

---

## Camera timestamp feather files

### Producer

The raw camera frame timestamps originate with the `ataraxis-video-system` (axvs) VideoSystem, which writes one
raw `{source_id}_log.npz` archive plus a shared camera manifest under `session.raw_data.camera_data_path`. The
forging pipeline does NOT consume the upstream axvs `camera_timestamps/` landing directory and does NOT hardlink
any feather: it re-extracts the logged timestamps from the raw camera logs and writes a `{name}_timestamps.feather`
into `session.processed_data.video_data_path`, where the per-camera output name comes from the camera manifest
recorded at acquisition time (not from a hardcoded source-ID registry).

Camera-timestamp extraction is a fully separate cross-system pipeline (`cross_system/video.py`, job
name `camera_timestamp_extraction` exposed as `VIDEO_JOB_NAME`, backed by its own
`video_tracker_path` tracker). It is NOT a behavior-pipeline job — `_discover_jobs` never produces a
camera job, and `BehaviorJobNames` has only `RUNTIME` and `MICROCONTROLLER`. The complete camera
stage — raw-log discovery, source-ID parsing, manifest-driven output naming, the `frame_time_us`
re-extraction, and its self-contained tracker-backed orchestrator — is owned by
`forging:camera-timestamp-extraction`. This skill documents only that camera timestamps enter behavior processing
as `{name}_timestamps.feather` files under `processed_data/video_data/`. The concrete per-system camera roles
(for example, face and body cameras) are part of the Mesoscope-VR hardware composition owned by
`mesoscope:mesoscope-vr` and
enter purely as manifest data, so no camera names are hardcoded here.

---

## Microcontroller module feather files

### Producer

The behavior pipeline itself, in-process. As its first stage, `run_behavior_processing_pipeline`
calls the slf-owned `_extract_microcontroller_logs` helper (`mesoscope_vr/processing.py`), which
imports `run_log_processing_pipeline` from `ataraxis-communication-interface` and runs the axci
log-processing binding in-process. The binding reads the raw `{controller_id}_log.npz` archives plus
`extraction_configuration.yaml` from `session.raw_data.behavior_data_path` and writes one feather
file per `(controller_id, module_type, module_id)` tuple into
`session.processed_data.microcontroller_data_path`. There is no required prior pipeline run: the real
upstream input is the raw log archives plus the extraction config, not pre-existing module feathers.
Only the axci binding and the communication protocol internals are owned by
`ataraxis@communication:log-processing`.

### Required location

After the in-process extraction stage writes them, the behavior pipeline discovers the module
feathers directly under `session.processed_data.microcontroller_data_path` with a non-recursive glob
(`controller_*_module_*.feather`), not a recursive `rglob` — files are expected at the canonical
microcontroller data location, not nested at arbitrary depth. The discovery glob and the
filename-to-tuple parse (including its `ValueError`-on-malformed-name contract) are agnostic primitives owned by
`forging:microcontroller-primitives`.

### Naming convention

```text
controller_{controller_id}_module_{module_type}_{module_id}.feather
```

The filename is split on `_` and must yield exactly five parts; malformed filenames raise `ValueError`. See
`forging:microcontroller-primitives` for the `(controller_id, module_type, module_id)` parse contract.

### Schema

The axci log-processing binding writes a standard five-column feather schema for every module feather:

| Column         | Dtype    | Description                                                        |
|----------------|----------|--------------------------------------------------------------------|
| `timestamp_us` | `UInt64` | Message timestamp in microseconds since UTC epoch                  |
| `command`      | `UInt8`  | Command code the module was executing when the message was sent    |
| `event`        | `UInt8`  | Event code identifying the message type                            |
| `dtype`        | `String` | NumPy dtype string for the data payload (null if no data)          |
| `data`         | `Binary` | Serialized binary payload (null if no data)                        |

Behavior processing reads these files via memory-mapped Polars IPC and partitions rows by `event` in a single
pass. axci writes uncompressed IPC, which is what makes memory-mapped reads safe. The single-pass event-partition
and typed-array extraction helpers are agnostic primitives owned by `forging:microcontroller-primitives`.

### Module registry

Only modules registered in `sollertia_forgery.mesoscope_vr.microcontrollers._MODULE_REGISTRY` produce
behavior outputs. The registry is keyed by the `(module_type, module_id)` pair parsed from each feather filename.
The table below is the eligibility contract — which `(type, id)` pairs are processable and which
`MesoscopeHardwareState` fields gate each one. The per-module event codes, output column schemas, and unit
conversions are owned by `mesoscope:mesoscope-vr-module-parsing`.

| `(type, id)` | Kind          | Output filename                | Required `MesoscopeHardwareState` fields                  |
|--------------|---------------|--------------------------------|-----------------------------------------------------------|
| `(2, 1)`     | Encoder       | `encoder_data.feather`         | `cm_per_pulse`                                            |
| `(1, 1)`     | Mesoscope TTL | `mesoscope_frame_data.feather` | `recorded_mesoscope_ttl` (must be True)                   |
| `(3, 1)`     | Brake         | `brake_data.feather`           | `maximum_brake_strength`, `minimum_brake_strength`        |
| `(5, 1)`     | Water valve   | `valve_data.feather`           | `valve_scale_coefficient`, `valve_nonlinearity_exponent`  |
| `(5, 2)`     | Gas puff      | `gas_puff_data.feather`        | `delivered_gas_puffs` (must be True)                      |
| `(4, 1)`     | Lick sensor   | `lick_data.feather`            | `lick_threshold`                                          |
| `(6, 1)`     | Torque        | `torque_data.feather`          | `torque_per_adc_unit`                                     |
| `(7, 1)`     | Screen        | `screen_data.feather`          | `screens_initially_on`                                    |

Eligibility rule: a module feather produces a job only if its `(type, id)` pair is registered
**and** all listed hardware state fields are set to a non-None value (boolean fields must be
`True`). Ineligible modules are silently skipped — they do not produce an error, because a session
may have feather files for hardware that was not configured for that run.

The `MesoscopeHardwareState` YAML itself is authored and validated via `assets:session-hardware-state` in the
assets' plugin. This skill only documents how the behavior pipeline consults specific fields
for module eligibility.

### Jobs produced

One `(microcontroller_processing, "{controller_id}-{module_type}-{module_id}")` per eligible module.
Ineligible modules contribute zero jobs even if the feather file exists.

---

## Experiment configuration dependency

`MesoscopeExperimentConfiguration` YAML is required only for `MESOSCOPE_EXPERIMENT` sessions. It
defines the trial structures (`MesoscopeWaterRewardTrial`, `MesoscopeGasPuffTrial`) used to decompose
VR wall cue sequences into trials. Without it, the runtime processing job cannot emit `vr_cue_data.feather`,
`vr_trigger_zone_data.feather`, or `trial_data.feather` for experiment sessions. Authoring and
schema reference live in `assets:experiment-configuration`.

---

## Cross-library handoff contract

The behavior pipeline consumes raw acquisition outputs and performs the microcontroller log
extraction itself, in-process. The raw inputs it requires are:

| Raw input source                   | Owns the protocol / binding             | Raw artifact consumed                                                                | Drives behavior job          |
|------------------------------------|-----------------------------------------|--------------------------------------------------------------------------------------|------------------------------|
| Mesoscope-VR acquisition runtime   | `mesoscope:mesoscope-vr-runtime`        | `{raw_data}/behavior_data/1_log.npz`                                                 | `runtime_processing`         |
| `ataraxis-video-system`            | `ataraxis@video:log-processing`         | `{raw_data}/camera_data/{source_id}_log.npz` + camera manifest                       | (camera extraction stage)    |
| `ataraxis-communication-interface` | `ataraxis@communication:log-processing` | `{raw_data}/behavior_data/{controller_id}_log.npz` + `extraction_configuration.yaml` | `microcontroller_processing` |

The acquisition side of the first row spans two owners. `mesoscope:mesoscope-vr-runtime` owns the runtime message
code enum written into the archive, and `experiment:data-management` owns `assemble_session_logs`, the preprocessing
primitive that archives the raw log directory into the `1_log.npz` this pipeline reads.

The microcontroller module feathers under `{processed_data}/microcontroller_data/` are not an
external handoff: the behavior pipeline's in-process axci extraction stage produces them from the raw
archives plus the extraction config (see Microcontroller module feather files above). The axci
binding and protocol internals are owned by `ataraxis@communication:log-processing`, but there is no
required separate pre-run of that pipeline.

The camera timestamps enter behavior processing through the separate cross-system camera-timestamp
extraction stage (see `forging:camera-timestamp-extraction`), which re-extracts the raw `camera_data/`
logs into `{name}_timestamps.feather` files under `processed_data/video_data/`. This is not a behavior-pipeline
job: `_discover_jobs` only produces `runtime_processing` and `microcontroller_processing` jobs.

**What gating actually applies:** If you run the behavior pipeline against a session whose
`raw_data/behavior_data/` lacks both the runtime `1_log.npz` archive and any eligible microcontroller
module feather (after the in-process extraction stage), `prepare_behavior_processing_batch_tool`
reports `"No processable behavior jobs discovered for this session."`. The in-process extraction
stage additionally raises `FileNotFoundError` if `extraction_configuration.yaml` is absent from
`raw_data/behavior_data/`.

---

## Prerequisites checklist

Before handing a session off to `forging:behavior-processing`:

```text
Behavior Input Prerequisites:
- [ ] Session is eligible per PROCESSABLE_SESSION_TYPES (see Session eligibility above)
- [ ] 1_log.npz runtime archive present in raw_data/behavior_data/ (if a runtime job is expected)
- [ ] Raw microcontroller log archives + extraction_configuration.yaml present in raw_data/behavior_data/
- [ ] Hardware state YAML valid per assets:session-hardware-state
- [ ] Hardware state fields populated for every module you expect to process
- [ ] Experiment configuration YAML valid per assets:experiment-configuration (MESOSCOPE_EXPERIMENT only)
- [ ] Camera timestamp extraction completed — {name}_timestamps.feather files present (see forging:camera-timestamp-extraction)
```

---

## Related skills

| Skill                                   | Relationship                                                                    |
|-----------------------------------------|---------------------------------------------------------------------------------|
| `forging:forging-mcp-environment-setup` | Prerequisite: MCP server connectivity                                           |
| `forging:data-processing-design`        | Reference: the platform-general processing doctrine these inputs feed           |
| `forging:microcontroller-primitives`    | Owner: agnostic feather discovery, naming, and event-partition primitives       |
| `forging:camera-timestamp-extraction`   | Owner: manifest-driven camera timestamp re-extraction and renaming              |
| `mesoscope:mesoscope-vr-module-parsing` | Owner: per-module event codes, conversions, and output schemas                  |
| `mesoscope:mesoscope-vr-runtime`        | Upstream: owns the runtime message code enum written into the raw archive       |
| `experiment:data-management`            | Upstream: owns `assemble_session_logs`, which archives the raw log directory    |
| `assets:session-discovery`              | Upstream: session discovery and filtering                                       |
| `assets:session-data`                   | Reference: session marker and layout                                            |
| `assets:project-hierarchy`              | Reference: project / animal / session hierarchy                                 |
| `assets:session-hardware-state`         | Reference: MesoscopeHardwareState YAML                                          |
| `assets:session-descriptors`            | Reference: per-session descriptor YAML                                          |
| `assets:experiment-configuration`       | Reference: MesoscopeExperimentConfiguration YAML                                |
| `forging:behavior-processing`           | Downstream: consumes the inputs documented here                                 |
| `forging:behavior-results`              | Downstream: documents outputs derived from these inputs                         |
| `ataraxis@video:log-processing`         | Owner of the raw camera log archive format and acquisition protocol             |
| `ataraxis@communication:log-processing` | Owner of the axci log-processing binding the behavior pipeline calls in-process |
