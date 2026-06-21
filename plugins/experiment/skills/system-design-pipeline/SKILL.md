---
name: system-design-pipeline
description: >-
  End-to-end orchestration guide for designing and building a new Sollertia acquisition system:
  phase ordering and handoff conditions across the shared-assets contract, the hardware interfaces,
  the acquisition runtime, and the Unity corridor task. Use when planning or building a new
  acquisition system from scratch, or deciding which design skill to invoke next. For operating an
  already-built system, see `/pipeline`.
user-invocable: false
---

# Sollertia system design pipeline

End-to-end orchestration reference for building a new Sollertia acquisition system. Covers the
canonical build-phase ordering, the handoff conditions between phases, the cross-repository ordering
hazards, and the hand-off to `/pipeline` for operating the finished system.

This skill is the build-time counterpart to `/pipeline`: this skill builds a new acquisition system;
`/pipeline` operates one that already exists. Mesoscope-VR is the current worked example throughout;
substitute the system you are building wherever a Mesoscope-VR skill or class is named.

---

## Scope

**Covers:**
- Canonical phase ordering for building a new acquisition system across the four owning layers
- Handoff conditions that gate each phase before the next begins
- The cross-repository ordering hazards (which repo must be complete before the next imports)
- Where the assets, unity, and ataraxis plugins fit into the build, and the hand-off to `/pipeline`

**Does not cover:**
- The line-by-line authoring for any individual phase — each phase dispatches to its owning skill
- Operating an already-built system (configure, run, preprocess) — see `/pipeline`
- The Unity corridor scene mechanics beyond template authoring — see the unity plugin's
  `unity:task-prefabs` and `unity:task-scenes`
- MCP server connectivity — see each plugin's `*-mcp-environment-setup` skill

**Handoff rules:** This skill dispatches to the owning skill at each phase. Always invoke the
relevant skill for the detailed authoring recipe, parameter reference, and troubleshooting.

---

## The build-then-operate split

A new acquisition system is built across four layers, each owned by a different repository and
plugin. Building all four is design-time, AI-assisted work: authoring against documented contracts,
verifying hardware interfaces through the ataraxis MCP servers, and wiring the system package. Once
the four layers are complete, the system is *operated* through `/pipeline`, and the Sollertia
configuration-time / runtime split applies (runtime acquisition is deterministic and AI-independent).

| Layer                  | Repository                                             | Owning skill                                                 |
|------------------------|--------------------------------------------------------|--------------------------------------------------------------|
| Shared-assets contract | `sollertia-shared-assets`                              | `assets:library-extension`                                   |
| Hardware interfaces    | `sollertia-micro-controllers` + `sollertia-experiment` | `/microcontroller-interface` (+ ataraxis)                    |
| Acquisition runtime    | `sollertia-experiment`                                 | `/acquisition-system-design` → `/acquisition-system-runtime` |
| Corridor task          | `sollertia-virtual-reality`                                | `assets:task-templates` → `unity:task-prefabs`               |

### Phase ordering is load-bearing

The repositories import one another, so the phases run in dependency order, not parallel.
`sollertia-experiment` imports `sollertia-shared-assets`, and `sollertia-shared-assets` runs its
import-time parity and contract checks on a bare import. A half-wired shared-assets contract makes
`import sollertia_shared_assets` raise, which also breaks every `sle` entry point. Complete and
verify each phase's handoff condition before starting the next.

---

## Build phases

```text
1. Shared-assets contract   assets:library-extension  (+ the shared-assets README recipes)
2. Hardware interfaces      /microcontroller-interface, ataraxis firmware-module / camera-interface, /zaber-interface
3. Acquisition runtime      /acquisition-system-design → /acquisition-system-runtime
4. Corridor task            assets:task-templates → unity:task-prefabs
5. External services (opt.) /google-sheets-processing  (read/write processors; read-asset via assets:library-extension)
6. Agentic assets (opt.)    /acquisition-system-design (per-system instance + runtime skills)
→  Operate                  /pipeline  (configure a host and run the first session)
```

### Phase 1: Shared-assets contract

- **Plugin / Skill:** `assets:library-extension`, which defers the line-by-line recipe to the
  `sollertia-shared-assets` README ("Adding New Acquisition Systems", and "Adding a New Trial Class" /
  "Adding a New Trigger Type" / "Adding New Session Types" as the system needs them).
