---
name: acquisition-system-design
description: >-
  Documents the platform-general design pattern for a Sollertia data acquisition system:
  the YAML system configuration, the per-subsystem binding classes, and the runtime
  orchestrator that owns master start/stop. Use when designing a new acquisition system,
  adding a hardware subsystem, or auditing an existing system's configuration/binding layer
  for pattern compliance.
user-invocable: false
---

# Acquisition system design

Documents the platform-general design pattern for a Sollertia data acquisition system at the
configuration and binding-class layer. An acquisition system is a host-PC stack centered on one
**main PC** that manages most hardware and data processing, optionally coordinating one or more
**additional PCs** each dedicated to a specific instrument. For example, for Mesoscope-VR, a second PC dedicated
to microscope control. The main PC composes the lower-level hardware interfaces (microcontrollers,
cameras, motors, etc.) into a runnable platform that produces session data; additional PCs run their
own acquisition software and are coordinated through filesystem paths and network settings in the
system configuration rather than through binding classes.

This skill is a **pattern skill** — it documents the conventions and contracts that all Sollertia
acquisition systems share, but does not document any single system's specific composition. For
concrete instances, see the per-system skills (currently `mesoscope:mesoscope-vr`).

Detailed authoring patterns live in three reference files, loaded on demand:

- [references/layer-patterns.md](references/layer-patterns.md) — full per-layer class patterns, field
  conventions, lifecycle rules, and the cross-layer contracts.
- [references/subsystem-types.md](references/subsystem-types.md) — the per-type lifecycle surface for
  each major subsystem type (microcontroller, camera, motor/SDK) currently on the platform.
- [references/workflows.md](references/workflows.md) — step-by-step procedures for adding a subsystem,
  building a new system, and extending an existing subsystem.

---

## Scope

**Covers:**
- Three-layer architecture (System Configuration YAML → Configuration dataclasses + Binding classes →
  Lifecycle orchestrator)
- Top-level system configuration pattern (`@dataclass` + `YamlConfig`, composition, validation, YAML roundtrip)
- Per-subsystem configuration dataclass pattern (naming, field conventions, units in field names)
- Per-subsystem binding class pattern (constructor signature, lifecycle methods, idempotency, `__del__` semantics)
- Shared cross-system configuration-file lifecycle (the `SystemConfiguration` registry and the
  create / resolve / load helpers) plus each system's registration and typed `get_system_configuration` accessor
- Cross-layer contracts (field naming consistency, schema versioning)
- Lifecycle ordering rules (DataLogger first, binding classes second, reverse on shutdown)
- Workflows for adding a new hardware subsystem, building a new acquisition system, extending an existing subsystem

**Does not cover** (delegated):
- The per-firmware-module Python wrapper layer (`cross_system/module_interfaces.py`) and slmc
  firmware Modules — see `/microcontroller-interface`.
- Concrete Mesoscope-VR composition (the binding-class instances and their YAML field surface) — see
  `mesoscope:mesoscope-vr`.
- The platform-general runtime-behavior pattern (state machine, runtime loop, event dispatch) — see
  `/acquisition-system-runtime`.
- Concrete Mesoscope-VR runtime behavior (state machine, training modes, CLI) — see
  `mesoscope:mesoscope-vr-runtime`.
- The Unity VR task driver subsystem — see `/vr-driver-interface`.
- Low-level VideoSystem mechanics — see `ataraxis@video:camera-interface`.
- Low-level MicroControllerInterface mechanics — see `ataraxis@communication:microcontroller-interface`.
- Zaber motor interface mechanics — see `/zaber-interface`.
- Per-session metadata, task templates, and experiment configuration — owned by the assets plugin
  (e.g. `assets:session-descriptors`, `assets:task-templates`, `assets:experiment-configuration`).
- The `sollertia-shared-assets` enum/registry side of registering a new acquisition system (the
  `AcquisitionSystems` member, the dispatch registries, the per-system descriptor / hardware-state /
  experiment-config / raw-data dataclasses, and the `from_task_template` experiment-configuration
  builder) — owned by `assets:library-extension`.
