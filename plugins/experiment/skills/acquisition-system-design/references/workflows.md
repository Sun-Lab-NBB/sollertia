# Acquisition system design workflows

Step-by-step procedures for the four common changes to a Sollertia acquisition system. Loaded on demand from
`acquisition-system-design`'s SKILL.md. The layer patterns these steps reference live in `layer-patterns.md`.

---

## Adding a new hardware subsystem to an existing system

When an acquisition system gains a new category of hardware (e.g., a system that previously had only microcontrollers
gains a camera), follow these steps:

1. **Decide whether the subsystem belongs in an existing dataclass section or needs its own.** Cameras and
   microcontrollers always get their own sections. Discrete-device categories (a single MQTT broker, a single power
   supply) can fold into `<System>ExternalAssets`.

2. **Author the configuration dataclass.** Follow the Layer 2a pattern in `layer-patterns.md`. Field names follow
   `<device>_<parameter>_<unit>`, every field has a default, and every field has a docstring.

3. **Add the new section to the system configuration.** Use `field(default_factory=...)`. Bump the consumer library's
   version per Contract 2 (schema versioning) in `layer-patterns.md`.

4. **Author the binding class.** Follow the Layer 2b pattern in `layer-patterns.md`. Constructor takes the new
   configuration dataclass, and the lifecycle methods follow the conventions above.

5. **Wire the binding class into the lifecycle orchestrator.** Add the construction in the correct order (per the
   construction order in `layer-patterns.md`) and the tear-down in the reverse order.

6. **Update the per-system instance skill.** Add a section documenting the new subsystem's configuration surface and
   binding-class composition. *Worked example:* see `mesoscope:mesoscope-vr`.

7. **Regenerate the system configuration YAML.** Run the system's `sle <system> configure system` CLI command, which
   writes a default-constructed instance of the registered configuration class. The configuration-write MCP tool takes a
   complete payload instead of generating defaults, so use it to edit an existing configuration rather than to
   regenerate one. Every older deployment needs its YAML file regenerated against the new schema. For the current worked
   example's command names and options, see `mesoscope:mesoscope-vr-runtime`.

---

## Building a new acquisition system from scratch

1. **Register the system in `sollertia-shared-assets` first.** The new `AcquisitionSystems` member is only the first of
   several coupled touches, and a half-wired registry fails the import-time parity check so the package will not import.
   Hand the entire shared-assets recipe to `assets:library-extension`, which owns the enum member, the four dispatch
   registries, the per-system record dataclasses, the `from_task_template` builder, and the session-type claim a new
   session type needs. Bump that package's version.

2. **Read the seam catalog before writing any code.** `/library-extension` owns the seam map for sollertia-experiment
   and sollertia-micro-controllers, meaning the configuration registry, the shared `cross_system` primitives, the MCP
   and CLI registration seams, and the firmware module, controller target, and board family seams. It also owns the
   phase order and the gating conditions of the steps below, and the audit view of which seams a half-built system
   still misses. The autonomy boundary at the acquisition engine belongs to the same skill, under its section "The
   acquisition engine is a human-in-the-loop rewrite".

3. **Define the system's hardware composition.** List every hardware subsystem, meaning microcontrollers, cameras,
   motors, and external devices, plus the per-subsystem device count.

4. **Author the per-subsystem configuration dataclasses.** One per subsystem, in the new system's package alongside the
   system configuration class. Follow the Layer 2a pattern in `layer-patterns.md`.

5. **Author the system configuration class.** It inherits `SystemConfiguration` from `cross_system`, composes the
   configuration dataclasses, and carries a `name` field plus any auxiliary sections.

6. **Register the system and add a typed accessor.** Call
   `register_system_configuration(AcquisitionSystems.<SYSTEM>, <System>SystemConfiguration)` at import time, and expose
   a typed `get_system_configuration() -> <System>SystemConfiguration` accessor over the shared loader. The shared
   `cross_system` helpers handle create, resolve, and load with no per-system code.

7. **Author the per-subsystem binding classes.** One per subsystem, in `<system>/binding_classes.py`.