- **Actions:** Append the `AcquisitionSystems` member; author the `<system>/` subpackage holding the
  `<System>HardwareState`, the `<System>ExperimentConfiguration` (with its own trial classes and
  `from_task_template` builder), the `<System>RawData`, and the per-session-type descriptors; register
  each in `HARDWARE_STATE_REGISTRY`, `EXPERIMENT_CONFIGURATION_REGISTRY`, `SYSTEM_RAW_DATA_REGISTRY`,
  `DESCRIPTOR_REGISTRY`, and `SYSTEM_SESSION_TYPES`; re-export the classes from the top-level package;
  bump the `sollertia-shared-assets` version.
- **Ordering hazard:** This phase is first and must fully complete. Because `sle` imports
  `sollertia-shared-assets`, a half-wired registry breaks both packages' imports.
- **Handoff condition:** `python -c "import sollertia_shared_assets"` succeeds — the import-time
  `_assert_registry_coverage`, `_assert_descriptor_contract`, and `_assert_experiment_configuration_contract`
  checks all pass — and the new system appears in `list_supported_acquisition_systems_tool`.

### Phase 2: Hardware interfaces

- **Plugin / Skill:** `/microcontroller-interface` owns the Sollertia paired-module stack — the module
  catalog, type-code allocation, and the "Adding a paired Module + Interface" workflow — and delegates the
  per-side base mechanics to `ataraxis@microcontroller:firmware-module` (the firmware `Module`) and
  `ataraxis@communication:microcontroller-interface` (the PC-side `ModuleInterface`). Cameras use
  `ataraxis@video:camera-interface`; motors use `/zaber-interface`.
- **Actions:** For each hardware module the system drives that the shared module catalog does not
  already provide, author the paired C++ `Module` and Python `ModuleInterface`, allocating a type code
  (the catalog tracks the next free code) and instance ids. Hardware already in the catalog is reused
  with no new code. The firmware and PC sides must agree on the protocol — the type code, the
  `PACKED_STRUCT` command and parameter layouts, and the event codes.
- **System instrument:** Beyond the ataraxis-contracted hardware above, the system's primary scientific
  instrument is integrated through a bespoke, system-specific driver — no shared contract exists because
  each instrument differs. Mesoscope-VR, for example, drives its mesoscope through a custom ScanImage bridge
  (MQTT plus a MATLAB function). Author the instrument's driver as a Layer-2 subsystem in Phase 3 via
  `/acquisition-system-design`.
- **Cross-plugin handoffs:**
  - `ataraxis@communication:microcontroller-setup` to discover controllers and verify MQTT
  - `ataraxis@video:camera-setup` to discover and exercise cameras
- **Handoff condition:** Every new `Module` + `ModuleInterface` pair round-trips commands and data
  against connected hardware through the ataraxis MCP servers; reused catalog modules need no new code.

### Phase 3: Acquisition runtime

- **Plugin / Skill:** `/acquisition-system-design` (static composition) → `/acquisition-system-runtime`
  (the runtime loop). Follow the "Building a new acquisition system from scratch" workflow and the
  per-package deliverables manifest in `/acquisition-system-design`'s `references/workflows.md`.
- **Actions:** Author the system package: the per-subsystem configuration dataclasses, the
  `<System>SystemConfiguration` subclass with its `register_system_configuration` call and typed
  `get_system_configuration` accessor, the binding classes, the lifecycle orchestrator, the per-mode
  logic functions, the visualizer / UIs / instrument driver the system needs, the `sle <system>` CLI
  command group (registered in `interfaces/entry_points.py`), and the `interfaces/<system>_tools.py`
  MCP tool module. Bump the `sollertia-experiment` version.
- **Handoff condition:** The `sle <system>` CLI group is reachable; `read_system_configuration_tool`
  validates the system configuration; the per-system MCP tools register (the server discovers every
  `*_tools.py` module by filename suffix).

### Phase 4: Corridor task

- **Plugin / Skill:** `assets:task-templates` (author the `TaskTemplate`) → unity plugin
  `unity:task-prefabs` and `unity:task-scenes` (generate the prefab and scene).
- **Actions:** Author a corridor task template for the system. Every Sollertia system runs the linear
  infinite corridor, so a new system reuses the existing corridor — this is template authoring rather
  than new scene engineering. The experiment configuration's `unity_scene_name` must match the task
  template's filename stem so `SessionData.create()` resolves the snapshot.
- **Handoff condition:** The experiment configuration's `unity_scene_name` resolves to a real task
  template and Unity scene.

