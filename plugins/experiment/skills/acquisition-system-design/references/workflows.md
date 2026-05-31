# Acquisition system design workflows

Step-by-step procedures for the three common changes to a Sollertia acquisition system. Loaded on
demand from `acquisition-system-design`'s SKILL.md. The layer patterns these steps reference live in
[layer-patterns.md](layer-patterns.md).

---

## Adding a new hardware lane to an existing system

When an acquisition system gains a new category of hardware (e.g., a system that previously had
only microcontrollers gains a camera), follow these steps:

1. **Decide whether the lane belongs in an existing dataclass section or needs its own.** Cameras
   and microcontrollers always get their own sections. Discrete-device categories (a single MQTT
   broker, a single power supply) can fold into `<System>ExternalAssets`.

2. **Author the calibration dataclass.** Follow the Layer 2a pattern in
   [layer-patterns.md](layer-patterns.md#layer-2a-per-lane-calibration-dataclasses). Field names
   follow `<device>_<parameter>_<unit>`; every field has a default; every field has a docstring.

3. **Add the new section to the system configuration.** Use `field(default_factory=...)`. Bump the
   consumer library's version per Contract 2 (schema versioning) in
   [layer-patterns.md](layer-patterns.md#contract-2-schema-versioning).

4. **Author the binding class.** Follow the Layer 2b pattern in
   [layer-patterns.md](layer-patterns.md#layer-2b-per-lane-binding-classes). Constructor takes the
   new calibration dataclass; lifecycle methods follow the conventions above.

5. **Wire the binding class into the lifecycle orchestrator.** Add the construction in the correct
   order (per the construction order in
   [layer-patterns.md](layer-patterns.md#construction-order)) and the teardown in the reverse order.

6. **Update the per-system instance skill.** Add a section documenting the new lane's calibration
   surface and binding-class composition (for Mesoscope-VR, this is `experiment:mesoscope-vr`).

7. **Regenerate the system configuration YAML.** Use the system's configuration tooling (e.g., the
   `write_system_configuration_tool` MCP tool, or the `sle mesoscope configure` CLI for Mesoscope-VR)
   to write a new configuration file from the updated defaults. Older deployments need their YAML
   files re-generated against the new schema.

---

## Building a new acquisition system from scratch

1. **Define the system's hardware composition.** List every hardware lane (microcontrollers,
   cameras, motors, external devices) and the per-lane device count.

2. **Register the system on the `sollertia-shared-assets` side.** Adding the
   `sollertia_shared_assets.AcquisitionSystems` enum value is only the first of several coupled
   touches — the new system also needs its `<System>HardwareState`, `<System>ExperimentConfiguration`,
   and `<System>RawData` dataclasses, the matching `HARDWARE_STATE_REGISTRY`,
   `EXPERIMENT_CONFIGURATION_REGISTRY`, and `SYSTEM_RAW_DATA_REGISTRY` entries, and an experiment-config
   factory, all guarded by the import-time `_assert_registry_coverage()` parity check. Hand the entire
   slsa-side recipe off to the assets plugin's `/library-extension` ("Adding a new `AcquisitionSystems`
   member") and bump that package's version. Do not stop at the enum value — a half-wired registry
   fails the parity check and the package will not import.

3. **Author the per-lane calibration dataclasses.** One per lane, in the new system's package
   (typically `<system>/configuration.py`).

4. **Author the system configuration class.** Inherits `YamlConfig`, composes the calibration
   dataclasses, includes a `name` field and any auxiliary sections (filesystem, sheets, etc.).

5. **Author the module-level helpers.** `create_system_configuration_file`,
   `get_system_configuration_path`, `get_system_configuration`.

6. **Author the per-lane binding classes.** One per lane, in `<system>/binding_classes.py`.

7. **Author the lifecycle orchestrator.** Typically, in `<system>/system_controller.py`.

8. **Wire the system into the package's CLI.** Add a `<system>` command group to the `sle` CLI (e.g.,
   `sle mesoscope`), alongside the hardware-agnostic `sle get` group.

9. **(Optional but recommended) Author dedicated agentic assets for the new system.** A new
   acquisition system optionally benefits from its own per-system instance skill in this plugin,
   documenting the system's hardware lanes, calibration field surface, binding-class composition, and
   lifecycle. Follow the structure of `experiment:mesoscope-vr`. The system runs without it, but
   omitting it leaves the system driveable yet undocumented for agents (and the pattern skills above
   keep pointing at Mesoscope-VR as the sole worked instance).

10. **(Optional but recommended) Author a per-system runtime skill** when the system has non-trivial
    runtime modes / state machines / training behaviors. Follow the structure of
    `experiment:mesoscope-vr-runtime`.

---

## Extending an existing lane (adding a new device to an existing dataclass)

This is the most common change. Examples: adding a new camera to `MesoscopeCameras`, adding a new
microcontroller-driven sensor to `MesoscopeMicroControllers`.

1. **Add the new fields** to the existing calibration dataclass, following the field-naming
   convention.

2. **Extend the binding class** to instantiate the new device's wrapper and include it in the
   underlying controller's `module_interfaces` tuple (or equivalent).

3. **Update the system-specific instance skill** to document the new device's calibration field
   surface.

4. **Bump the consumer library's version** per Contract 2 (schema versioning) in
   [layer-patterns.md](layer-patterns.md#contract-2-schema-versioning).

5. **Regenerate the system configuration YAML** on every deployment.

For microcontroller-module additions, also follow `experiment:microcontroller-interface`'s
"Adding a paired Module + Interface" workflow first — the firmware-and-wrapper pair must exist
before the binding class can compose it.
