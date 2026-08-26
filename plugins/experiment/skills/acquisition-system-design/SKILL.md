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
**additional PCs** each dedicated to a specific instrument. The main PC composes the lower-level
hardware interfaces (microcontrollers, cameras, motors, and similar devices) into a runnable
platform that produces session data. An additional PC runs its own acquisition software and is
coordinated through filesystem paths and network settings in the system configuration rather than
through a binding class.

This skill is a **pattern skill**. It documents the conventions and contracts that all Sollertia
acquisition systems share, and it documents no single system's specific composition. For a concrete
instance, see the [Worked example](#worked-example) section and `mesoscope:mesoscope-vr`.

Detailed authoring patterns live in three reference files, loaded on demand:

- [references/layer-patterns.md](references/layer-patterns.md) covers the per-layer class patterns,
  field conventions, lifecycle rules, and the cross-layer contracts.
- [references/subsystem-types.md](references/subsystem-types.md) covers the per-type lifecycle surface
  of each major subsystem type on the platform: microcontroller, camera, and motor or SDK.
- [references/workflows.md](references/workflows.md) covers the step-by-step procedures for adding a
  subsystem, building a new system, and extending an existing subsystem.

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

**Does not cover:**
- The per-firmware-module Python wrapper layer (`cross_system/module_interfaces.py`) and the slmc
  firmware Modules. Owned by `/microcontroller-interface`.
- The seam catalog a new system composes: the configuration registry, the shared `cross_system`
  primitives, the MCP and CLI seams, and the slmc module, target, and board seams. Owned by
  `/library-extension`.
- Concrete Mesoscope-VR composition, meaning the binding-class instances and their YAML field
  surface. Owned by `mesoscope:mesoscope-vr`.
- The platform-general runtime-behavior pattern (state machine, runtime loop, event dispatch).
  Owned by `/acquisition-system-runtime`.
- Concrete Mesoscope-VR runtime behavior. Owned by `mesoscope:mesoscope-vr-runtime`.
- The Unity VR task driver subsystem. Owned by `/vr-driver-interface`.
- Low-level VideoSystem mechanics. Owned by `video:camera-interface` (ataraxis marketplace).
- Low-level MicroControllerInterface mechanics. Owned by `communication:microcontroller-interface`.
- Zaber motor interface mechanics. Owned by `/zaber-interface`.
- Per-session metadata, task templates, and experiment configuration. Owned by the assets plugin,
  through `assets:session-descriptors`, `assets:task-templates`, and `assets:experiment-configuration`.
- The `sollertia-shared-assets` enum and registry side of registering a new acquisition system: the
  `AcquisitionSystems` member, the dispatch registries, the per-system descriptor, hardware-state,
  experiment-config, and raw-data dataclasses, and the `from_task_template` experiment-configuration
  builder. Owned by `assets:library-extension`.
- The implementation of external data-service processors, meaning the Google Sheets `SurgeryLog` and
  `WaterLog` classes, their schema contract, and authoring a custom one. Owned by
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
│  <Cameras>Bindings(data_logger, camera_configuration, output_directory)         │
│      └── wraps N × VideoSystem instances + per-camera lifecycle                 │
│  <Boards>Bindings(data_logger, microcontroller_configuration)                   │
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
[references/layer-patterns.md](references/layer-patterns.md). The pattern is the same for every Sollertia acquisition
system. What differs between systems is the *specific* set of hardware subsystems, the *specific* configuration fields
per subsystem, and the *specific* runtime states.

---

## Layer 1: System configuration

The top-level system configuration is a single plain `@dataclass` (NOT `slots=True`) that inherits
from `ataraxis_data_structures.YamlConfig` and composes one nested dataclass per concern via
`field(default_factory=...)`. Each hardware subsystem gets one section, and auxiliary host-state
concerns such as filesystem paths and external service identifiers get theirs. The class subclasses
`SystemConfiguration`, carries a free-form `name` label, and optionally overrides `__post_init__`
(to normalize YAML-shaped fields and validate user input) and `save()` (for YAML-roundtrip
translation).

The shared cross-system registry owns the configuration-file lifecycle. `register_system_configuration`
is the only system-specific wiring that lifecycle requires, and everything else is shared
(`cross_system/system_configuration.py`). `create_system_configuration_file` writes a
**default-constructed** instance of the registered class, and only then unlinks every other
`*_system_configuration.yaml` in the same directory, so a failed write leaves the host with its
previous system identity. It identifies the file it just wrote by inode through `samefile`, because a
case-insensitive filesystem keeps a differently-cased directory entry for that same file
(`cross_system/system_configuration.py`). `get_system_configuration_path` raises
`FileNotFoundError` unless exactly one file matches the glob (`cross_system/system_configuration.py`),
and `_system_configuration_filename` derives the filename from the enum value as
`f"{system}_system_configuration.yaml"`. Each system's module registers its `SystemConfiguration`
subclass at import time and exposes a typed `get_system_configuration()` accessor.

For the full class-definition pattern, the `__post_init__` and `save()` overrides, and the shared configuration-file
lifecycle, see [references/layer-patterns.md](references/layer-patterns.md#layer-1-system-configuration).

---

## Layer 2a: Per-subsystem configuration dataclasses

Each hardware subsystem embeds a `@dataclass(slots=True)` named `<System><Subsystem>` that captures the
**parameters** the binding class uses to instantiate per-device wrappers. Every field has
four properties:

- an explicit, narrow type,
- a sensible default, meaning a reference-rig value where the parameter is a measured calibration and a
  working factory setting otherwise, or `Path()` for a filesystem field, which reads as not configured
  until the deployment sets it,
- a triple-quoted docstring,
- a name following the `<device-or-module>_<parameter>_<unit>` convention (the unit suffix is
  included whenever the unit is non-obvious).

**GenTL and GenICam camera subsystems** follow an additional standard practice. Each camera declares
an optional configuration-file path field, `<role>_camera_configuration_path: Path = Path()`, where
an empty path reads as unset. The field points at an `ataraxis-video-system` `GenicamConfiguration`
YAML that records the camera's expected node configuration. The field is declarative. It records
where the expected configuration lives so agents can verify the live camera against it, and dump or
restore it on request, and the acquisition runtime never auto-applies it. By convention these YAMLs
live in the working-directory `configuration/` folder beside the system configuration. The GenICam
dump and restore mechanics are owned by `video:camera-setup`. A system's own MCP tool module may add
a verify tool that diffs the live configuration against the stored file. *Worked example:* see
`mesoscope:mesoscope-vr`.

For the field-naming table, the type conventions, and the defaults rules, see
[references/layer-patterns.md](references/layer-patterns.md#layer-2a-per-subsystem-configuration-dataclasses).

---

## Layer 2b: Per-subsystem binding classes

Each hardware subsystem has one binding class that composes the subsystem's per-device wrappers and
orchestrates their lifecycle. The **shared contract** is the same across subsystem types. The
constructor takes the most-shared dependency first (`data_logger`, when the subsystem logs to it),
then the per-subsystem configuration dataclass, then any optional inputs. It caches the configuration,
instantiates per-device wrappers as public attributes, and wraps them in private low-level
controllers. Bring-up and tear-down are idempotent for the microcontroller and camera types, and
`__del__` calls the tear-down as a safety net. A third-party-SDK subsystem connects in `__init__` and
relies on the orchestrator calling its `disconnect()` explicitly.

The bring-up flag is set **before** the first bring-up step, so a failure partway through still routes
through the tear-down, and each low-level controller's own tear-down self-guards. The tear-down clears
that flag only **after** every step has run, so a failure severe enough to escape the isolated steps
leaves the instance stoppable on a retry. Each step of a multi-device tear-down is wrapped in
`run_shutdown_step`, which catches the failure and echoes an ERROR so later steps still run
(`cross_system/shutdown_tools.py`).

The **bring-up sequence and method surface are specific to each subsystem type:**

- **Microcontroller subsystems** expose `start()` and `stop()`. `start()` brings the controllers
  online, attaches host-side shared-memory assets through `initialize_local_assets()`, then pushes
  runtime parameters through `set_parameters()`.
- **Camera subsystems** split acquisition from saving, exposing `start_<role>_camera()` and then
  `save_<role>_camera_frames()`, with a unified `stop()`. Each camera's parameters are set at
  construction.
- **Third-party-SDK subsystems**, such as the Zaber motor subsystem, connect in `__init__` and expose
  `disconnect` plus position methods for their lifecycle. They may omit the `data_logger`.

For the shared class template and the lifecycle-method conventions, see
[references/layer-patterns.md](references/layer-patterns.md#layer-2b-per-subsystem-binding-classes). For the per-type
bring-up sequences, see [references/subsystem-types.md](references/subsystem-types.md).

---

## Layer 3: Lifecycle orchestrator

One class per acquisition system (typically `<System>System` or `<System>VRSystem`, in a `system_controller` module)
composes the Layer-2 binding classes and owns the master start/stop. It instantiates the DataLogger first (so each
`MicroControllerInterface.__init__` can register a manifest entry), constructs the binding classes in a fixed order,
starts the DataLogger before any binding class, and tears everything down in reverse so the DataLogger outlives every
consumer. It also owns all cross-subsystem synchronization, and individual binding classes stay oblivious to one
another. The VR task driver is a standard subsystem of every acquisition system, because every Sollertia system presents
a Unity task in the linear infinite corridor, as the `AcquisitionSystems` docstring in
`sollertia-shared-assets/src/sollertia_shared_assets/enums.py` states. The orchestrator constructs the driver only for
the session types in `SESSION_TYPES_USING_VR_TASK` (`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`)
and holds `None` for every other session type. See `/acquisition-system-runtime` and `/vr-driver-interface`. For
microcontroller keepalive, the orchestrator passes each `MicroControllerInterface` a `keepalive_interval` at
construction, and AXCI sends the keepalive messages and raises on timeout.

For the construction/shutdown order diagrams, keepalive handling, and cross-subsystem signaling rules,
see [references/layer-patterns.md](references/layer-patterns.md#layer-3-lifecycle-orchestrator).

---

## Cross-layer contracts

Three contracts must hold for the architecture to function:

1. **Configuration field naming agreement**: each configuration dataclass field that feeds a wrapper
   constructor matches the wrapper's keyword-argument name conceptually (unit suffixes may be dropped
   at the API boundary).
2. **Schema versioning**: any add / remove / rename / type-change of a configuration field is a schema
   change and MUST be paired with a version bump on the owning package. Older YAML must fail loudly
   rather than silently misconfigure.
3. **Lifecycle ordering**: the DataLogger instance exists before any binding class `__init__`, the DataLogger is
   started before any binding class `start()`, `start()` precedes any wrapper command, and `stop()` precedes
   DataLogger shutdown.

For the field→kwarg table and the full per-contract rules, see
[references/layer-patterns.md](references/layer-patterns.md#cross-layer-contracts).

---

## Workflows

Four common changes each have a step-by-step procedure in [references/workflows.md](references/workflows.md):

- **Adding a new hardware subsystem to an existing system.** Author the configuration dataclass and
  binding class, wire them into the orchestrator, and regenerate the configuration YAML.
- **Building a new acquisition system from scratch.** Hand the `sollertia-shared-assets` registry work
  to `assets:library-extension`, hand the seam and phase-order questions to
  `/library-extension`, then author the dataclasses, helpers, binding classes, orchestrator,
  and CLI group.
- **Extending an existing subsystem.** Add fields to an existing configuration dataclass and its
  binding class. This is the most common change.
- **Authoring a custom data-service processor.** Reuse or author a `SurgeryLog` or `WaterLog`-style
  processor for an external request/response service, such as a Google Sheet or a LIMS. See also
  `/google-sheets-processing`.

---

## Auxiliary sections beyond hardware subsystems

Beyond the per-subsystem sections, the system configuration also captures host- and system-level
state that is not a hardware subsystem and has no binding class: filesystem paths, external-service
identifiers, network endpoints, and parameters for command-line tools the stack runs as subprocesses.
Which of these a system needs is system-specific. Four categories recur:

| Auxiliary section | Holds                                               | Kind of value                     |
|-------------------|-----------------------------------------------------|-----------------------------------|
| Filesystem paths  | Mount points for acquired data and storage          | `Path` fields, empty until set    |
| External services | Identifiers of services the runtime reads or writes | Opaque identifier strings         |
| Network settings  | Broker endpoint for cross-machine messaging         | A host address and a port         |
| External tools    | Parameters for a tool run as a subprocess           | An environment, a path, and knobs |

These follow the same dataclass pattern as hardware subsystems and have no binding class, so
orchestrator code or per-session setup and preprocessing steps consume them directly. The **Filesystem
paths** and **Network settings** rows are also where the main PC coordinates any additional PCs. An
instrument PC running its own acquisition stack is reached through the mounted paths and network
endpoints declared here, never through a binding class. *Worked example:* Mesoscope-VR runs one such
instrument PC. See `mesoscope:mesoscope-vr`.

The **External services** row holds only the identifiers. The code that reads records from, and writes results back to,
those services is a **data-service processor**, a separate category that also has no binding class and is constructed by
per-session setup or preprocessing code. For its lifecycle surface see the "External data-service processors" category
in [references/subsystem-types.md](references/subsystem-types.md). For the Google Sheets processors `SurgeryLog` and
`WaterLog`, their schema contract, and authoring a custom one, see `/google-sheets-processing`.

The **External tools** row is the out-of-process external-tool binding category. A section in this category owns no
binding class and instead parameterizes a command-line tool that the stack runs as a subprocess, because the tool needs
a Python environment the acquisition process cannot import. Its fields carry the environment name, the tool's model or
project path, and the tool's runtime knobs, and leaving those values empty disables the tool for that deployment. An
out-of-process tool is invoked from the system's own preprocessing code, so its exit status and its written outputs are
that system's contract to define. *Worked example:* Mesoscope-VR declares one such tool. See `mesoscope:mesoscope-vr`.

**Filesystem fields rule:** Every filesystem field SHOULD be verifiable on demand through a mount-check tool the
system's own `<system>_tools.py` module exposes, which reports whether the path exists and is writable. The check is
system-specific, and the pattern is that the system configuration's MCP tooling exposes a mount-check entry point that
an agent or operator invokes. An unset root for an optional storage destination reports as not configured with an ok
status, so the feature that consumes it is skipped. A path the system writes to is checked for existence and
writability. A path the system only reads, such as a stored device configuration or an external tool's project file,
is checked for existence and readability instead, because a write probe would reject a valid read-only input. Sections
outside the filesystem section contribute their own paths to the same report, so the check covers every declared path
rather than one section. For the current worked example, see `mesoscope:mesoscope-vr`.

---

## Worked example

`AcquisitionSystems` currently holds one member, so Mesoscope-VR is the only registered acquisition system and the only
concrete consumer of every pattern above. For its configuration surface, binding-class composition, and modification
workflows, see `mesoscope:mesoscope-vr`. For its runtime states, session modes, and CLI, see
`mesoscope:mesoscope-vr-runtime`. Read `mesoscope_vr/system.py` and `mesoscope_vr/binding_classes.py` for the Layer-1
and Layer-2 source.

---

## Extension

`/library-extension` owns the seam catalog a new acquisition system composes: the
configuration-registry seam, the shared `cross_system` primitives a new system reuses rather than
rewrites, and the MCP tool-module and CLI group registration seams in `interfaces/`. It also records
that the platform exposes no generic runtime base class, so each system writes its own controller
against those seams. The "Building a new acquisition system from scratch" workflow in
[references/workflows.md](references/workflows.md) defers every phase-order question to that skill.

---

## Maintenance contract

This skill documents durable design patterns. It is updated when:

- A new layer is added to the architecture (e.g., a fourth layer between the system configuration
  and the binding classes).
- A new lifecycle method becomes mandatory (e.g., a new asynchronous-asset initialization phase).
- A new naming or unit convention is adopted across the platform.
- A cross-layer contract changes (e.g., schema-versioning rules are tightened).

This skill is NOT updated when:

- A specific acquisition system gains a new subsystem or field. That belongs to the per-system
  instance skill.
- A specific binding class changes its internal implementation. This skill documents the *pattern*
  that class follows.

When you are unsure whether a change belongs here or in a per-system skill, ask whether it applies
to every Sollertia acquisition system or only to one. The pattern skill answers "every", and a
per-system skill answers "only this one".

---

## Related skills

The last two rows resolve through the ataraxis marketplace.

| Skill                                     | Relationship                                                       |
|-------------------------------------------|--------------------------------------------------------------------|
| `/microcontroller-interface`              | The per-module wrapper layer that binding classes compose.         |
| `/zaber-interface`                        | Zaber motor mechanics a motor binding class composes.              |
| `/acquisition-system-runtime`             | The runtime counterpart to this static-composition pattern.        |
| `/vr-driver-interface`                    | The Unity VR task driver every acquisition system carries.         |
| `/acquisition-system-setup`               | Hardware discovery that populates configuration fields.            |
| `/pipeline`                               | Operate-time phase ordering for an existing system.                |
| `/system-design-pipeline`                 | Build-time phase ordering across the four repositories.            |
| `/google-sheets-processing`               | The external data-service processor category and its schema.       |
| `/library-extension`                      | The seam catalog a new system composes in sle and slmc.            |
| `assets:library-extension`                | The shared-assets enum and registry recipe for a new system.       |
| `mesoscope:mesoscope-vr`                  | The current worked instance of this pattern.                       |
| `mesoscope:mesoscope-vr-runtime`          | The current worked instance's runtime behavior.                    |
| `video:camera-interface`                  | VideoSystem mechanics a camera binding class composes.             |
| `communication:microcontroller-interface` | MicroControllerInterface mechanics a board binding class composes. |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

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
- [ ] Filesystem fields default to Path() (empty), which reads as not configured until a deployment sets it

Binding classes:
- [ ] Constructor takes the most-shared dependency first (data_logger when the subsystem logs to it),
      then the configuration dataclass, then optional args
- [ ] A bring-up flag initialized first when the class exposes a bring-up and tear-down pair, one per
      independently startable device where the type has several
- [ ] Per-device wrappers instantiated as public attributes
- [ ] Underlying low-level controllers instantiated as private attributes
- [ ] __del__ calls stop() for the microcontroller and camera types, while an SDK-connection subsystem
      is disconnected by the orchestrator instead
- [ ] Bring-up and tear-down are idempotent and follow the subsystem type's documented sequence
      (see references/subsystem-types.md). The microcontroller initialize_local_assets then
      set_parameters flow is type-specific rather than universal
- [ ] start() sets the bring-up flag before the first bring-up step, so a partial failure routes through stop()
- [ ] stop() clears the flag only after every teardown step, so a failure leaves the instance stoppable on retry
- [ ] Each teardown step of a multi-device subsystem is isolated through run_shutdown_step
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
- [ ] Keepalive is passed to each MicroControllerInterface at construction and AXCI enforces it,
      so the orchestrator and the binding classes run no keepalive-watch loop of their own
```