8. **Author the lifecycle orchestrator.** Typically in `<system>/system_controller.py`. This step sits on the platform's
   autonomy boundary, and only its scaffolding half carries an author-derived recipe. Scaffolding the orchestrator from
   the Mesoscope-VR package split, and composing the `cross_system` primitives that already cover the hardware families
   that the rig drives, are that recipe work, which you complete autonomously. The hardware inventory, the wiring
   topology, the per-mode semantics of the state machine, the calibration values, the safety interlocks, and the
   tear-down order have no recipe. Escalate those to the human supervisor and co-design them in a generative,
   collaborative mode. What is missing there is a hardware fact that no repository records, rather than capability, so
   the work must be human-supervised. `/library-extension` owns the full treatment of this boundary, under its section
   "The acquisition engine is a human-in-the-loop rewrite".

9. **Wire the system into the package's CLI.** Add a `<system>` command group beside the hardware-agnostic `sle get`
   group, then add one import line and one `add_command` call inside `_register_subcommands` in
   `interfaces/entry_points.py`. The CLI side has no glob-based discovery, so this edit is mandatory.

10. **Author the system's MCP tool module.** Add `interfaces/<system>_tools.py` exposing the system's configuration and
    data-management tools: configuration read, write, validate, and describe-schema, hardware and mount verification,
    and the preprocess, delete, and migrate session tools. The MCP server globs `*_tools.py` directly inside
    `interfaces/`, so the module registers itself and the filename suffix is load-bearing (the `_register_tool_modules`
    glob in `interfaces/mcp_server.py`). The hardware-agnostic `get_tools` are inherited unchanged. Without this module
    the new system is CLI-driveable and exposes no system-specific MCP surface to agents.

11. **Author the system's dedicated companion plugin.** System-specific skills do not live in the experiment plugin,
    which stays system-agnostic. They live in a dedicated companion plugin at `plugins/<system>/`. That plugin is a
    required deliverable rather than an optional one, because the pattern skills above and the assets plugin's schema
    skills carry pointers that assume the per-system skills exist. `assets:library-extension` and `/library-extension`
    between them own the full cross-repo touch list, including the plugin manifest and the marketplace entry, so work
    through them here rather than duplicating the list.

**Per-package deliverables.** A complete acquisition-system package holds:

- `<system>/system.py` for the per-subsystem configuration dataclasses, the `<System>SystemConfiguration` subclass, its
  `register_system_configuration` call, and the typed `get_system_configuration` accessor.
- `<system>/binding_classes.py` for the per-subsystem binding classes.
- `<system>/system_controller.py` for the lifecycle orchestrator.
- `<system>/data_acquisition.py` for the per-mode logic functions: one per session type the system runs, plus one per
  non-session runtime mode such as hardware maintenance.
- `<system>/acquisition_components.py` for the shared runtime-state types (trial state, log message codes) and for the
  hardware setup, tear-down, and snapshot helpers that the orchestrator and the per-mode logic functions both call.
  Keeping them here leaves `system_controller.py` holding only the state machine.
- `<system>/data_preprocessing.py` for the session-lifecycle orchestrators, meaning the preprocess, purge, and migrate
  entry points that compose the six shared `cross_system` primitives and add the system's own conversion, compression,
  and cleanup steps around them.
- `<system>/__init__.py` re-exporting the per-mode logic functions, the configuration helpers, and the three
  session-lifecycle entry points, because the CLI group and the tool module import them from the package.
- `<system>/system_health.py` for the on-demand health-report builders the system's own package owns, such as the
  filesystem-mount report that both the `check-mounts` CLI command and the mount-check MCP tool call.
- `<system>/visualizer.py`, `<system>/runtime_ui.py`, `<system>/maintenance_ui.py`, and an instrument-driver module,
  added as the system's hardware and runtime modes require.
- `interfaces/<system>.py` for the `sle <system>` CLI command group, registered in `entry_points.py`.
- `interfaces/<system>_tools.py` for the per-system MCP tool module, auto-discovered by the server.

*Worked example:* the only registered system's package instantiates this layout. See `mesoscope:mesoscope-vr`.

The shared `cross_system` package supplies the reusable building blocks these compose: the `SystemConfiguration` create,
resolve, and load lifecycle, the microcontroller `ModuleInterface` wrappers (see `/microcontroller-interface`), the
Zaber stack (see `/zaber-interface`), and the session-preprocessing primitives (see `/data-management`).

---

