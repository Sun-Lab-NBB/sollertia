---
name: acquisition-system-design
description: >-
  Documents the platform-general design pattern for a Sollertia data acquisition system: a top-level
  YAML system configuration composed of per-hardware-lane calibration dataclasses, per-lane binding
  classes that compose the lower-level interface instances and run their lifecycle, and a runtime
  orchestrator that owns the master start/stop. Use when designing a new acquisition system from
  scratch, adding a new hardware lane to an existing system, or auditing an existing system's
  configuration/binding layer for pattern compliance.
user-invocable: false
---

# Acquisition system design

Documents the platform-general design pattern for a Sollertia data acquisition system at the
configuration and binding-class layer. An acquisition system is the host-PC stack that composes
lower-level hardware interfaces (microcontrollers, cameras, motors, etc.) into a runnable platform
that produces session data.

This skill is a **pattern skill** — it documents the conventions and contracts that all Sollertia
acquisition systems share, but does not document any single system's specific composition. For
concrete instances, see the per-system skills (currently `experiment:mesoscope-vr`).

Detailed authoring patterns live in two reference files, loaded on demand:

- [references/layer-patterns.md](references/layer-patterns.md) — full per-layer class patterns, field
  conventions, lifecycle rules, and the cross-layer contracts.
- [references/workflows.md](references/workflows.md) — step-by-step procedures for adding a lane,
  building a new system, and extending an existing lane.

---

## Scope

**Covers:**
- Three-layer architecture (System Configuration YAML → Calibration dataclasses + Binding classes →
  Lifecycle orchestrator)
- Top-level system configuration pattern (`@dataclass` + `YamlConfig`, composition, validation, YAML roundtrip)
- Per-lane calibration dataclass pattern (naming, field conventions, units in field names)
- Per-lane binding class pattern (constructor signature, lifecycle methods, idempotency, `__del__` semantics)
- Module-level helpers for configuration file lifecycle (`create_system_configuration_file`,
  `get_system_configuration_path`, `get_system_configuration`)
- Cross-layer contracts (field naming consistency, schema versioning)
- Lifecycle ordering rules (DataLogger first, binding classes second, reverse on shutdown)
- Workflows for adding a new hardware lane, building a new acquisition system, extending an existing lane

**Does not cover** (delegated):
- The per-firmware-module Python wrapper layer (`cross_system/module_interfaces.py`) and slmc
  firmware Modules — see `experiment:microcontroller-interface`.
- Concrete Mesoscope-VR composition (the binding-class instances and their YAML field surface) — see
  `experiment:mesoscope-vr`.
- The platform-general runtime-behavior pattern (state machine, runtime loop, event dispatch) — see
  `experiment:acquisition-system-runtime`.
- Concrete Mesoscope-VR runtime behavior (state machine, training modes, CLI) — see
  `experiment:mesoscope-vr-runtime`.
- The Unity VR task driver lane — see `experiment:vr-driver-interface`.
- Low-level VideoSystem mechanics — see `ataraxis@video:camera-interface`.
- Low-level MicroControllerInterface mechanics — see `ataraxis@communication:microcontroller-interface`.
- Zaber motor interface mechanics — see `experiment:zaber-interface`.
- Per-session metadata, task templates, and experiment configuration — owned by the assets plugin
  (e.g. `assets:session-descriptors`, `assets:task-templates`, `assets:experiment-configuration`).
- Server-transfer configuration — owned by the forging plugin (`forging:server-configuration`).
- The `sollertia-shared-assets` enum/registry side of registering a new acquisition system (the
  `AcquisitionSystems` member, the dispatch registries, the per-system descriptor / hardware-state /
  experiment-config / raw-data dataclasses, and the experiment-config factory) — owned by the assets
  plugin's `/library-extension`.

---

## Three-layer architecture

A Sollertia acquisition system is composed of three layers, top-down:

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Layer 1: System Configuration YAML                                             │
│  ─────────────────────────────────                                              │
│  <System>SystemConfiguration   ── persisted to disk as <system>_configuration.yaml │
│      ├── <System>FileSystem    ── filesystem paths (host-machine state)         │
│      ├── <System>Cameras       ── per-camera calibration                        │
│      ├── <System>MicroControllers ── per-microcontroller-module calibration     │
│      ├── <System>ExternalAssets ── third-party hardware (motors, MQTT, etc.)    │
│      └── auxiliary sections     ── e.g., external service IDs                   │
└────────────────────────────┬────────────────────────────────────────────────────┘
                             │ instantiates and parameterizes
