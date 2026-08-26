# Acquisition system design workflows

Step-by-step procedures for the three common changes to a Sollertia acquisition system. Loaded on
demand from `acquisition-system-design`'s SKILL.md. The layer patterns these steps reference live in
`layer-patterns.md`.

---

## Adding a new hardware subsystem to an existing system

When an acquisition system gains a new category of hardware (e.g., a system that previously had
only microcontrollers gains a camera), follow these steps:

1. **Decide whether the subsystem belongs in an existing dataclass section or needs its own.** Cameras
   and microcontrollers always get their own sections. Discrete-device categories (a single MQTT
   broker, a single power supply) can fold into `<System>ExternalAssets`.

2. **Author the configuration dataclass.** Follow the Layer 2a pattern in
   `layer-patterns.md`. Field names
   follow `<device>_<parameter>_<unit>`; every field has a default; every field has a docstring.

3. **Add the new section to the system configuration.** Use `field(default_factory=...)`. Bump the
   consumer library's version per Contract 2 (schema versioning) in
   `layer-patterns.md`.

4. **Author the binding class.** Follow the Layer 2b pattern in
   `layer-patterns.md`. Constructor takes the
   new configuration dataclass; lifecycle methods follow the conventions above.

5. **Wire the binding class into the lifecycle orchestrator.** Add the construction in the correct
   order (per the construction order in
   `layer-patterns.md`) and the teardown in the reverse order.

6. **Update the per-system instance skill.** Add a section documenting the new subsystem's configuration
   surface and binding-class composition (for Mesoscope-VR, this is `mesoscope:mesoscope-vr`).

7. **Regenerate the system configuration YAML.** Use the system's configuration tooling (e.g., the
   `write_system_configuration_tool` MCP tool, or the `sle mesoscope configure` CLI for Mesoscope-VR)
   to write a new configuration file from the updated defaults. Older deployments need their YAML
   files re-generated against the new schema.

---

## Building a new acquisition system from scratch

1. **Define the system's hardware composition.** List every hardware subsystem (microcontrollers,
   cameras, motors, external devices) and the per-subsystem device count.

2. **Register the system on the `sollertia-shared-assets` side.** Adding the
   `sollertia_shared_assets.AcquisitionSystems` enum value is only the first of several coupled
   touches — the new system also needs its `<System>HardwareState`, `<System>ExperimentConfiguration`,
   and `<System>RawData` dataclasses, the matching `HARDWARE_STATE_REGISTRY`,
   `EXPERIMENT_CONFIGURATION_REGISTRY`, and `SYSTEM_RAW_DATA_REGISTRY` entries, and the
   `from_task_template` builder on the experiment-configuration dataclass (enforced by
   `_assert_experiment_configuration_contract`), all guarded by the import-time
   `_assert_registry_coverage()` parity check. Hand the entire slsa-side recipe off to the assets
   plugin's `assets:library-extension` ("Adding a new `AcquisitionSystems` member") and bump that package's
   version. Do not stop at the enum value — a half-wired registry fails the parity check and the package
   will not import.

3. **Author the per-subsystem configuration dataclasses.** One per subsystem, in the new system's package
   alongside the system configuration class (Mesoscope-VR for example defines both in `mesoscope_vr/system.py`).

4. **Author the system configuration class.** Inherits `SystemConfiguration` (from `cross_system`),
   composes the configuration dataclasses, includes a `name` field and any auxiliary sections
   (filesystem, sheets, etc.).

5. **Register the system and add a typed accessor.** Register the `SystemConfiguration` subclass at
   import time via `register_system_configuration(AcquisitionSystems.<SYSTEM>, <System>SystemConfiguration)`,
   and expose a typed `get_system_configuration() -> <System>SystemConfiguration` accessor over the
   shared cross-system loader. The shared `cross_system` helpers handle create / resolve / load.

