# Per-subsystem-type lifecycle surface

The bring-up sequence and method surface of a Layer-2b binding class are specific to each subsystem
type. This file documents the lifecycle surface of each major subsystem type on the platform. It is
loaded on demand from `acquisition-system-design`'s SKILL.md. The shared class template and the
lifecycle-method conventions live in `layer-patterns.md`.

---

## Shared contract (all subsystem types)

Every binding class, regardless of type:

- takes the most-shared dependency first in its constructor (`data_logger`, when the subsystem logs to
  it), then its per-subsystem configuration dataclass, then any optional inputs,
- instantiates per-device wrappers as **public** attributes and wraps them in **private** low-level
  controllers,
- exposes an idempotent bring-up and tear-down pair for the microcontroller and camera types, with
  `__del__` calling the tear-down as a safety net, while a third-party-SDK subsystem connects in
  `__init__` and relies on the orchestrator calling its `disconnect()` explicitly,
- carries a bring-up flag when it exposes an idempotent bring-up and tear-down pair, raising the flag
  before the first bring-up step and clearing it only after the last tear-down step, so a partial
  bring-up still tears down and a failed tear-down stays retryable. A subsystem that connects in
  `__init__` needs no flag, because it has no separate bring-up to guard,
- isolates every tear-down step through `run_shutdown_step` (`cross_system/shutdown_tools.py`),
- stays oblivious to other subsystems, because cross-subsystem coordination is the orchestrator's job.

What differs is the bring-up sequence and the method names, below.

---

## Microcontroller subsystems

Wrap N × `MicroControllerInterface`, each composing one or more `ModuleInterface` wrappers.

| Aspect      | Detail                                     |
|-------------|--------------------------------------------|
| Constructor | `(data_logger, <subsystem>_configuration)` |
| Bring-up    | `start()`                                  |
| Tear-down   | `stop()`                                   |

`start()` runs three steps in order:

1. **Start each `MicroControllerInterface`** (in a defined order), spawning its communication
   subprocess.
2. **Call `initialize_local_assets()`** on every wrapper backed by a `SharedMemoryArray`, connecting
   the parent process to the shared memory the wrapper allocated in `__init__`. Without this, the
   wrapper's live properties (lick count, delivered volume, etc.) return stale data.
3. **Push runtime parameters** to each module via `set_parameters()`, sending the runtime parameter
   struct that mirrors the firmware's `CustomRuntimeParameters`.

If any step fails, the wrapper's runtime state is unsafe and `start()` MUST raise rather than leave a
half-started subsystem. `stop()` tears down each controller (which also resets the wrapped hardware).

**Current instance:** the worked example's microcontroller binding class. See `mesoscope:mesoscope-vr`.
For the per-wrapper mechanics and the firmware-side contract, see `/microcontroller-interface`.

---

## Camera subsystems

Wrap N × `VideoSystem`.

| Aspect      | Detail                                                                  |
|-------------|-------------------------------------------------------------------------|
| Constructor | `(data_logger, <subsystem>_configuration, output_directory)`            |
| Bring-up    | `start_<role>_camera()` then `save_<role>_camera_frames()` (per camera) |
| Tear-down   | `stop()` (stops acquisition and saving for all cameras)                 |

Camera subsystems **split acquisition from saving** across separate methods:

- `start_<role>_camera()` begins frame **acquisition** while saving stays off. Acquisition is started
  early, so an operator can preview the live feed before the subject is in position.
- `save_<role>_camera_frames()` begins **writing** frames to disk. Saving is started later, when the
  runtime begins.

Each `VideoSystem` is fully configured at construction (camera index, display rate, encoder, pixel
format, quantization, preset), so its parameters are set once, at construction.

**Current instance:** the worked example's camera binding class. See `mesoscope:mesoscope-vr`. For the
`VideoSystem` API, encoding configuration, and acquisition patterns, see `video:camera-interface`
(ataraxis marketplace).

---

## Motor / third-party-SDK subsystems

Wrap N × third-party SDK connection (e.g., `ZaberConnection`).

| Aspect      | Detail                                                                                        |
|-------------|-----------------------------------------------------------------------------------------------|
| Constructor | Opens the SDK connection synchronously in `__init__`, with no deferred `start()`              |
| Bring-up    | `connect()` in `__init__`, plus position methods such as `restore_position` and `park_motors` |
| Tear-down   | `disconnect()`                                                                                |

These subsystems open the SDK connection inside `__init__` and expose `disconnect` plus position and
state methods for their lifecycle, because a third-party SDK manages its own session. A per-session
position snapshot typically captures their state, so the binding class may take no `data_logger`. For
the current worked example's snapshot files, see `mesoscope:mesoscope-vr-snapshots`.

**Current instance:** the worked example's motor binding class. See `mesoscope:mesoscope-vr`. For the
motor mechanics, meaning park and unpark safety, position management, and MCP discovery, see
`/zaber-interface`.

---

## Asynchronous asset-subsystem drivers

Some subsystems are driven by an **asynchronous typed-event** source. The Unity VR task driver is one such subsystem,
and a standard subsystem of every acquisition system. The orchestrator composes and drives these directly (they sit
outside the Layer-2b `start`/`stop` binding-class surface): they open an MQTT (or similar) connection via `connect` /
`disconnect` and surface per-cycle messages as typed events through a `cycle()` pump, which the runtime loop dispatches.
They follow the runtime skill's event pattern. See `/vr-driver-interface` for the driver surface and
`/acquisition-system-runtime` for how the orchestrator pumps and dispatches their events.

---

## External data-service processors

Some subsystems are not hardware at all. They read records from, and write results back to, an
**external request/response data service** such as a Google Sheet, a LIMS, or a REST registry. The
canonical examples are the `SurgeryLog` and `WaterLog` classes that interface with the platform's
surgery and water-restriction Google Sheets. These sit **outside both** the Layer-2b binding-class
surface and the orchestrator's runtime-event pump. Nothing composes them at construction, nothing
starts or stops them, and no registry backs them, so they carry no `_assert_registry_coverage()` entry.
Instead, **per-session setup or preprocessing code constructs them on demand** and calls them.

The lifecycle surface is request and response rather than start and stop:

- the constructor takes the **record identity** (project, animal, session), a `credentials_path`, and
  a `sheet_id` or equivalent endpoint. It authenticates, validates the source's schema, and caches the
  connection.
- `extract_*` methods parse records into a typed platform dataclass, which is the **read** direction.
  That dataclass is a **registered read asset** in slsa (`READ_ASSET_REGISTRY`), cached on disk to
  standardize the downstream interface. `update_*` methods write runtime-discovered values back to the
  external source, which is the **write** direction and produces no dataclass. A processor may do one
  direction or both.
- `close()` releases the connection and its underlying SSL socket, so the caller that owns the
  processor closes it in a `try/finally`. `__del__` is a backstop only
  (`SurgeryLog` and `WaterLog` in `cross_system/google_sheet_tools.py`). There is no shared `data_logger` and no
  orchestrator coordination.

Unlike hardware subsystems, these are **gated on configuration**. When their identifier is unset the
system skips them entirely, and a system that uses no external service needs no credentials. For the
processor API, the schema contract, the auth model, and the procedure for authoring a custom one, see
`/google-sheets-processing` and the "Authoring a custom data-service processor" workflow in
`workflows.md`.