## Extending an existing subsystem (adding a new device to an existing dataclass)

This is the most common change. Two examples are adding a camera to the system's camera section and adding a
microcontroller-driven sensor to its microcontroller section.

1. **Add the new fields** to the existing configuration dataclass, following the field-naming convention.

2. **Extend the binding class** to instantiate the new device's wrapper and include it in the underlying controller's
   `module_interfaces` tuple (or equivalent).

3. **Update the system-specific instance skill** to document the new device's configuration field surface.

4. **Bump the consumer library's version** per Contract 2 (schema versioning) in `layer-patterns.md`.

5. **Regenerate the system configuration YAML** on every deployment.

For a microcontroller-module addition, follow the "Adding a paired Module + Interface" workflow in
`/microcontroller-interface` first. The firmware-and-wrapper pair must exist before the binding class can compose it.

---

## Authoring a custom data-service processor

When a system reads from or writes to an external request/response data service, such as a Google Sheet, a LIMS, or a
REST registry, that integration is a **data-service processor** rather than a binding class. See the "External
data-service processors" category in `subsystem-types.md`. The shipped processors `SurgeryLog` and `WaterLog`, in
`cross_system/google_sheet_tools.py`, are hard-coded to the Sollertia sheet schema, so the first decision is whether to
reuse one or author a new one.

1. **Decide between reuse and authoring.** Reuse `SurgeryLog` or `WaterLog` unchanged when the external source already
   matches the Sollertia schema, meaning the same required headers, tab layout, and identity model.
   `/google-sheets-processing` and its `references/sheet-schema-contract.md` carry that schema. Reuse means setting the
   sheet identifiers in the system configuration's external-services section and sharing the document with the service
   account, with no code. Author a new processor **only** when the schema or the service itself differs.

2. **Choose the direction.** A **read processor**, such as `SurgeryLog`, parses records into a typed platform dataclass
   that is snapshotted to disk. A **write processor**, such as `WaterLog`, consumes runtime-discovered values and writes
   them into a pre-existing record. A processor may do both.

3. **Define the schema contract.** Declare the required-header set, the structural assumptions, and the field mapping.
   The structural assumptions name which row holds the headers, where the data begins, and whether record identity is a
   column value or a tab name. Validate all of it **in the constructor**, mirroring the `_REQUIRED_*_HEADERS` check, so
   a malformed source fails loudly before any extract or update call.

4. **Implement the lifecycle contract.** The constructor takes whichever parts of the record identity the source is
   keyed by (`SurgeryLog` takes project and animal, `WaterLog` takes animal and session date), a `credentials_path`, and
   a sheet identifier or endpoint. It authenticates, builds the header-to-location map, validates, and caches the
   connection. Expose `extract_*` and `update_*` methods, and retry every API call. Expose a `close()` that releases the
   connection, and treat `__del__` as a backstop only, because the caller owning the processor closes it in a
   `try/finally` (`SurgeryLog` in `cross_system/google_sheet_tools.py`). See the processor contract in
   `/google-sheets-processing`.

5. **Place it by reuse scope.** A processor that any acquisition system could consume goes in `cross_system/`, beside
   `google_sheet_tools.py`, and is exported from `cross_system/__init__.py`. A system-specific one goes in the system's
   own package.

6. **Register a new read asset.** If a read processor emits a record type that is not an existing
   `sollertia-shared-assets` dataclass, register it as a read asset. Add the dataclass, its `ReadAssets` member, and its
   `READ_ASSET_REGISTRY` entry through the assets plugin's `assets:library-extension` ("Adding a new read asset"), and
   its read/amend surface through `assets:data-assets` before wiring the processor. A *write* processor produces no
   on-disk dataclass and needs no registry entry, because it writes runtime values directly to the external source.

7. **Wire it into preprocessing or per-session setup.** Construct the processor from the configured identifier and apply
   the platform gating policy. Skip all processing, and require no credentials, when every identifier is unset. Require
   valid credentials as soon as any identifier is set. Skip an individually unset identifier with a warning. For the
   current worked example's branch policy, see `mesoscope:mesoscope-vr-runtime`.

8. **Document it.** Update `/google-sheets-processing`, or author a sibling skill for a non-Sheets service, and update
   the consuming system's instance skill with the new processor's surface.
