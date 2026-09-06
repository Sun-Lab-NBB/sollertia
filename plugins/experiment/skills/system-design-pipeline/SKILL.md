---
name: system-design-pipeline
description: >-
  End-to-end orchestration guide for designing and building a new Sollertia acquisition system: phase ordering and
  handoff conditions across the shared-assets contract, the hardware interfaces, the acquisition runtime, and the Unity
  corridor task. Use when planning or building a new acquisition system from scratch, or deciding which design skill to
  invoke next. For operating an already-built system, see `/pipeline`.
user-invocable: false
---

# Sollertia system design pipeline

End-to-end orchestration reference for building a new Sollertia acquisition system.

This skill is the build-time counterpart to `/pipeline`. This skill builds a new acquisition system, and `/pipeline`
operates one that already exists. Mesoscope-VR is the current worked example throughout, so substitute the system you
are building wherever a `mesoscope:*` skill is named. See `mesoscope:mesoscope-vr`.

---

## Scope

**Covers:**
- Canonical phase ordering for building a new acquisition system across the four owning layers
- Handoff conditions that gate each phase before the next begins
- The cross-repository ordering hazards, meaning which repository must be complete before the next imports
- Where the assets, unity, forging, and ataraxis plugins fit into the build, and the handoff to `/pipeline`

**Does not cover:**
- The seam-by-seam catalog of what a new system touches in sollertia-experiment and sollertia-micro-controllers,
  owned by `/library-extension`
- The line-by-line authoring for any individual phase, owned by the skill each phase dispatches to
- Operating an already-built system, meaning configure, run, and preprocess, owned by `/pipeline`
- The Unity corridor scene mechanics beyond template authoring, owned by `unity:task-prefabs` and `unity:task-scenes`
- MCP server connectivity, owned by each plugin's `*-mcp-environment-setup` skill

**Handoff rules:** This skill dispatches to the owning skill at each phase. Always invoke the relevant skill for the
detailed authoring recipe, parameter reference, and troubleshooting.

---

## The build-then-operate split

A new acquisition system is built across four layers, each owned by a different repository and plugin. Building all
four is design-time, AI-assisted work: authoring against documented contracts, verifying hardware interfaces through
the ataraxis MCP servers, and wiring the system package. Once the four layers are complete, the system is *operated*
through `/pipeline`, where the Sollertia configuration-time / runtime split applies and runtime acquisition is
deterministic and AI-independent.

| Layer               | Repository                                             | Owning skill                                                 |
|---------------------|--------------------------------------------------------|--------------------------------------------------------------|
| Platform contracts  | `sollertia-shared-assets`, then the two sle/slmc seams | `assets:library-extension` → `/library-extension`            |
| Hardware interfaces | `sollertia-micro-controllers` + `sollertia-experiment` | `/microcontroller-interface` (+ ataraxis)                    |
| Acquisition runtime | `sollertia-experiment`                                 | `/acquisition-system-design` → `/acquisition-system-runtime` |
| Corridor task       | `sollertia-virtual-reality`                            | `assets:task-templates` → `unity:task-prefabs`               |

### Phase ordering is load-bearing

The repositories import one another, so the phases run in dependency order rather than in parallel.
`sollertia-experiment` imports `sollertia-shared-assets`, and `sollertia-shared-assets` runs its import-time coverage
and contract checks on a bare import, through the module-level `_assert_registry_coverage()`,
`_assert_descriptor_contract()`, and `_assert_experiment_configuration_contract()` calls at the bottom of
`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`. A half-wired shared-assets contract makes
`import sollertia_shared_assets` raise, which also breaks every `sle` entry point. Complete and verify each phase's
handoff condition before starting the next.

---

## Build phases

```text
1a. Shared-assets contract   assets:library-extension  (+ the shared-assets README recipes)
1b. Library seams            /library-extension        (sollertia-experiment + sollertia-micro-controllers)
2.  Hardware interfaces      /microcontroller-interface, ataraxis firmware-module / camera-interface, /zaber-interface
3.  Acquisition runtime      /acquisition-system-design → /acquisition-system-runtime
4.  Corridor task            assets:task-templates → unity:task-generator → unity:task-prefabs (+ zone-prefabs)
5.  External services (opt.) /google-sheets-processing  (read/write processors, read asset via assets:library-extension)
6.  Agentic assets           /acquisition-system-design (the system's companion plugin and its skills)
7.  External tool bindings   /acquisition-system-design (separate-environment CLI tools, optional)
8.  Downstream design (opt.) forging:data-processing-design
→   Operate                  /pipeline  (configure a host and run the first session)
```