- The implementation of external data-service processors (the Google Sheets `SurgeryLog` / `WaterLog`
  classes, their schema contract, and authoring a custom one) — owned by
  `/google-sheets-processing`. This skill documents only where such processors sit in the
  architecture (see [Auxiliary sections](#auxiliary-sections-beyond-hardware-subsystems) and the
  "External data-service processors" category in
  [references/subsystem-types.md](references/subsystem-types.md)).

---

## Three-layer architecture

A Sollertia acquisition system is composed of three layers, top-down:

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Layer 1: System Configuration YAML                                             │
│  ─────────────────────────────────                                              │
│  <System>SystemConfiguration ── persisted as <system>_system_configuration.yaml │
│      ├── <System>FileSystem    ── filesystem paths (host-machine state)         │
│      ├── <System>Cameras       ── per-camera configuration                      │
│      ├── <System>MicroControllers ── per-microcontroller-module configuration   │
│      ├── <System>ExternalAssets ── third-party hardware (motors, MQTT, etc.)    │
│      └── auxiliary sections     ── e.g., external service IDs                   │
└────────────────────────────┬────────────────────────────────────────────────────┘
                             │ instantiates and parameterizes
┌────────────────────────────▼────────────────────────────────────────────────────┐
│  Layer 2: Per-hardware-subsystem binding classes                                │
│  ─────────────────────────────────────────                                      │
│  VideoSystems(data_logger, camera_configuration, output_directory)              │
│      └── wraps N × VideoSystem instances + per-camera lifecycle                 │
│  MicroControllerInterfaces(data_logger, microcontroller_configuration)          │
│      ├── instantiates N × ModuleInterface subclasses from configuration fields  │
│      └── wraps N × MicroControllerInterface instances + per-controller lifecycle│
│  <Subsystem>Bindings(<subsystem>_configuration, ...)                            │
│      └── one per hardware subsystem (zaber motors, custom devices, etc.)        │
└────────────────────────────┬────────────────────────────────────────────────────┘
                             │ composed and lifecycle-orchestrated by
┌────────────────────────────▼────────────────────────────────────────────────────┐
│  Layer 3: Lifecycle orchestrator (one per acquisition system)                   │
│  ─────────────────────────────────────────────────────────                      │
│  <System>System / <System>Runtime / system_controller module                    │
│      ├── owns DataLogger and the VR task driver                                 │
│      ├── constructs each Layer-2 binding class in the correct order             │
│      ├── coordinates start / stop / state transitions                           │
│      └── exposes the public runtime surface to CLI / GUI consumers              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

Each layer's conventions are summarized below and documented in full in
[references/layer-patterns.md](references/layer-patterns.md). The pattern is the same for every
Sollertia acquisition system; what differs between systems is the *specific* set of hardware subsystems,
the *specific* configuration fields per subsystem, and the *specific* runtime states.

---

## Layer 1: System configuration

The top-level system configuration is a single plain `@dataclass` (NOT `slots=True`) that inherits
from `ataraxis_data_structures.YamlConfig` and composes one nested dataclass per concern via
`field(default_factory=...)` — one per hardware subsystem, plus auxiliary host-state concerns like
filesystem paths and external service IDs. It subclasses `SystemConfiguration`, carries a free-form
`name` label, and optionally overrides `__post_init__` (to normalize YAML-shaped fields and validate
user input) and `save()` (for YAML-roundtrip translation). Its configuration-file lifecycle (create /
resolve / load) is handled by the shared cross-system registry; the system's module registers its
`SystemConfiguration` subclass and exposes a typed `get_system_configuration()` accessor.

For the full class-definition pattern, the `__post_init__` / `save()` overrides, and the shared
configuration-file lifecycle, see
[references/layer-patterns.md](references/layer-patterns.md#layer-1-system-configuration).

---

## Layer 2a: Per-subsystem configuration dataclasses

Each hardware subsystem embeds a `@dataclass(slots=True)` named `<System><Subsystem>` that captures the
**parameters** the binding class uses to instantiate per-device wrappers. Every field has
four properties:

- an explicit, narrow type;
- a sensible default — a reference-rig value (a measured calibration value where the parameter is a
  calibration, otherwise a working factory setting), or `Path()` for filesystem fields the user
  MUST set;
- a triple-quoted docstring;
- a name following the `<device-or-module>_<parameter>_<unit>` convention (the unit suffix is
  included whenever the unit is non-obvious).

**GenTL/GenICam camera subsystems** follow an additional standard practice: each camera declares an
optional configuration-file path field (`<role>_camera_configuration_path: Path = Path()`, empty =
unset) pointing to a GenICam configuration YAML — an `ataraxis-video-system` `GenicamConfiguration`
file — that records the camera's expected node configuration. The field is declarative: it records
*where* the expected configuration lives so agents can verify the live camera against it, and dump or
restore it on request; the acquisition runtime does not auto-apply it. By convention the YAMLs live
in the working-directory `configuration/` folder next to the system configuration. The GenICam
dump/restore mechanics are owned by `ataraxis@video:camera-setup`; a system's own MCP server may add
a verify tool that diffs the live configuration against the stored file (Mesoscope-VR's
`verify_camera_configuration_tool` is the worked example).

For the field-naming table, the type conventions, and the defaults rules, see
[references/layer-patterns.md](references/layer-patterns.md#layer-2a-per-subsystem-configuration-dataclasses).

---

## Layer 2b: Per-subsystem binding classes

Each hardware subsystem has one binding class that composes the subsystem's per-device wrappers and
orchestrates their lifecycle. The **shared contract** is the same across subsystem types: the
constructor takes the most-shared dependency first (`data_logger`, when the subsystem logs to it),
then the per-subsystem configuration dataclass, then any optional inputs; it caches the configuration,
instantiates per-device wrappers as public attributes, and wraps them in private low-level
controllers. Bring-up and tear-down are idempotent, and `__del__` calls the tear-down as a safety net.

The **bring-up sequence and method surface are specific to each subsystem type:**

- **Microcontroller subsystems** expose `start()` / `stop()`; `start()` brings the controllers online,
  attaches host-side shared-memory assets (`initialize_local_assets()`), then pushes runtime
  parameters (`set_parameters()`).
- **Camera subsystems** expose a `start_<role>_camera()` → `save_<role>_camera_frames()` split
  (acquisition is separate from saving) with a unified `stop()`; each camera's parameters are set at
  construction.
- **Third-party-SDK subsystems** (e.g., the Zaber motor subsystem) connect in `__init__` and expose
  `connect` / `disconnect` plus position methods for their lifecycle; they may omit the `data_logger`.

For the shared class template and the lifecycle-method conventions, see
[references/layer-patterns.md](references/layer-patterns.md#layer-2b-per-subsystem-binding-classes).
For the per-type bring-up sequences, see [references/subsystem-types.md](references/subsystem-types.md).

---

## Layer 3: Lifecycle orchestrator

One class per acquisition system (typically `<System>System` or `<System>VRSystem`, in a
`system_controller` module) composes the Layer-2 binding classes and owns the master start/stop. It
instantiates the DataLogger first (so each `MicroControllerInterface.__init__` can register a
manifest entry), constructs the binding classes in a fixed order, starts the DataLogger before any
binding class, and tears everything down in reverse so the DataLogger outlives every consumer. It also
owns all cross-subsystem synchronization — individual binding classes stay oblivious to one another.
The VR task driver is a standard subsystem of every acquisition system, but the orchestrator constructs it
only for the session types that run the corridor task (experiment sessions); for training and
window-checking sessions the driver is not constructed and `self._vr_task` is `None` (see
`/acquisition-system-runtime` and `/vr-driver-interface`).
(For microcontroller keepalive, the orchestrator passes each `MicroControllerInterface` a
`keepalive_interval` at construction; AXCI sends the keepalive messages and raises on timeout.)

For the construction/shutdown order diagrams, keepalive handling, and cross-subsystem signaling rules,
see [references/layer-patterns.md](references/layer-patterns.md#layer-3-lifecycle-orchestrator).

---

## Cross-layer contracts

Three contracts must hold for the architecture to function:

1. **Configuration field naming agreement** — each configuration dataclass field that feeds a wrapper
   constructor matches the wrapper's keyword-argument name conceptually (unit suffixes may be dropped
   at the API boundary).
2. **Schema versioning** — any add / remove / rename / type-change of a configuration field is a schema
   change and MUST be paired with a version bump on the owning package; older YAML must fail loudly
   rather than silently misconfigure.
3. **Lifecycle ordering** — the DataLogger is running before any binding class `__init__`, `start()`
   precedes any wrapper command, and `stop()` precedes DataLogger shutdown.

For the field→kwarg table and the full per-contract rules, see
[references/layer-patterns.md](references/layer-patterns.md#cross-layer-contracts).

---

## Workflows

Three common changes each have a step-by-step procedure in
[references/workflows.md](references/workflows.md):

- **Adding a new hardware subsystem to an existing system** — author the configuration dataclass and binding
  class, wire them into the orchestrator, and regenerate the configuration YAML.
- **Building a new acquisition system from scratch** — hand the `sollertia-shared-assets` registry
  work to `assets:library-extension`, then author the dataclasses, helpers, binding classes,
  orchestrator, and CLI group.
- **Extending an existing subsystem** — add fields to an existing configuration dataclass and its binding
  class (the most common change).
- **Authoring a custom data-service processor** — reuse or author a `SurgeryLog` / `WaterLog`-style
  processor for an external request/response service (a Google Sheet, a LIMS). See also
  `/google-sheets-processing`.

---

## Auxiliary sections beyond hardware subsystems

Beyond the per-subsystem sections, the system configuration also captures host- and system-level
state that is not a hardware subsystem and has no binding class: filesystem paths, external-service
identifiers, and network endpoints consumed directly by orchestrator code or by per-session setup
steps. Exactly which of these a system needs is system-specific; the current platform (Mesoscope-VR)
uses three:

| Auxiliary section | Holds                                                       | Concrete fields in use                                       |
|-------------------|-------------------------------------------------------------|--------------------------------------------------------------|
| Filesystem paths  | Local mount points for acquired data and long-term storage  | `mesoscope_directory`, `storage_directories` (NAS / Server)  |
| External services | Identifiers for external services the runtime reads/writes  | Google Sheets IDs (`surgery_sheet_id`, `water_log_sheet_id`) |
| Network settings  | Broker endpoint for cross-process / cross-machine messaging | MQTT broker IP + port (`vr_task.ip` / `vr_task.port`)        |

These follow the same dataclass pattern as hardware subsystems but have no binding class — they're
consumed directly by orchestrator code or by per-session setup steps. The **Filesystem paths** and
**Network settings** rows are also where the main PC coordinates any additional PCs: an instrument PC
running its own acquisition stack (e.g., Mesoscope-VR's microscope-control PC, whose output lands in
`mesoscope_directory`) is reached through the mounted paths and network endpoints declared here, never
through a binding class.

The **External services** row holds only the *identifiers*; the code that actually reads records from,
and writes results back to, those services is a **data-service processor** — a separate category that
also has no binding class and is constructed by per-session setup or preprocessing code. For its
lifecycle surface see the "External data-service processors" category in
[references/subsystem-types.md](references/subsystem-types.md), and for the Google Sheets processors
(`SurgeryLog` / `WaterLog`), their schema contract, and authoring a custom one, see
`/google-sheets-processing`.

**Filesystem fields rule:** Every filesystem field SHOULD be checked at configuration load time via
a `check_system_mounts_tool` (or equivalent) that verifies the path exists and is writable. The
check is system-specific; the pattern is that the system configuration's MCP tooling exposes a
mount-check entry point.

---

## Mesoscope-VR as a worked example

The Mesoscope-VR acquisition system is the current consumer of every pattern in this skill:

- **System Configuration**: `MesoscopeSystemConfiguration` in
  `sollertia_experiment/mesoscope_vr/system.py`. Composes 6 sections (filesystem, sheets,
  cameras, microcontrollers, acquisition, assets) plus a top-level `name` field. Implements
  `__post_init__` for valve calibration tuple normalization and `save()` for tuple→dict YAML
  roundtrip.
- **Configuration dataclasses**: `MesoscopeFileSystem`, `MesoscopeGoogleSheets`, `MesoscopeCameras`,
  `MesoscopeMicroControllers`, `MesoscopeAcquisition`, `MesoscopeVRAssets`. All use `slots=True` and
  follow the field-naming convention.
- **Binding classes**: `MicroControllerInterfaces`, `VideoSystems`, `ZaberMotors` in
  `sollertia_experiment/mesoscope_vr/binding_classes.py`. `MicroControllerInterfaces` exposes the
  full `start` / `stop` / `__del__` lifecycle; `VideoSystems` starts per-camera via
  `start_face_camera` / `start_body_camera` with a unified `stop` / `__del__`; `ZaberMotors` connects
  in `__init__` and tears down via `disconnect`.
- **Lifecycle orchestrator**: `MesoscopeVRSystem` in
  `sollertia_experiment/mesoscope_vr/system_controller.py`.

For the Mesoscope-VR-specific surface — actual field names and values, binding-class
composition details, modification workflows — see `mesoscope:mesoscope-vr`. For Mesoscope-VR's
runtime states, training modes, and CLI, see `mesoscope:mesoscope-vr-runtime`.

---

## Maintenance contract

This skill documents durable design patterns. It is updated when:

- A new layer is added to the architecture (e.g., a fourth layer between the system configuration
  and the binding classes).
- A new lifecycle method becomes mandatory (e.g., a new asynchronous-asset initialization phase).
- A new naming or unit convention is adopted across the platform.
- A cross-layer contract changes (e.g., schema-versioning rules are tightened).

This skill is NOT updated when:

- A specific acquisition system gains a new subsystem or field — that's the per-system instance skill's
  domain.
- A specific binding class's internal implementation changes — the *pattern* it follows is what's
  documented here.

When you're unsure whether a change belongs here or in a per-system skill, ask: "Does this apply
to every Sollertia acquisition system, or only this one?" The pattern skill answers "every";
per-system skills answer "only this one."

---

## Related skills

| Skill                                              | Relationship                                                                                                                            |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| `/microcontroller-interface`                       | The per-module wrapper layer that binding classes compose. Authoritative for slmc/sle conventions.                                      |
| `/zaber-interface`                                 | Shared Zaber motor interface mechanics. Binding classes that include motors compose this.                                               |
| `mesoscope:mesoscope-vr`                           | The current Mesoscope-VR worked instance of this pattern.                                                                               |
| `/acquisition-system-runtime`                      | The runtime-behavior counterpart to this static-composition pattern.                                                                    |
| `mesoscope:mesoscope-vr-runtime`                   | Mesoscope-VR-specific runtime behavior (state machine, training modes, CLI). Built on this pattern.                                     |
| `/vr-driver-interface`                             | The Unity VR task driver, a standard subsystem of every acquisition system.                                                             |
| `ataraxis@video:camera-interface`                  | Low-level VideoSystem mechanics. Camera binding classes compose VideoSystem instances.                                                  |
| `ataraxis@communication:microcontroller-interface` | Low-level MicroControllerInterface mechanics. Microcontroller binding classes compose these.                                            |
| `/acquisition-system-setup`                        | Post-flash hardware discovery used to populate system configuration fields.                                                             |
| `/pipeline`                                        | End-to-end acquisition-system lifecycle orchestration context.                                                                          |
| `assets:library-extension`                         | Owns the `sollertia-shared-assets` enum/registry recipe for a new system; step 2 of the build-a-new-system workflow hands off here.     |
| `/google-sheets-processing`                        | Owns the external data-service processor category — the `SurgeryLog` / `WaterLog` API, schema contract, and custom-processor authoring. |
| `/system-design-pipeline`                          | Orchestrates this static-composition phase into the full cross-repo build of a new acquisition system.                                  |

---

## Verification checklist

```text
When designing a new system or extending an existing one:

System configuration:
- [ ] Top-level class is @dataclass (not slots) and inherits from YamlConfig
- [ ] Composes per-subsystem dataclasses via field(default_factory=...)
- [ ] Has a `name` field with a free-form human-readable label
- [ ] Implements __post_init__ if YAML shape differs from in-memory shape
- [ ] Overrides save() if YAML roundtrip needs translation
- [ ] Top-level class subclasses SystemConfiguration
- [ ] System registers its SystemConfiguration subclass via register_system_configuration at import time
- [ ] System exposes a typed get_system_configuration() accessor that validates the loaded config's type
- [ ] Configuration create / resolve / load is delegated to the shared cross_system helpers

Configuration dataclasses:
- [ ] @dataclass(slots=True)
- [ ] Class named <System><Subsystem>
- [ ] Every field follows <device>_<parameter>_<unit> naming
- [ ] Every field has an explicit type annotation
- [ ] Every field has a sensible default
- [ ] Every field has a triple-quoted docstring describing purpose + units
- [ ] Filesystem fields default to Path() (empty); user MUST set them

Binding classes:
- [ ] Constructor takes the most-shared dependency first (data_logger when the subsystem logs to it),
      then the configuration dataclass, then optional args
- [ ] _started: bool = False initialized first (subsystems with a start()/stop() lifecycle)
- [ ] Per-device wrappers instantiated as public attributes
- [ ] Underlying low-level controllers instantiated as private attributes
- [ ] __del__ calls stop() (or disconnect() for SDK-connection subsystems)
- [ ] Bring-up and tear-down are idempotent and follow the subsystem type's documented sequence
      (see references/subsystem-types.md) — the microcontroller initialize_local_assets → set_parameters
      flow is type-specific, not universal
- [ ] stop() (or disconnect()) resets the started/connected flag before tearing down
- [ ] Lifecycle methods cite the controller's start/stop order requirements

Cross-layer contract:
- [ ] Configuration field names align conceptually with wrapper constructor kwarg names
- [ ] Consumer library version bumped on any schema change
- [ ] Per-system instance skill updated with the new subsystem / field surface

Lifecycle orchestrator:
- [ ] Constructs DataLogger → binding classes → VR task driver in the documented order
- [ ] Calls .start() on each in the same order
- [ ] Calls .stop() in reverse order
- [ ] DataLogger stops only after every binding class has stopped
- [ ] Cross-subsystem signaling lives in the orchestrator, not the binding classes
- [ ] Keepalive is passed to each MicroControllerInterface at construction; AXCI enforces it
      (no orchestrator/binding keepalive-watch loop)
```