┌────────────────────────────▼────────────────────────────────────────────────────┐
│  Layer 2: Per-hardware-lane binding classes                                     │
│  ─────────────────────────────────────────                                      │
│  VideoSystems(data_logger, camera_configuration, output_directory)              │
│      └── wraps N × VideoSystem instances + per-camera lifecycle                 │
│  MicroControllerInterfaces(data_logger, microcontroller_configuration)          │
│      ├── instantiates N × ModuleInterface subclasses from calibration fields    │
│      └── wraps N × MicroControllerInterface instances + per-controller lifecycle│
│  <Lane>Bindings(<lane>_configuration, ...)                                      │
│      └── one per hardware lane (motors, custom devices, etc.)                   │
└────────────────────────────┬────────────────────────────────────────────────────┘
                             │ composed and lifecycle-orchestrated by
┌────────────────────────────▼────────────────────────────────────────────────────┐
│  Layer 3: Lifecycle orchestrator (one per acquisition system)                   │
│  ─────────────────────────────────────────────────────────                      │
│  <System>System / <System>Runtime / system_controller module                     │
│      ├── owns DataLogger and MQTTCommunication (where applicable)               │
│      ├── constructs each Layer-2 binding class in the correct order             │
│      ├── coordinates start / stop / state transitions                           │
│      └── exposes the public runtime surface to CLI / GUI consumers              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

Each layer's conventions are summarized below and documented in full in
[references/layer-patterns.md](references/layer-patterns.md). The pattern is the same for every
Sollertia acquisition system; what differs between systems is the *specific* set of hardware lanes,
the *specific* calibration fields per lane, and the *specific* runtime states.

---

## Layer 1: System configuration

The top-level system configuration is a single plain `@dataclass` (NOT `slots=True`) that inherits
from `ataraxis_data_structures.YamlConfig` and composes one nested dataclass per concern via
`field(default_factory=...)` — one per hardware lane, plus auxiliary host-state concerns like
filesystem paths and external service IDs. It carries a free-form `name` label, optionally overrides
`__post_init__` (to normalize YAML-shaped fields and validate user input) and `save()` (for
YAML-roundtrip translation), and exposes three module-level helpers for the configuration file
lifecycle: `create_system_configuration_file`, `get_system_configuration_path`, and
`get_system_configuration`.