### Phase 1: Platform contracts

This phase runs in two ordered halves. Half 1 registers the system with sollertia-shared-assets, and half 2 maps the
sollertia-experiment and sollertia-micro-controllers seams the later phases fill in. Every later phase depends on
half 2, because half 2 names the files each of them touches.

#### Half 1: the shared-assets contract

- **Plugin / Skill:** `assets:library-extension`, which defers the line-by-line recipe to the
  `sollertia-shared-assets` README ("Adding New Acquisition Systems", plus "Adding a New Trial Class",
  "Adding a New Trigger Type", and "Adding New Session Types" as the system needs them).
- **Actions:** Append the `AcquisitionSystems` member. Author the `<system>/` subpackage holding the
  `<System>HardwareState`, the `<System>ExperimentConfiguration` with its own trial classes and `from_task_template`
  builder, the `<System>RawData`, and the per-session-type descriptors. Register each in `HARDWARE_STATE_REGISTRY`,
  `EXPERIMENT_CONFIGURATION_REGISTRY`, `SYSTEM_RAW_DATA_REGISTRY`, `DESCRIPTOR_REGISTRY`, and `SYSTEM_SESSION_TYPES`
  (`registries.py`). Re-export the classes from the top-level package, then bump the `sollertia-shared-assets` version.
- **Ordering hazard:** This half is first and must fully complete. Because `sle` imports `sollertia-shared-assets`, a
  half-wired registry breaks both packages' imports.
- **Gate on both of these before starting half 2:** `python -c "import sollertia_shared_assets"` succeeds, meaning the
  import-time `_assert_registry_coverage`, `_assert_descriptor_contract`, and
  `_assert_experiment_configuration_contract` checks all pass (`registries.py`), and the new system appears in
  `list_supported_acquisition_systems_tool`.

#### Half 2: the sollertia-experiment and sollertia-micro-controllers seams

- **Plugin / Skill:** `/library-extension`, which catalogues the configuration-registry seam, the shared
  `cross_system` primitives, the MCP and CLI registration seams, and the slmc module, target, and board seams.
- **Actions:** Walk the seam catalog and record, per seam, whether the new system composes the existing primitive or
  authors its own. The catalog is the input to Phases 2 through 4, because it names the exact file and symbol each
  later phase edits.
- **Handoff condition:** Every seam in the catalog is marked either "reuse as-is" or "author in Phase N", with no
  seam left undecided.

### Phase 2: Hardware interfaces

- **Plugin / Skill:** `/microcontroller-interface` owns the Sollertia paired-module stack, meaning the module catalog,
  type-code allocation, and the "Adding a paired Module + Interface" workflow. It delegates the per-side base
  mechanics to `microcontroller:firmware-module` (ataraxis marketplace, the firmware `Module`) and
  `communication:microcontroller-interface` (the PC-side `ModuleInterface`). Cameras use `video:camera-interface`, and
  motors use `/zaber-interface`.
- **Actions:** For each hardware module that the system drives and that the shared module catalog does not already
  provide, author the paired C++ `Module` and Python `ModuleInterface`, allocating a type code and instance ids from
  `/microcontroller-interface`'s catalog. Hardware already in the catalog is reused with no new code. The firmware and
  PC sides must agree on the protocol, meaning the type code, the `PACKED_STRUCT` command and parameter layouts, and the
  event codes.
- **System instrument:** Beyond the ataraxis-contracted hardware above, the system's primary scientific instrument is
  integrated through a bespoke, system-specific driver, because each instrument differs and no shared contract exists.
  The current worked example drives its instrument through a custom bridge. See `mesoscope:mesoscope-vr`. Author the
  instrument's driver as a Layer-2 subsystem in Phase 3 through `/acquisition-system-design`.
- **Cross-plugin handoffs:**
  - `communication:microcontroller-setup` to discover controllers and verify MQTT
  - `video:camera-setup` to discover and exercise cameras