### Phase 5: External data-service processors (optional)

- **Plugin / Skill:** `/google-sheets-processing` owns the external data-service processors — the
  `SurgeryLog` / `WaterLog` Google Sheets classes, their schema contract, and authoring a custom one.
- **Actions:** If the system reads from or writes to an external request/response data service (a Google
  Sheet, a LIMS, a registry), decide reuse-versus-author. Reuse `SurgeryLog` / `WaterLog` when the source
  matches the Sollertia schema; author a new processor when the schema or service differs. A read processor
  parses records into a typed on-disk dataclass; a write processor writes runtime-discovered values back. A
  reusable processor lives in `cross_system/`; a system-specific one in the system package. Wire it into
  preprocessing, gating on the configured identifiers.
- **Read-asset coupling:** If a read processor emits a record type that is not an existing
  `sollertia-shared-assets` read asset, register the new asset through `assets:library-extension`
  ("Adding a New Read Asset") and its read / amend surface through `assets:data-assets` (Phase 1
  contract work) before wiring the processor.
- **Credentials:** A request/response service authenticates with a credentials file managed by
  `sollertia-shared-assets` (the `CredentialsTypes` registry — currently only `google`, for Google Sheets). A
  service needing a new credential category registers it in slsa (a `CredentialsTypes` + `CREDENTIALS_FILE_REGISTRY`
  entry, a maintainer-curated contract decision in Phase 1); the credential file itself is configured per host at
  operate time (`/pipeline` → `assets:working-directory`).
- **Handoff condition:** The processor authenticates and round-trips against the external source; a read
  processor's records validate and snapshot to disk.

### Phase 6: Agentic assets (optional but recommended)

- **Plugin / Skill:** `/acquisition-system-design` (workflow steps 10–11).
- **Actions:** Author a per-system instance skill and, when the system has non-trivial runtime modes,
  a per-system runtime skill — modeled on `mesoscope:mesoscope-vr` and `mesoscope:mesoscope-vr-runtime`.
  The system runs without them, but omitting them leaves it driveable yet undocumented for agents.
- **Handoff condition:** The system has an instance skill the operate pipeline can dispatch to.

### Operate: hand off to `/pipeline`

Once the four build layers pass their handoff conditions, the system type exists. Hand off to
`/pipeline` to configure a host for the new system, bring up its hardware, author an experiment, and
run the first session.

---

## Cross-repository couplings

Each coupling spans two repositories, so a change on one side requires the matching change on the
other before the system imports or runs.

| Coupling                                                               | Both sides must agree on                                       |
|------------------------------------------------------------------------|----------------------------------------------------------------|
| `AcquisitionSystems` member ↔ the `sle` runtime + CLI + MCP module     | the system identifier; slsa is registered before sle imports   |
| Firmware `Module` ↔ PC `ModuleInterface`                               | the type code, command / parameter layout, and event codes     |
| `<System>ExperimentConfiguration` trial classes ↔ the runtime          | the trial vocabulary the orchestrator instantiates             |
| Experiment configuration `unity_scene_name` ↔ the Unity scene          | the task-template filename stem                                |
| Read data-service processor ↔ its `sollertia-shared-assets` read asset | the record schema the processor emits and the dataclass stores |

---

## Decision tree: which phase to start from

```text
Is the system registered in sollertia-shared-assets (imports cleanly, appears in
list_supported_acquisition_systems_tool)?
├─ no  → Phase 1 (assets:library-extension)
└─ yes
    └─ Does every hardware module have a verified Module + ModuleInterface pair?
        ├─ no  → Phase 2 (/microcontroller-interface, ataraxis firmware-module / camera-interface, /zaber-interface)
        └─ yes
            └─ Does the sle package exist (CLI group reachable, config validates, MCP tools registered)?
                ├─ no  → Phase 3 (/acquisition-system-design → /acquisition-system-runtime)
                └─ yes
                    └─ Does a corridor task template exist for the experiment config's unity_scene_name?
                        ├─ no  → Phase 4 (assets:task-templates → unity:task-prefabs)
                        └─ yes → the system type is built; hand off to /pipeline to configure a host and
                                 run the first session (optional: external-service processors, Phase 5;
                                 agentic assets, Phase 6)
```

---

## Cross-plugin handoffs at a glance

Each row points to the system-agnostic owning skill; Mesoscope-VR is the worked example built
through these same skills.

