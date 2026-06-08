# Per-subsystem-type lifecycle surface

The bring-up sequence and method surface of a Layer-2b binding class are specific to each subsystem
type. This file documents the lifecycle surface for each major
subsystem type currently on the platform. Loaded on demand from `acquisition-system-design`'s
SKILL.md; the shared class template and lifecycle-method conventions live in
[layer-patterns.md](layer-patterns.md#layer-2b-per-subsystem-binding-classes).

---

## Shared contract (all subsystem types)

Every binding class, regardless of type:

- takes the most-shared dependency first in its constructor (`data_logger`, when the subsystem logs to
  it), then its per-subsystem configuration dataclass, then any optional inputs;
- instantiates per-device wrappers as **public** attributes and wraps them in **private** low-level
  controllers;
- exposes an **idempotent** bring-up / tear-down pair, with `__del__` calling the tear-down as a
  safety net;
- stays oblivious to other subsystems — cross-subsystem coordination is the orchestrator's job.

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

**Current instance:** Mesoscope-VR's `MicroControllerInterfaces` (ACTOR / SENSOR / ENCODER boards).
For the per-wrapper mechanics and the firmware-side contract, see `experiment:microcontroller-interface`.

---

## Camera subsystems

Wrap N × `VideoSystem`.

| Aspect      | Detail                                                                  |
|-------------|-------------------------------------------------------------------------|
| Constructor | `(data_logger, <subsystem>_configuration, output_directory)`            |
| Bring-up    | `start_<role>_camera()` then `save_<role>_camera_frames()` (per camera) |
| Tear-down   | `stop()` (stops acquisition and saving for all cameras)                 |

Camera subsystems **split acquisition from saving** across separate methods:

- `start_<role>_camera()` begins frame **acquisition** (saving stays off). Acquisition is started
  early — for live preview / monitoring before the subject is in position.
- `save_<role>_camera_frames()` begins **writing** frames to disk. Saving is started later, when the
  runtime begins.

Each `VideoSystem` is fully configured at construction (camera index, display rate, encoder, pixel
format, quantization, preset), so its parameters are set once, at construction.

**Current instance:** Mesoscope-VR's `VideoSystems` (face and body cameras). For the `VideoSystem`
API, encoding configuration, and acquisition patterns, see `ataraxis@video:camera-interface`.

---

## Motor / third-party-SDK subsystems

Wrap N × third-party SDK connection (e.g., `ZaberConnection`).

| Aspect      | Detail                                                                                                       |
|-------------|--------------------------------------------------------------------------------------------------------------|
| Constructor | Opens the SDK connection synchronously in `__init__` (no deferred `start()`)                                 |
| Bring-up    | `connect()` (in `__init__`) — and position setup methods (`restore_position`, `park_motors`/`unpark_motors`) |
| Tear-down   | `disconnect()`                                                                                               |

These subsystems expose `connect` / `disconnect` plus position/state methods for their lifecycle,
because a third-party SDK manages its own session. Their state is typically captured as per-session
position snapshots (read/written by `experiment:mesoscope-vr-snapshots`), so the binding class may take no
`data_logger`.

**Current instance:** Mesoscope-VR's `ZaberMotors` (HeadBar / Wheel / LickPort groups), which connects
to all ports in `__init__` and restores positions from the previous session's snapshot. For motor
mechanics (park/unpark safety, position management, MCP discovery), see `experiment:zaber-interface`.

---

## Asynchronous asset-subsystem drivers

Some subsystems are driven by an **asynchronous typed-event** source — the canonical example is the
Unity VR task driver. The orchestrator composes and drives these directly (they sit outside the
Layer-2b `start`/`stop` binding-class surface): they open an MQTT (or similar) connection via
`connect` / `disconnect` and surface per-cycle messages as typed events through a `cycle()` pump, which
the runtime loop dispatches. They follow the runtime skill's event pattern. See
`experiment:vr-driver-interface` for the driver surface and `experiment:acquisition-system-runtime`
for how the orchestrator pumps and dispatches their events.

---

## External data-service processors

Some subsystems are not hardware at all — they read records from, and write results back to, an
**external request/response data service** (a Google Sheet, a LIMS, a REST registry). The canonical
examples are the `SurgeryLog` and `WaterLog` classes that interface with the platform's surgery and
water-restriction Google Sheets. These sit **outside both** the Layer-2b binding-class surface and the
orchestrator's runtime-event pump: they are not composed at construction, not started/stopped, and not
registry-backed (no `_assert_registry_coverage()` entry). Instead, **per-session setup or
preprocessing code constructs them on demand**, calls them, and lets them be garbage-collected.

The lifecycle surface is request/response, not start/stop:

- the constructor takes the **record identity** (project / animal / session), a `credentials_path`,
  and a `sheet_id` (or equivalent endpoint), authenticates, validates the source's schema, and caches
  the connection;
- `extract_*` methods parse records into a typed platform dataclass (the **read** direction) — that
  dataclass is a **registered read asset** in slsa (`READ_ASSET_REGISTRY`), cached on disk to
  standardize the downstream interface; `update_*` methods write runtime-discovered values back to the
  external source (the **write** direction), producing no dataclass; a processor may do one or both;
- `__del__` closes the connection — there is no shared `data_logger` and no orchestrator coordination.

Unlike hardware subsystems, these are **gated on configuration**: when their identifier is unset the
system skips them entirely (a system that uses no external service needs no credentials). For the
processor API, the schema contract, the auth model, and the procedure for authoring a custom one, see
`experiment:google-sheets-processing` and the "Authoring a custom data-service processor" workflow in
[workflows.md](workflows.md).