- **Handoff condition:** Every new `Module` + `ModuleInterface` pair round-trips a `set_parameters` / `send_command`
  call against connected hardware from a Python REPL, per `/microcontroller-interface`'s verification checklist. The
  ataraxis MCP servers confirm only that the controller enumerates (`list_microcontrollers_tool`) and that the MQTT
  broker answers (`check_mqtt_broker_tool`), and neither carries a command-send tool. Reused catalog modules need no
  new code.

### Phase 3: Acquisition runtime

- **Plugin / Skill:** `/acquisition-system-design` for the static composition, then `/acquisition-system-runtime` for
  the runtime loop. Follow the "Building a new acquisition system from scratch" workflow and the per-package
  deliverables manifest in `/acquisition-system-design`'s `references/workflows.md`.
- **This is the human-in-the-loop phase of the build.** Every other phase is agent-ownable glue against a documented
  contract, and this one is co-designed with the human supervisor. An acquisition engine is written per system rather
  than derived from a common runtime superclass, because it is defined by a physical hardware inventory and by
  lab-local wiring conventions (the "Extending the Platform" section of `sollertia-experiment/README.md`). Scaffold the
  controller from the Mesoscope-VR package split and compose the `cross_system` primitives. Settle the hardware
  inventory, the wiring topology, the per-mode semantics of the state machine, the calibration values, the safety
  interlocks, and the teardown ordering with the human supervisor. `/library-extension` owns the full treatment of this
  boundary, under "The acquisition engine is a human-in-the-loop rewrite".
- **Actions:** Author the system package: the per-subsystem configuration dataclasses, the
  `<System>SystemConfiguration` subclass with its `register_system_configuration` call and typed
  `get_system_configuration` accessor, the binding classes, the lifecycle orchestrator, the per-mode logic functions,
  and the visualizer, UIs, and instrument driver the system needs. The orchestrator composes the platform-general VR
  task driver that couples the runtime to the Unity scene, covered by `/vr-driver-interface` alongside
  `/microcontroller-interface` and `/zaber-interface`. Add the `sle <system>` CLI command group, registered in
  `interfaces/entry_points.py`, and the `interfaces/<system>_tools.py` MCP tool module. Bump the `sollertia-experiment`
  version.
- **Handoff condition:** The `sle <system>` CLI group is reachable, the system's configuration validation tool reports
  the configuration as valid with an empty `issues` list, and the per-system MCP tools register, because the server
  imports every `*_tools.py` module in `interfaces/` through `_register_tool_modules()` (`interfaces/mcp_server.py`).

### Phase 4: Corridor task

- **Plugin / Skill:** `assets:task-templates` authors the `TaskTemplate`, `unity:task-generator` owns the editor
  pipeline that turns it into a prefab, `unity:task-prefabs` creates and inspects the task, and `unity:zone-prefabs`
  manufactures the trigger zone prefabs the trial structures reference. `unity:unity-tests` runs the Unity Test
  Framework suite over the result.
- **Actions:** Author a corridor task template for the system. Every Sollertia acquisition system runs the linear
  infinite corridor, as the `AcquisitionSystems` docstring in `enums.py` states, so a new system reuses the existing
  corridor, which makes this template authoring rather than new scene engineering. The experiment configuration's
  `unity_scene_name` must match the task template's filename stem, because `SessionData.create()` resolves the template
  from the task templates directory by that stem and raises `FileNotFoundError` when it is absent
  (`data_hierarchy/session_data.py`).
- **Handoff condition:** The experiment configuration's `unity_scene_name` resolves to a real task template and Unity
  scene, and the Unity test suite passes.

### Phase 5: External data-service processors (optional)

- **Plugin / Skill:** `/google-sheets-processing` owns the external data-service processors, meaning the `SurgeryLog`
  and `WaterLog` Google Sheets classes, their schema contract, and authoring a custom one.
- **Actions:** If the system reads from or writes to an external request/response data service, such as a Google
  Sheet, a LIMS, or a registry, decide reuse versus author. Reuse `SurgeryLog` and `WaterLog` when the source matches
  the Sollertia schema, and author a new processor when the schema or the service differs. A read processor parses
  records into a typed on-disk dataclass, and a write processor writes runtime-discovered values back. A reusable
  processor lives in `cross_system/`, and a system-specific one lives in the system package. Wire it into
  preprocessing, gated on the configured identifiers.