For the full class-definition pattern, the `__post_init__` / `save()` overrides, and the
module-helper rules, see
[references/layer-patterns.md](references/layer-patterns.md#layer-1-system-configuration).

---

## Layer 2a: Per-lane calibration dataclasses

Each hardware lane embeds a `@dataclass(slots=True)` named `<System><Lane>` that captures the
**physical parameters** the binding class uses to instantiate per-device wrappers. Every field has
four properties:

- an explicit, narrow type;
- a sensible default — a reference-rig calibration value, or `Path()` for filesystem fields the user
  MUST set;
- a triple-quoted docstring;
- a name following the `<device-or-module>_<parameter>_<unit>` convention (the unit suffix is
  included whenever the unit is non-obvious).

For the field-naming table, the type conventions, and the defaults rules, see
[references/layer-patterns.md](references/layer-patterns.md#layer-2a-per-lane-calibration-dataclasses).

---

## Layer 2b: Per-lane binding classes

Each hardware lane has one binding class that composes the lane's per-device wrappers and orchestrates
their lifecycle. The constructor takes `data_logger` first, then the per-lane calibration dataclass,
then any optional inputs; it initializes `_started = False`, caches the configuration, instantiates
per-device wrappers as public attributes, and wraps them in private low-level controllers. `start()`
and `stop()` are idempotent, `__del__` calls `stop()` as a safety net, and `start()` follows the
controller-start → `initialize_local_assets()` → `set_parameters()` order. Lanes that wrap a
third-party SDK connection (e.g., the Zaber motor lane) may expose `connect` / `disconnect` plus
position methods in place of `start` / `stop`.

For the full class template, the `start()` calling-order detail, and the lifecycle-method table, see
[references/layer-patterns.md](references/layer-patterns.md#layer-2b-per-lane-binding-classes).

---

## Layer 3: Lifecycle orchestrator

One class per acquisition system (typically `<System>System` or `<System>VRSystem`, in a
`system_controller` module) composes the Layer-2 binding classes and owns the master start/stop. It
instantiates the DataLogger first (so each `MicroControllerInterface.__init__` can register a
manifest entry), constructs the binding classes in a fixed order, starts the DataLogger before any
binding class, and tears everything down in reverse so the DataLogger outlives every consumer. It also
owns keepalive enforcement and all cross-lane synchronization — individual binding classes stay
oblivious to one another.

For the construction/shutdown order diagrams, keepalive enforcement, and cross-lane signaling rules,
see [references/layer-patterns.md](references/layer-patterns.md#layer-3-lifecycle-orchestrator).

---

## Cross-layer contracts

Three contracts must hold for the architecture to function:

1. **Calibration field naming agreement** — each calibration dataclass field that feeds a wrapper
   constructor matches the wrapper's keyword-argument name conceptually (unit suffixes may be dropped
   at the API boundary).
2. **Schema versioning** — any add / remove / rename / type-change of a calibration field is a schema
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

- **Adding a new hardware lane to an existing system** — author the calibration dataclass and binding
  class, wire them into the orchestrator, and regenerate the configuration YAML.
- **Building a new acquisition system from scratch** — hand the `sollertia-shared-assets` registry
  work to `assets:library-extension`, then author the dataclasses, helpers, binding classes,
  orchestrator, and CLI group.
- **Extending an existing lane** — add fields to an existing calibration dataclass and its binding
  class (the most common change).

---

## Auxiliary sections beyond hardware lanes

The system configuration also captures host-machine state that isn't a hardware lane:

| Section type       | Typical contents                                                            |
|--------------------|-----------------------------------------------------------------------------|
| Filesystem section | `root_directory`, `nas_directory`, `server_directory`, etc. (`Path` fields) |
| External services  | Google Sheets IDs, Slack webhook URLs, hosted endpoint URLs                 |
| Network settings   | MQTT broker IP and port for cross-process / cross-machine communication     |
| Software paths     | Paths to third-party acquisition software the system orchestrates           |

These follow the same dataclass pattern as hardware lanes but don't have binding classes — they're
consumed directly by orchestrator code or by per-session setup steps.

**Filesystem fields rule:** Every filesystem field SHOULD be checked at configuration load time via
a `check_system_mounts_tool` (or equivalent) that verifies the path exists and is writable. The
check is system-specific; the pattern is that the system configuration's MCP tooling exposes a
mount-check entry point.

---

## Mesoscope-VR as a worked example

The Mesoscope-VR acquisition system is the current consumer of every pattern in this skill:

- **System Configuration**: `MesoscopeSystemConfiguration` in
  `sollertia_experiment/mesoscope_vr/system.py`. Composes 5 sections (filesystem, sheets,
  cameras, microcontrollers, assets) plus a top-level `name` field. Implements
  `__post_init__` for valve calibration tuple normalization and `save()` for tuple→dict YAML
  roundtrip.
- **Calibration dataclasses**: `MesoscopeFileSystem`, `MesoscopeGoogleSheets`, `MesoscopeCameras`,
  `MesoscopeMicroControllers`, `MesoscopeVRAssets`. All use `slots=True` and follow the
  field-naming convention.
- **Binding classes**: `MicroControllerInterfaces`, `VideoSystems`, `ZaberMotors` in
  `sollertia_experiment/mesoscope_vr/binding_classes.py`. `MicroControllerInterfaces` exposes the
  full `start` / `stop` / `__del__` lifecycle; `VideoSystems` starts per-camera via
  `start_face_camera` / `start_body_camera` with a unified `stop` / `__del__`; `ZaberMotors` connects
  in `__init__` and tears down via `disconnect`.
- **Lifecycle orchestrator**: `_MesoscopeVRSystem` in
  `sollertia_experiment/mesoscope_vr/system_controller.py`.

For the Mesoscope-VR-specific surface — actual field names, calibration values, binding-class
composition details, modification workflows — see `experiment:mesoscope-vr`. For Mesoscope-VR's
runtime states, training modes, and CLI, see `experiment:mesoscope-vr-runtime`.

---

## Maintenance contract

This skill documents durable design patterns. It is updated when:

- A new layer is added to the architecture (e.g., a fourth layer between the system configuration
  and the binding classes).
- A new lifecycle method becomes mandatory (e.g., a new asynchronous-asset initialization phase).
- A new naming or unit convention is adopted across the platform.
- A cross-layer contract changes (e.g., schema-versioning rules are tightened).

This skill is NOT updated when:

- A specific acquisition system gains a new lane or field — that's the per-system instance skill's
  domain.
- A specific binding class's internal implementation changes — the *pattern* it follows is what's
  documented here.

When you're unsure whether a change belongs here or in a per-system skill, ask: "Does this apply
to every Sollertia acquisition system, or only this one?" The pattern skill answers "every";
per-system skills answer "only this one."

---

## Related skills

| Skill                                              | Relationship                                                                                                                        |
|----------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `experiment:microcontroller-interface`             | The per-module wrapper layer that binding classes compose. Authoritative for slmc/sle conventions.                                  |
| `experiment:zaber-interface`                       | Shared Zaber motor interface mechanics. Binding classes that include motors compose this.                                           |
| `experiment:mesoscope-vr`                          | The current Mesoscope-VR worked instance of this pattern.                                                                           |
| `experiment:acquisition-system-runtime`            | The runtime-behavior counterpart to this static-composition pattern.                                                                |
| `experiment:mesoscope-vr-runtime`                  | Mesoscope-VR-specific runtime behavior (state machine, training modes, CLI). Built on this pattern.                                 |
| `experiment:vr-driver-interface`                   | The Unity VR task driver lane an acquisition system composes for VR coupling.                                                       |
| `ataraxis@video:camera-interface`                  | Low-level VideoSystem mechanics. Camera binding classes compose VideoSystem instances.                                              |
| `ataraxis@communication:microcontroller-interface` | Low-level MicroControllerInterface mechanics. Microcontroller binding classes compose these.                                        |
| `experiment:acquisition-system-setup`              | Post-flash hardware discovery used to populate system configuration fields.                                                         |
| `experiment:pipeline`                              | End-to-end acquisition-system lifecycle orchestration context.                                                                      |
| `assets:library-extension`                         | Owns the `sollertia-shared-assets` enum/registry recipe for a new system; step 2 of the build-a-new-system workflow hands off here. |

---

## Verification checklist

```text
When designing a new system or extending an existing one:

System configuration:
- [ ] Top-level class is @dataclass (not slots) and inherits from YamlConfig
- [ ] Composes per-lane dataclasses via field(default_factory=...)
- [ ] Has a `name` field with a free-form human-readable label
- [ ] Implements __post_init__ if YAML shape differs from in-memory shape
- [ ] Overrides save() if YAML roundtrip needs translation
- [ ] Module-level helpers exist: create_system_configuration_file, get_system_configuration_path,
      get_system_configuration
- [ ] create_system_configuration_file rejects AcquisitionSystems values other than the supported system

Calibration dataclasses:
- [ ] @dataclass(slots=True)
- [ ] Class named <System><Lane>
- [ ] Every field follows <device>_<parameter>_<unit> naming
- [ ] Every field has an explicit type annotation
- [ ] Every field has a sensible default
- [ ] Every field has a triple-quoted docstring describing purpose + units
- [ ] Filesystem fields default to Path() (empty); user MUST set them

Binding classes:
- [ ] Constructor takes data_logger first, then calibration dataclass, then optional args
- [ ] _started: bool = False initialized first
- [ ] Per-device wrappers instantiated as public attributes
- [ ] Underlying low-level controllers instantiated as private attributes
- [ ] __del__ calls stop()
- [ ] start() is idempotent and follows the controller → initialize_local_assets → set_parameters order
- [ ] stop() is idempotent and resets _started before tearing down
- [ ] Lifecycle methods cite the controller's start/stop order requirements

Cross-layer contract:
- [ ] Calibration field names align conceptually with wrapper constructor kwarg names
- [ ] Consumer library version bumped on any schema change
- [ ] Per-system instance skill updated with the new lane / field surface

Lifecycle orchestrator:
- [ ] Constructs DataLogger → MQTTCommunication → binding classes in the documented order
- [ ] Calls .start() on each in the same order
- [ ] Calls .stop() in reverse order
- [ ] DataLogger stops only after every binding class has stopped
- [ ] Keepalive enforcement and cross-lane signaling live in the orchestrator, not the binding classes
```