| You need to…                                                   | Use…                                                              |
|----------------------------------------------------------------|-------------------------------------------------------------------|
| Register the system's contract in shared-assets                | `assets:library-extension`                                        |
| Add a trial class or trigger type                              | `assets:library-extension`                                        |
| Add a paired `Module` + `ModuleInterface` (catalog, type code) | `/microcontroller-interface`                                      |
| Write the firmware `Module` base mechanics                     | `ataraxis@microcontroller:firmware-module`                        |
| Write the PC `ModuleInterface` base mechanics                  | `ataraxis@communication:microcontroller-interface`                |
| Verify a microcontroller interface against hardware            | `ataraxis@communication:microcontroller-setup`                    |
| Write or verify a camera (`VideoSystem`) interface             | `ataraxis@video:camera-interface` / `ataraxis@video:camera-setup` |
| Add a Zaber motor subsystem                                    | `/zaber-interface`                                                |
| Integrate the system's primary instrument (bespoke driver)     | `/acquisition-system-design`                                      |
| Design the system's static composition                         | `/acquisition-system-design`                                      |
| Implement the runtime loop, modes, CLI, MCP module             | `/acquisition-system-runtime`                                     |
| Couple the runtime to the Unity VR task                        | `/vr-driver-interface`                                            |
| Author the corridor task template                              | `assets:task-templates`                                           |
| Generate the Unity prefab / scene from the template            | `unity:task-prefabs` / `unity:task-scenes`                        |
| Author a read / write external data-service processor          | `/google-sheets-processing`                                       |
| Register a new external read asset                             | `assets:library-extension` / `assets:data-assets`                 |
| Register a new external-service credential category            | `assets:library-extension`                                        |
| Author the per-system instance / runtime skills                | `/acquisition-system-design` (steps 10–11)                        |
| Configure a host and run the first session                     | `/pipeline`                                                       |

---

## Related skills

| Skill                                              | Relationship                                                                                 |
|----------------------------------------------------|----------------------------------------------------------------------------------------------|
| `/pipeline`                                        | The operate-time counterpart; receives the built system for its first run                    |
| `assets:library-extension`                         | Owns Phase 1 — the shared-assets contract, registries, and trial/trigger recipes             |
| `/microcontroller-interface`                       | Owns the Sollertia paired `Module` + `ModuleInterface` workflow and module catalog (Phase 2) |
| `ataraxis@microcontroller:firmware-module`         | Owns the firmware `Module` authoring in Phase 2                                              |
| `ataraxis@communication:microcontroller-interface` | Owns the `ModuleInterface` and `MicroControllerInterface` mechanics in Phase 2               |
| `ataraxis@video:camera-interface`                  | Owns the `VideoSystem` interface authoring in Phase 2                                        |
| `/zaber-interface`                                 | Owns the Zaber motor subsystem in Phase 2                                                    |
| `/acquisition-system-design`                       | Owns Phase 3 static composition and the per-package deliverables manifest                    |
| `/acquisition-system-runtime`                      | Owns Phase 3 runtime loop, modes, CLI, and MCP tool module                                   |
| `assets:task-templates`                            | Owns the corridor task template in Phase 4                                                   |
| `unity:task-prefabs`                               | Owns Unity prefab and scene generation in Phase 4                                            |
| `/google-sheets-processing`                        | Owns the external data-service processors in Phase 5                                         |
| `assets:data-assets`                               | Reads and amends the on-disk read assets a processor emits                                   |
| `/acquisition-system-setup`                        | Resolves a built system to its owning instance skill during operation                        |

---

## Verification checklist

```text
System build orchestration:
- [ ] Identified the current build phase and its handoff condition
- [ ] Confirmed the owning skill was invoked for the phase (not duplicated inline)
- [ ] Phase 1 (shared-assets) completed before any Phase 3 (sle) work began
- [ ] `import sollertia_shared_assets` succeeds and the system appears in list_supported_acquisition_systems_tool
- [ ] Every new hardware module pair is verified against hardware through the ataraxis MCP servers
- [ ] The sle CLI group is reachable and the per-system MCP tools register
- [ ] The experiment configuration's unity_scene_name resolves to a real task template and scene
- [ ] If the system integrates an external data service, its processor round-trips and (for reads) snapshots to disk
- [ ] Each cross-repository coupling has both sides in agreement, with version bumps where required
- [ ] Handed off to /pipeline for host configuration and the first session
```