6. **Author the per-subsystem binding classes.** One per subsystem, in `<system>/binding_classes.py`.

7. **Author the lifecycle orchestrator.** Typically, in `<system>/system_controller.py`.

8. **Wire the system into the package's CLI.** Add a `<system>` command group to the `sle` CLI (e.g.,
   `sle mesoscope`), alongside the hardware-agnostic `sle get` group, and register it in
   `interfaces/entry_points.py`'s `_register_subcommands`.

9. **Author the system's MCP tool module.** Add `interfaces/<system>_tools.py` exposing the system's
   configuration and data-management tools — system-configuration read / write / validate /
   describe-schema, hardware and mount verification, and the preprocess / delete / migrate session
   tools — mirroring `mesoscope_vr_tools.py`. The MCP server discovers every `*_tools.py` module by
   filename suffix, so the module registers automatically; the hardware-agnostic `get_tools` are
   inherited unchanged. Without this module the new system is CLI-driveable but exposes no
   system-specific MCP surface to agents.

10. **Author the system's dedicated companion plugin.** System-specific skills do not live in the experiment
    plugin, which stays system-agnostic. They live in a dedicated companion plugin at `plugins/<system>/`,
    mirroring `plugins/mesoscope/`. That plugin is a required deliverable rather than an optional one: the
    pattern skills above and the assets plugin's schema skills carry pointers that assume the per-system skills
    exist, so a system without them is driveable yet undocumented for agents. Author three things:

    - `plugins/<system>/skills/<system>/SKILL.md` — the per-system instance skill, documenting the system's
      hardware subsystems, configuration field surface, binding-class composition, and lifecycle. Follow the
      structure of `mesoscope:mesoscope-vr`.
    - `plugins/<system>/.claude-plugin/plugin.json` — the plugin manifest, carrying `name` (the `<system>`
      token), `version`, a `description` naming the layers the plugin covers and the MCP servers it relies on,
      `repository`, `license`, and `"skills": "./skills/"`. A companion plugin declares no `mcpServers` block
      of its own, and it consumes the servers the core plugins declare.
    - An entry in the marketplace manifest at `.claude-plugin/marketplace.json`, appended to its `plugins`
      array with `name`, `source` (`./plugins/<system>`), and a `description`. Without that entry the plugin
      is not installable from the marketplace.

    `assets:library-extension` owns the full cross-repo touch list for a new acquisition system and names these
    same deliverables. Work through it alongside this step so no coupled touch is missed.

11. **Author a per-system runtime skill** in the same companion plugin when the system has non-trivial
    runtime modes / state machines / training behaviors. Follow the structure of
    `mesoscope:mesoscope-vr-runtime`.

**Per-package deliverables.** A complete acquisition-system package mirrors the Mesoscope-VR layout:

- `<system>/system.py` — the per-subsystem configuration dataclasses, the `<System>SystemConfiguration`
  subclass, its `register_system_configuration` call, and the typed `get_system_configuration` accessor.
- `<system>/binding_classes.py` — the per-subsystem binding classes.
- `<system>/system_controller.py` — the lifecycle orchestrator.
- `<system>/data_acquisition.py` — the per-mode logic functions (one per session type the system runs).
- `<system>/visualizer.py`, `<system>/runtime_ui.py`, `<system>/maintenance_ui.py`, and an instrument-driver
  module — added as the system's hardware and runtime modes require.
- `interfaces/<system>.py` — the `sle <system>` CLI command group (registered in `entry_points.py`).
- `interfaces/<system>_tools.py` — the per-system MCP tool module (auto-discovered by the server).

The shared `cross_system` package supplies the reusable building blocks these compose: the
`SystemConfiguration` create / resolve / load lifecycle, the microcontroller `ModuleInterface` wrappers (see
`/microcontroller-interface`), the Zaber stack (see `/zaber-interface`), and the
session-preprocessing primitives (see `/data-management`).

---