- **Read-asset coupling:** If a read processor emits a record type that is not an existing `sollertia-shared-assets`
  read asset, register the new asset through `assets:library-extension` ("Adding a New Read Asset"). Register its read
  and amend surface through `assets:data-assets`, as Phase 1 contract work, before wiring the processor.
- **Credentials:** A request/response service authenticates with a credentials file managed by
  `sollertia-shared-assets`. `CredentialsTypes` currently declares one category, `google`, for Google Sheets
  (`enums.py`). A service needing a new category registers a `CredentialsTypes` member and a matching
  `CREDENTIALS_FILE_REGISTRY` entry, which is a maintainer-curated contract decision in Phase 1. The credential file
  itself is configured per host at operate time, through `/pipeline` and `assets:working-directory`.
- **Handoff condition:** The processor authenticates and round-trips against the external source, and a read
  processor's records validate and snapshot to disk.

### Phase 6: Agentic assets

- **Plugin / Skill:** `/acquisition-system-design`, whose "Author the system's dedicated companion plugin" step owns
  the per-system skill authoring recipe.
- **Actions:** Author the system's companion plugin at `plugins/<system>/`, holding a per-system instance skill and,
  when the system has non-trivial runtime modes, a per-system runtime skill. Model both on the current worked
  example's pair, `mesoscope:mesoscope-vr` and `mesoscope:mesoscope-vr-runtime`. The companion plugin is a required
  deliverable, because the experiment plugin stays system-agnostic and its pattern skills carry pointers that assume
  the per-system skills exist.
- **Handoff condition:** The system has an instance skill to which the operate pipeline can dispatch.

### Phase 7: External tool bindings (optional)

- **Plugin / Skill:** `/external-tool-bindings` owns the binding convention, the admission test that decides whether
  the tool registers or binds, the artifact contract, and the ordered workflow for adding a binding.
  `/acquisition-system-design` owns the acquisition-system configuration layer that hosts the tool's configuration
  section.
- **Actions:** Run the admission test in `/external-tool-bindings` before writing any binding code. When the system
  runs a tool whose dependencies conflict with the acquisition stack, keep that tool in its own environment and launch
  it from preprocessing as a subprocess. Add a configuration section holding the environment name, the tool's project
  path, and its runtime parameters. Gate the launch on the session types that acquire the accompanying data, and join
  the subprocess before the transfer to long-term storage, so a failure retains the local session copy. What a given
  system launches, and with which flags, belongs to that system. See `mesoscope:mesoscope-vr`.
- **Manual host configuration:** An environment name and a tool project path are host-specific, and the system's
  configuration-validation and mount-check tools cannot confirm an environment name. You MUST set and confirm both by
  hand on every host that runs the tool.
- **Handoff condition:** A preprocessing run over a session that triggers the tool writes the tool's output files
  beside the source data and completes the transfer to long-term storage.

### Phase 8: Downstream processing design (optional)

- **Plugin / Skill:** `forging:data-processing-design`, which owns the agnostic-versus-per-system processing pattern.
- **Actions:** Decide which of the new system's raw data streams the agnostic forging primitives already parse and
  which need per-system parsers, then register the per-system entries the forging registries require before the
  system's sessions are processed.
- **Handoff condition:** A recorded session from the new system processes end to end through the forging plugin.

### Operate: hand off to `/pipeline`

Once the four build layers pass their handoff conditions, the system type exists. Hand off to `/pipeline` to configure
a host for the new system, bring up its hardware, author an experiment, and run the first session.

---

## Cross-repository couplings

Each coupling spans two repositories, so a change on one side requires the matching change on the other before the
system imports or runs.

| Coupling                                                               | Both sides must agree on                                       |
|------------------------------------------------------------------------|----------------------------------------------------------------|
| `AcquisitionSystems` member ↔ the `sle` runtime + CLI + MCP module     | the system identifier, slsa is registered before sle imports   |
| Firmware `Module` ↔ PC `ModuleInterface`                               | the type code, command / parameter layout, and event codes     |
| `<System>ExperimentConfiguration` trial classes ↔ the runtime          | the trial vocabulary the orchestrator instantiates             |
| Experiment configuration `unity_scene_name` ↔ the Unity scene          | the task-template filename stem                                |
| Read data-service processor ↔ its `sollertia-shared-assets` read asset | the record schema the processor emits and the dataclass stores |
| A subprocess tool's configuration fields ↔ that tool's CLI flags       | the flag names and value formats passed to the subprocess      |

