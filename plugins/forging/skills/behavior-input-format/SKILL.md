---
name: behavior-input-format
description: >-
  Documents the behavior-processing-specific input artifacts consumed by the sollertia-forgery pipeline:
  the Mesoscope-VR runtime NPZ archive, the ataraxis-video-system camera timestamp feather files, and
  the ataraxis-communication-interface microcontroller module feather files. Delegates session layout,
  hardware state, and experiment configuration authoring to the configuration plugin. Use when the user
  asks about upstream handoff, module eligibility, or why a behavior job is missing on disk.
user-invocable: true
---

# Behavior input format

Authoritative reference for the **behavior-processing-specific** input artifacts: the runtime NPZ
archive, the camera timestamp feathers (upstream `ataraxis-video-system`), and the microcontroller
module feathers (upstream `ataraxis-communication-interface`). This skill only covers artifacts that
are unique to the behavior subsystem. Session metadata, directory layout, hardware state, and
experiment configuration are owned by the configuration plugin and are referenced, not redocumented.

---

## Scope

**Covers:**
- Runtime NPZ archive format, source ID, message codes, and the runtime job contract
- Camera timestamp feather handoff from axvs and the `_CAMERA_OUTPUT_NAMES` registry
- Microcontroller module feather handoff from axci and the `_MODULE_REGISTRY` registry
- Module-to-hardware-state-field eligibility table
- Cross-library handoff contract (ordering between axvs / axci / forgery)

**Does not cover:**
- Session directory layout, `raw_data/` / `processed_data/` hierarchy (see `/configuration:session-data` and
  `/configuration:project-hierarchy`)
- Session marker (`session_data.yaml`), `SessionData`, and `SessionTypes` (see `/configuration:session-data`)
- Hardware state YAML (`MesoscopeHardwareState`) authoring and validation (see `/configuration:session-snapshots`)
- Experiment configuration YAML (`MesoscopeExperimentConfiguration`) authoring and validation
  (see `/configuration:experiment-configuration`)
- Session descriptor YAMLs (see `/configuration:session-descriptors`)
- Session discovery and filtering (see `/session-discovery`)
- Output formats, verification, and data querying (see `/behavior-results`)
- Batch orchestration workflow (see `/behavior-processing`)
- Upstream axvs / axci processing (see `/video:log-processing` and `/communication:log-processing`)