## Extending an existing subsystem (adding a new device to an existing dataclass)

This is the most common change. Examples: adding a new camera to `MesoscopeCameras`, adding a new
microcontroller-driven sensor to `MesoscopeMicroControllers`.

1. **Add the new fields** to the existing configuration dataclass, following the field-naming
   convention.

2. **Extend the binding class** to instantiate the new device's wrapper and include it in the
   underlying controller's `module_interfaces` tuple (or equivalent).

3. **Update the system-specific instance skill** to document the new device's configuration field
   surface.

4. **Bump the consumer library's version** per Contract 2 (schema versioning) in
   `layer-patterns.md`.

5. **Regenerate the system configuration YAML** on every deployment.

For microcontroller-module additions, also follow `/microcontroller-interface`'s
"Adding a paired Module + Interface" workflow first — the firmware-and-wrapper pair must exist
before the binding class can compose it.

---

## Authoring a custom data-service processor

When a system reads from or writes to an external request/response data service (a Google Sheet, a
LIMS, a REST registry), the integration is a **data-service processor**, not a binding class — see
the "External data-service processors" category in `subsystem-types.md`. The
shipped processors (`SurgeryLog`, `WaterLog` in `cross_system/google_sheet_tools.py`) are hard-coded
to the Sollertia sheet schema, so reuse-vs-author is the first decision.

1. **Decide reuse vs. author.** If the external source already matches the Sollertia schema (same
   required headers, tab layout, and identity model — see
   `/google-sheets-processing` → references/sheet-schema-contract.md), reuse `SurgeryLog` /
   `WaterLog` as-is: set the sheet identifiers in the system configuration's external-services section
   and share the document with the service account. No code. Author a new processor **only** when the
   schema or the service itself differs.

2. **Choose the direction.** A **read processor** (like `SurgeryLog`) parses records into a typed
   platform dataclass that gets snapshotted to disk. A **write processor** (like `WaterLog`) consumes
   runtime-discovered values and writes them into a pre-existing record. A processor may do both.

3. **Define the schema contract.** Declare the required-header (or required-field) set, the structural
   assumptions (which row holds headers, whether record identity is a column value or a tab/record
   name, where data begins), and the field mapping. Validate all of it **in the constructor** so a
   malformed source fails loudly before any extract/update — mirror the `_REQUIRED_*_HEADERS` check.

4. **Implement the lifecycle contract.** Constructor takes `(identity…, credentials_path,
   sheet_id-or-endpoint)`, authenticates (service account where applicable), builds the
   `header → location` map, validates, and caches the connection. Expose `extract_*` / `update_*`
   methods. Retry every API call. Close the connection in `__del__`. (See the processing-asset
   contract in `/google-sheets-processing`.)

5. **Place it by reuse scope.** A processor that any acquisition system could consume goes in
   `cross_system/` (like `google_sheet_tools.py`) and is exported from `cross_system/__init__.py`; a
   system-specific one goes in the system's own package.

6. **Register a new read asset.** If a read processor emits a record type that is not an existing
   `sollertia-shared-assets` dataclass, register it as a read asset. Add the dataclass, its
   `ReadAssets` member, and its `READ_ASSET_REGISTRY` entry through the assets plugin's
   `assets:library-extension` ("Adding a new read asset"), and its read/amend surface through
   `assets:data-assets` before wiring the processor. A *write* processor produces no on-disk dataclass
   and needs no registry entry — it writes runtime values directly to the external source.

7. **Wire it into preprocessing / per-session setup.** Construct the processor from the configured
   identifier, gating exactly as `_preprocess_google_sheet_data` does: skip all processing (and
   require no credentials) when every identifier is unset; require credentials when any is set; skip an
   individually-unset identifier with a warning.

8. **Document it.** Update `/google-sheets-processing` (or author a sibling skill for a
   non-Sheets service) and the consuming system's instance skill with the new processor's surface.