### The two registration seams are asymmetric

The MCP seam and the CLI seam register a new system's `interfaces/` code in opposite ways, and an agent that assumes
symmetry silently ships a system whose CLI group never appears.

- **MCP tools register automatically.** `_register_tool_modules` globs `*_tools.py` inside `interfaces/` and imports
  each match, so a new `interfaces/<system>_tools.py` self-registers through its `@mcp.tool()` decorators with no edit
  to an existing file (`interfaces/mcp_server.py`).
- **The CLI group registers by hand.** `_register_subcommands` carries one hardcoded import and one `add_command` call
  per group, so a third group requires editing that function (`interfaces/entry_points.py`).

`/library-extension` owns both seams, along with the rest of the sollertia-experiment and sollertia-micro-controllers
seam catalog.

---

## Decision tree: which phase to start from

```text
Is the system registered in sollertia-shared-assets (imports cleanly, appears in
list_supported_acquisition_systems_tool)?
├─ no  → Phase 1 half 1 (assets:library-extension)
└─ yes
    └─ Is the sle / slmc seam catalog walked and every seam decided?
        ├─ no  → Phase 1 half 2 (/library-extension)
        └─ yes
            └─ Does every hardware module have a verified Module + ModuleInterface pair?
                ├─ no  → Phase 2 (/microcontroller-interface, ataraxis firmware-module / camera-interface)
                └─ yes
                    └─ Does the sle package exist (CLI group reachable, config validates, MCP tools registered)?
                        ├─ no  → Phase 3 (/acquisition-system-design → /acquisition-system-runtime)
                        └─ yes
                            └─ Does a corridor task exist for the experiment config's unity_scene_name?
                                ├─ no  → Phase 4 (assets:task-templates → the unity task skills)
                                └─ yes → the system type is built. Hand off to /pipeline to configure a host and
                                         run the first session (also: external-service processors, Phase 5,
                                         agentic assets, Phase 6, external tool bindings, Phase 7, downstream
                                         processing design, Phase 8)
```

---

## Cross-plugin handoffs at a glance

Each row points to the system-agnostic owning skill. Mesoscope-VR is the worked example built through these same skills.

| You need to…                                                   | Use…                                              |
|----------------------------------------------------------------|---------------------------------------------------|
| Register the system's contract in shared-assets                | `assets:library-extension`                        |
| Add a trial class or trigger type                              | `assets:library-extension`                        |
| Map the sollertia-experiment and slmc seams before building    | `/library-extension`                              |
| Add a paired `Module` + `ModuleInterface` (catalog, type code) | `/microcontroller-interface`                      |
| Write the firmware `Module` base mechanics                     | `microcontroller:firmware-module`                 |
| Write the PC `ModuleInterface` base mechanics                  | `communication:microcontroller-interface`         |
| Verify a microcontroller interface against hardware            | `communication:microcontroller-setup`             |
| Write or verify a camera (`VideoSystem`) interface             | `video:camera-interface` / `video:camera-setup`   |
| Add a Zaber motor subsystem                                    | `/zaber-interface`                                |
| Integrate the system's primary instrument (bespoke driver)     | `/acquisition-system-design`                      |
| Design the system's static composition                         | `/acquisition-system-design`                      |
| Implement the runtime loop, modes, and CLI                     | `/acquisition-system-runtime`                     |
| Author the per-system `interfaces/<system>_tools.py` module    | `/acquisition-system-design`                      |
| Couple the runtime to the Unity VR task                        | `/vr-driver-interface`                            |
| Author the corridor task template                              | `assets:task-templates`                           |
| Run the editor pipeline that generates the task prefab         | `unity:task-generator`                            |
| Create, delete, or inspect a Unity task                        | `unity:task-prefabs`                              |
| Manufacture a trigger zone prefab                              | `unity:zone-prefabs`                              |
| Run the Unity Test Framework suite                             | `unity:unity-tests`                               |
| Author a read / write external data-service processor          | `/google-sheets-processing`                       |
| Register a new external read asset                             | `assets:library-extension` / `assets:data-assets` |
| Register a new external-service credential category            | `assets:library-extension`                        |
| Author the per-system instance / runtime skills                | `/acquisition-system-design`                      |
| Decide whether an outside tool registers or binds, and bind it | `/external-tool-bindings`                         |
| Design the downstream processing for the new system            | `forging:data-processing-design`                  |
| Configure a host and run the first session                     | `/pipeline`                                       |