**Note:** `/video:*` and `/communication:*` refer to the **video** and **communication** plugins
from the [ataraxis marketplace](https://github.com/Sun-Lab-NBB/ataraxis).

---

## Session eligibility

A session is eligible for behavior processing only if its `session_type` is in
`PROCESSABLE_SESSION_TYPES`:

| Session type           | Eligible | Notes                                                     |
|------------------------|----------|-----------------------------------------------------------|
| `LICK_TRAINING`        | yes      | Runtime, camera, and microcontroller jobs                 |
| `RUN_TRAINING`         | yes      | Runtime, camera, and microcontroller jobs                 |
| `MESOSCOPE_EXPERIMENT` | yes      | Same as training plus experiment-specific runtime outputs |
| (anything else)        | no       | Returned with `eligible=False` and no error               |

Use `/session-discovery` with
`session_types=["lick_training", "run_training", "mesoscope_experiment"]` to discover eligible
sessions. Eligibility is re-validated at prepare time by `prepare_behavior_processing_batch_tool`,
which re-loads `SessionData`, checks the type against `PROCESSABLE_SESSION_TYPES`, requires a
`*hardware_state*.yaml` file under `raw_data/`, and rejects the session unless at least one
runtime, camera, or microcontroller job is found on disk.

Only `MESOSCOPE_EXPERIMENT` sessions produce the experiment-specific outputs
(`reinforcing_guidance_state_data.feather`, `aversive_guidance_state_data.feather`,
`vr_cue_data.feather`, `vr_trigger_zone_data.feather`, `trial_data.feather`). Training sessions
produce only the state feathers (`system_state_data.feather`, `runtime_state_data.feather`) plus
any eligible microcontroller and camera outputs.

---

## Where behavior inputs live

Behavior processing reads three distinct artifact classes from a session directory. The directory
hierarchy itself is documented in `/configuration:session-data` and `/configuration:project-hierarchy`; this skill only captures
the subset the behavior pipeline touches:

```text
{session_root}/
├── raw_data/
│   ├── session_data.yaml                          ← see /session-data (not consumed here)
│   ├── *hardware_state*.yaml                      ← see /session-snapshots (gates module jobs)
│   ├── *experiment_configuration*.yaml            ← see /experiment-configuration (experiments only)
│   └── <nested>/
│       └── 1_log.npz                              ← runtime NPZ archive (this skill)
└── processed_data/
    ├── camera_timestamps/
    │   └── camera_{source_id}_timestamps.feather  ← axvs handoff (this skill)
    └── microcontroller_data/
        └── controller_{cid}_module_{type}_{id}.feather  ← axci handoff (this skill)
```

The pipeline uses `rglob` for all three artifact classes, so files may be nested at any depth below
`raw_data/` / `processed_data/`.

---

## Runtime log archive

### Producer

The `MesoscopeVRSystem` class in `sl_experiment.mesoscope_vr.data_acquisition` owns the runtime
`DataLogger` for every Mesoscope-VR session. On startup it instantiates:

```python
DataLogger(
    output_directory=session_data.raw_data.raw_data_path,
    instance_name="behavior",
    thread_count=10,
)
```

Following the DataLogger convention, this creates a `behavior_data_log/` subdirectory under
`raw_data/`, and every message written to the logger's input queue is persisted as an individual
`.npy` file in that directory. Runtime messages use the reserved class-level constant
`_source_id: np.uint8 = np.uint8(1)` — this ID is reserved for the runtime itself and MUST NOT
collide with any camera or microcontroller source ID on the same DataLogger. Cameras and
microcontrollers managed by the same `MesoscopeVRSystem` share this DataLogger but use higher IDs
(face / body cameras use 51 / 62; Actor / Sensor / Encoder microcontrollers use the 101-150
advisory range).

After the session ends, the Mesoscope-VR preprocessing pipeline
(`sl_experiment.mesoscope_vr.data_preprocessing`) runs `assemble_log_archives()` from
`ataraxis-data-structures`, which groups per-source `.npy` files into one uncompressed `.npz`
archive per source ID. The runtime archive is the only archive with source ID `1`, so the forging
pipeline discovers it with a single `rglob("1_log.npz")` pass against the session's `raw_data/`.

### Required location

Recursively under `{session_root}/raw_data/`. The typical on-disk layout places the archive at
`{raw_data_path}/behavior_data_log/1_log.npz`. The forging pipeline searches for `1_log.npz` and
raises `ValueError` if more than one match is found — the runtime DataLogger is guaranteed to
produce exactly one archive per session, so duplicate hits indicate a corrupted directory.

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
- `payload` is a `uint8` view of byte 9 onward of the entry — in other words, the original
  `serialized_data` array handed to `LogPackage`, bit-for-bit.

The forging runtime processor (`sollertia_forgery.processing.runtime.process_runtime_data`) only
ever talks to `LogArchiveReader` — it never opens the NPZ directly and never parses the 9-byte
header. Every nuance in the Message payloads and Cue sequence messages subsections below describes
the shape of `payload` only, not the header.

#### Onset message

The first message written to the runtime log has `acquisition_time=0` and carries the session's
absolute UTC timestamp as a serialized byte array produced by
`get_timestamp(output_format=TimestampFormats.BYTES)`. `LogArchiveReader` detects this message
during initialization and uses it to convert every subsequent message's relative `elapsed_us` into
an absolute UTC timestamp exposed as `message.timestamp_us`. Without the onset the reader cannot
resolve absolute time, so archives with a missing or corrupt onset are unreadable.

#### Message payloads

After the onset, every runtime message is a single `np.uint8` array written as
`LogPackage.serialized_data`. The reader returns this array verbatim as `message.payload`.
Payloads have **two disjoint shapes**:

1. **Code-prefixed state messages** — short arrays (≤ 9 bytes) whose first byte is a message code
   in the range 1-5.
2. **Cue sequence messages** — long arrays (> 500 bytes) with no leading code byte.

The forging runtime reader dispatches on payload length first (cue sequence detection), then on
`payload[0]` for the state-message codes.

The runtime message code enum lives in
`sl_experiment.mesoscope_vr.data_acquisition._MesoscopeVRLogMessageCodes`; the forging runtime
reader mirrors every code as a module-level constant (`_SYSTEM_STATE_CODE`, `_RUNTIME_STATE_CODE`,
`_REINFORCING_GUIDANCE_STATE_CODE`, `_AVERSIVE_GUIDANCE_STATE_CODE`, `_DISTANCE_SNAPSHOT_CODE`) in
`sollertia_forgery.processing.runtime`. These two definitions MUST stay in lock-step — any new
writer code requires a matching reader constant, or forgery will silently drop unrecognized
messages.

**Code-prefixed payload layouts:**

| Code | Name (sl-experiment enum)    | Payload layout                                         | Bytes |
|------|------------------------------|--------------------------------------------------------|-------|
| 1    | `SYSTEM_STATE`               | `[1: uint8][state_code: uint8]`                        | 2     |
| 2    | `RUNTIME_STATE`              | `[2: uint8][state_code: uint8]`                        | 2     |
| 3    | `REINFORCING_GUIDANCE_STATE` | `[3: uint8][enabled: uint8 (0 or 1)]`                  | 2     |
| 4    | `AVERSIVE_GUIDANCE_STATE`    | `[4: uint8][enabled: uint8 (0 or 1)]`                  | 2     |
| 5    | `DISTANCE_SNAPSHOT`          | `[5: uint8][traveled_distance: 8 bytes little-endian]` | 9     |

**SYSTEM_STATE / RUNTIME_STATE:** The trailing `state_code` byte is the numeric code of the
destination state. State enums live in sl-experiment and are not interpreted by forgery — codes
are stored verbatim in `system_state_data.feather` / `runtime_state_data.feather` for downstream
consumers to resolve.

**REINFORCING_GUIDANCE_STATE / AVERSIVE_GUIDANCE_STATE:** Emitted only during experiment sessions
when the corresponding Unity-side guidance mode toggles. The trailing byte is a boolean
(`0` = disabled, `1` = enabled). Training sessions never emit these codes, and forgery also gates
on the `experiment_configuration` before writing the guidance feathers, so non-experiment sessions
skip them even if the archive somehow contained them.

**DISTANCE_SNAPSHOT:** Emitted only when the runtime receives a `UNITY_TERMINATION` MQTT message
(i.e. an emergency pause triggered by a Unity crash) — the runtime snapshots the wheel encoder's
total traveled distance just before pausing, so the downstream cue-sequence decomposer can stitch
the pre-termination distance into the next sequence's cumulative offset. Sessions that complete
without a Unity crash never emit this code. The 8-byte tail is the traveled distance in
centimeters as a little-endian float64; the forging reader parses it via
`payload[1:9].view(dtype="<f8")[0]`.

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

The `ataraxis-video-system` (axvs) log-processing pipeline. Each camera's frame timestamps are
extracted from the camera's log archive via `/video:log-processing` and written to an uncompressed
Arrow IPC (`.feather`) file. The behavior pipeline consumes those files directly and hardlinks
them into `{session.processed_data_path}/behavior_data/` under a legacy per-camera name — it does
NOT re-decode the source NPZ archives.

### Required location

Recursively under `{session_root}/processed_data/`. Search glob:
`camera_*_timestamps.feather`. The pipeline parses the numeric source ID from each filename.

### Naming convention

```text
camera_{source_id}_timestamps.feather
```

The pipeline recognizes these source IDs:

| Source ID | Registered output filename       | Role        |
|-----------|----------------------------------|-------------|
| 51        | `face_camera_timestamps.feather` | Face camera |
| 62        | `body_camera_timestamps.feather` | Body camera |

A camera feather with any other source ID is discovered but raises `ValueError` at processing time
because no legacy output name is registered. Adding support for additional cameras requires
updating `_CAMERA_OUTPUT_NAMES` in `sollertia_forgery.processing.camera`.

### Schema

The axvs camera feathers already contain the final `frame_time_us` column in microsecond UTC form.
The behavior pipeline does not inspect the schema — it uses `os.link()` to hardlink the file into
`behavior_data/` under the legacy output name, leaving the source file untouched so axvs discovery
still finds it.

### Job produced

One `(camera_processing, "{source_id}")` per camera feather found.

---

## Microcontroller module feather files

### Producer

The `ataraxis-communication-interface` (axci) log-processing pipeline. axci extracts per-module
event data from controller log archives and writes one feather file per
`(controller_id, module_type, module_id)` tuple.

### Required location

Recursively under `{session_root}/processed_data/`. Search glob:
`controller_*_module_*.feather`. The pipeline parses three integers from each filename.

### Naming convention

```text
controller_{controller_id}_module_{module_type}_{module_id}.feather
```

The pipeline splits on `_` and expects exactly five parts. Malformed filenames raise `ValueError`.

### Schema

The axci log-processing pipeline writes a standard 5-column schema for every module feather:

| Column         | Dtype    | Description                                                        |
|----------------|----------|--------------------------------------------------------------------|
| `timestamp_us` | `UInt64` | Message timestamp in microseconds since UTC epoch                  |
| `command`      | `UInt8`  | Command code the module was executing when the message was sent    |
| `event`        | `UInt8`  | Event code identifying the message type                            |
| `dtype`        | `String` | NumPy dtype string for the data payload (null if no data)          |
| `data`         | `Binary` | Serialized binary payload (null if no data)                        |

Behavior processing reads these files via memory-mapped Polars IPC
(`pl.read_ipc(memory_map=True)`) and partitions rows by `event` in a single pass. axci writes
uncompressed IPC, which is what makes memory-mapped reads safe.

### Module registry

Only modules registered in `sollertia_forgery.processing.microcontrollers._MODULE_REGISTRY` produce
behavior outputs. The current registry:

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

The `MesoscopeHardwareState` YAML itself is authored and validated via `/configuration:session-snapshots` in the
configuration plugin. This skill only documents how the behavior pipeline consults specific fields
for module eligibility.

### Jobs produced

One `(microcontroller_processing, "{controller_id}-{module_type}-{module_id}")` per eligible module.
Ineligible modules contribute zero jobs even if the feather file exists.

---

## Experiment configuration dependency

`MesoscopeExperimentConfiguration` YAML is required only for `MESOSCOPE_EXPERIMENT` sessions. It
defines the trial structures (`WaterRewardTrial`, `GasPuffTrial`) used to decompose VR wall cue
sequences into trials. Without it, the runtime processing job cannot emit `vr_cue_data.feather`,
`vr_trigger_zone_data.feather`, or `trial_data.feather` for experiment sessions. Authoring and
schema reference live in `/configuration:experiment-configuration`.

---

## Cross-library handoff contract

The behavior pipeline is a pure consumer of upstream outputs:

| Upstream producer                  | Required upstream skill         | Artifact                                                              | Gates behavior job           |
|------------------------------------|---------------------------------|-----------------------------------------------------------------------|------------------------------|
| Mesoscope-VR acquisition runtime   | (acquisition-side; no skill)    | `{raw_data}/<nested>/1_log.npz`                                       | `runtime_processing`         |
| `ataraxis-video-system`            | `/video:log-processing`         | `{processed_data}/camera_timestamps/camera_*_timestamps.feather`      | `camera_processing`          |
| `ataraxis-communication-interface` | `/communication:log-processing` | `{processed_data}/microcontroller_data/controller_*_module_*.feather` | `microcontroller_processing` |

**Ordering constraint:** `/video:log-processing` and `/communication:log-processing` MUST complete successfully BEFORE
`/behavior-processing` can discover the session. If you run the behavior pipeline against a session
that has not been through the upstream pipelines, `prepare_behavior_processing_batch_tool` will
return `"No processable behavior jobs discovered for this session."` even though the raw NPZ
archives exist.

---

## Prerequisites checklist

Before handing a session off to `/behavior-processing`:

```text
Behavior Input Prerequisites:
- [ ] Session is eligible per PROCESSABLE_SESSION_TYPES (see Session eligibility above)
- [ ] 1_log.npz runtime archive present in raw_data/ (if a runtime job is expected)
- [ ] Hardware state YAML valid per /session-snapshots
- [ ] Hardware state fields populated for every module you expect to process
- [ ] Experiment configuration YAML valid per /experiment-configuration (MESOSCOPE_EXPERIMENT only)
- [ ] /video:log-processing completed — camera_*_timestamps.feather files present
- [ ] /communication:log-processing completed — controller_*_module_*.feather files present
```

---

## Related skills

| Skill                                     | Relationship                                                 |
|-------------------------------------------|--------------------------------------------------------------|
| `/forging-mcp-environment-setup`          | Prerequisite: MCP server connectivity                        |
| `/session-discovery`                      | Upstream: session discovery and filtering                    |
| `/configuration:session-data`             | Reference: session marker and layout                         |
| `/configuration:project-hierarchy`        | Reference: project / animal / session hierarchy              |
| `/configuration:session-snapshots`        | Reference: MesoscopeHardwareState YAML                       |
| `/configuration:session-descriptors`      | Reference: per-session descriptor YAML                       |
| `/configuration:experiment-configuration` | Reference: MesoscopeExperimentConfiguration YAML             |
| `/behavior-processing`                    | Downstream: consumes the inputs documented here              |
| `/behavior-results`                       | Downstream: documents outputs derived from these inputs      |
| `/video:log-processing`                   | Upstream producer of camera timestamp feathers               |
| `/communication:log-processing`           | Upstream producer of microcontroller module feathers         |