---

## Related skills

The `microcontroller:`, `communication:`, and `video:` entries below resolve through the ataraxis marketplace. Every
other entry resolves inside the sollertia marketplace.

| Skill                                     | Relationship                                                                                  |
|-------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/cli-reference`                          | Reference: the `sle` surface a new acquisition system extends                                 |
| `/pipeline`                               | The operate-time counterpart, receives the built system for its first run                     |
| `assets:library-extension`                | Owns Phase 1 half 1, the shared-assets contract, registries, and trial/trigger recipes        |
| `/library-extension`                      | Owns Phase 1 half 2, the sollertia-experiment and sollertia-micro-controllers seam catalog    |
| `/microcontroller-interface`              | Owns the Sollertia paired `Module` + `ModuleInterface` workflow and module catalog in Phase 2 |
| `microcontroller:firmware-module`         | Owns the firmware `Module` authoring in Phase 2                                               |
| `communication:microcontroller-interface` | Owns the `ModuleInterface` and `MicroControllerInterface` mechanics in Phase 2                |
| `video:camera-interface`                  | Owns the `VideoSystem` interface authoring in Phase 2                                         |
| `/zaber-interface`                        | Owns the Zaber motor subsystem in Phase 2                                                     |
| `/acquisition-system-design`              | Owns Phase 3 static composition, the deliverables manifest, and the MCP tool module           |
| `/acquisition-system-runtime`             | Owns Phase 3 runtime loop, modes, and the CLI surface pattern                                 |
| `/vr-driver-interface`                    | Owns the host-side VR task driver that couples the runtime to the Unity scene in Phase 3      |
| `assets:task-templates`                   | Owns the corridor task template in Phase 4                                                    |
| `unity:task-generator`                    | Owns the editor pipeline that generates the task prefab in Phase 4                            |
| `unity:task-prefabs`                      | Owns Unity task creation and prefab inspection in Phase 4                                     |
| `unity:zone-prefabs`                      | Owns trigger zone prefab manufacture in Phase 4                                               |
| `unity:unity-tests`                       | Owns the Unity Test Framework suite that verifies Phase 4                                     |
| `/google-sheets-processing`               | Owns the external data-service processors in Phase 5                                          |
| `assets:data-assets`                      | Reads and amends the on-disk read assets a processor emits                                    |
| `/external-tool-bindings`                 | Owns the register-versus-bind admission test and the binding workflow in Phase 7              |
| `forging:data-processing-design`          | Owns the downstream agnostic-versus-per-system processing design in Phase 8                   |
| `/acquisition-system-setup`               | Resolves a built system to its owning instance skill during operation                         |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

System build orchestration:
- [ ] Identified the current build phase and its handoff condition
- [ ] Confirmed the owning skill was invoked for the phase, not duplicated inline
- [ ] Phase 1 half 1 completed before half 2, and half 2 before any Phase 3 work began
- [ ] `import sollertia_shared_assets` succeeds and the system appears in list_supported_acquisition_systems_tool
- [ ] Every seam in the /library-extension catalog is marked reuse-as-is or author-in-Phase-N
- [ ] Phase 3's hardware-defined decisions were settled with the human supervisor rather than inferred
- [ ] Every new hardware module pair enumerates through communication:microcontroller-setup after flashing
- [ ] Every new hardware module pair round-trips a set_parameters / send_command call from a Python REPL
- [ ] The sle CLI group was registered by hand in entry_points.py, since only the MCP seam is automatic
- [ ] The experiment configuration's unity_scene_name resolves to a real task template and scene
- [ ] If the system integrates an external data service, its processor round-trips and (for reads) snapshots to disk
- [ ] If the system binds an external tool, its environment and project path are set and confirmed by hand
- [ ] Each cross-repository coupling has both sides in agreement, with version bumps where required
- [ ] Handed off to /pipeline for host configuration and the first session
```
