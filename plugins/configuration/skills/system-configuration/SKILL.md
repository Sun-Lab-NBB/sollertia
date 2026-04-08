---
name: configure-system-configuration
description: >-
  Authors and modifies acquisition system configuration YAML files (MesoscopeSystemConfiguration and
  ServerConfiguration) for sollertia-shared-assets via the sl-configure MCP server. Covers schema
  introspection, read/write tool usage, supported acquisition systems, and the relationship between
  system configuration and the binding classes that consume it. Use when generating or editing the
  system configuration for a new Sollertia host or modifying calibration values for an existing host.
user-invocable: true
---

# Sollertia system configuration

Authors and modifies acquisition system configuration YAML files for `sollertia-shared-assets` using
the `sl-configure mcp` MCP server.

---

## Scope

**Covers:**
- Authoring `MesoscopeSystemConfiguration` (currently the only supported acquisition system)
- Authoring `ServerConfiguration` (remote storage transfer settings)
- Schema introspection via `describe_*_schema_tool`
- Read / write / validate workflow for system YAML files
- Relationship between configuration fields and the binding classes that consume them

**Does not cover:**
- Authoring experiment configuration or task templates (see `/experiment-configuration`)
- Authoring session-level data (see `/session-data`)
- Initial working directory setup (see `/working-directory`)
- Binding class semantics for cameras and microcontrollers (see the experiment plugin's
  `/camera-interface` and `/microcontroller-interface`)

---

## What lives in the system configuration

The system configuration captures everything that is **host-machine-specific** and **stable across
sessions**. It is read once at the start of every runtime session by `sl-run`. It does not contain
per-session metadata (those live in session descriptors) or per-experiment task structure (that lives
in experiment configuration).

For the Mesoscope-VR system, the configuration is composed of these top-level sections (each is its own
nested dataclass):

| Section                          | What it parameterizes                                            |
|----------------------------------|------------------------------------------------------------------|
| `MesoscopeFileSystem`            | Local raw / processed / cache directories on the acquisition PC  |
| `MesoscopeMicroControllers`      | Actor / Sensor / Encoder ports + per-module calibration data     |
| `MesoscopeCameras`               | Face / body camera indices + encoding parameters                 |
| `MesoscopeExternalAssets`        | Mesoscope acquisition software paths and trigger settings        |
| `MesoscopeGoogleSheets`          | Sheet IDs for animal metadata, water log, surgery log            |
| Top-level fields                 | System name, default Unity scene, valve calibration assignments  |

The `ServerConfiguration` is a separate file that captures settings for transferring preprocessed
sessions to a remote storage tier.

---

## MCP tool surface

| Tool                                          | Purpose                                                                |
|-----------------------------------------------|------------------------------------------------------------------------|
| `list_supported_acquisition_systems_tool`     | Lists supported acquisition system names (currently: mesoscope)        |
| `describe_system_configuration_schema_tool`   | Returns the field schema for a given acquisition system                |
| `read_system_configuration_tool`              | Reads the current system configuration YAML from the working directory |
| `write_system_configuration_tool`             | Writes a new system configuration YAML to the working directory        |
| `read_server_configuration_tool`              | Reads the current server configuration YAML                            |
| `write_server_configuration_tool`             | Writes a new server configuration YAML                                 |

The write tools accept the full nested dictionary that maps onto the dataclass tree. They validate the
shape against the schema before writing and will refuse partial updates — to change a single field,
read first, mutate the dictionary, then write the whole thing back.

---

## Authoring workflow

### Step 1: Verify prerequisites

- The `sollertia-shared-assets` MCP server is connected (otherwise hand off to
  `/configuration-mcp-environment-setup`).
- The working directory is set (otherwise hand off to `/working-directory`).

### Step 2: Determine whether to create or modify

Call `read_system_configuration_tool`. If it returns a configuration, you are modifying. If it returns
an empty / not-found response, you are creating from scratch.

### Step 3: Inspect the schema

```text
list_supported_acquisition_systems_tool()
describe_system_configuration_schema_tool(acquisition_system="mesoscope")
```

The schema response gives you the canonical field names, types, defaults, and units. Use this as the
source of truth — do not guess field names from documentation.

### Step 4: Gather values from the user (creation case)

For a new host, the values you cannot guess are:

- **Camera indices** — must come from `ataraxis@video:camera-setup` MCP discovery on the actual hardware.
- **Microcontroller ports** — must come from `ataraxis@communication:microcontroller-setup` MCP discovery,
  AND the user must confirm which physical Teensy plays the actor / sensor / encoder role.
- **File system roots** — `local_root_directory`, `nas_directory`, `mesoscope_directory`, etc. Ask the user
  for the absolute paths.
- **Google Sheet IDs** — ask the user for the sheet IDs (the long alphanumeric segment in the sheet URL).
- **Calibration data** — the defaults baked into `MesoscopeMicroControllers` and `MesoscopeExternalAssets`
  are reasonable starting points. Only override if the user has freshly measured values (e.g., new valve
  calibration after rebuilding the water delivery system).

### Step 5: Write the configuration

```text
write_system_configuration_tool(
    acquisition_system="mesoscope",
    configuration={ ... full nested dict ... }
)
```

The tool validates the dictionary against the schema and writes the YAML atomically.

### Step 6: Verify

Call `read_system_configuration_tool` and confirm the returned configuration matches what you wrote.
For modification cases, diff the relevant fields against the previous read.

### Step 7 (server config): Repeat for ServerConfiguration if needed

If the user is also setting up remote storage transfer, repeat steps 2–6 with
`read_server_configuration_tool` / `write_server_configuration_tool`.

---

## Modifying an existing system configuration

Common modification cases:

| Change                          | Section to mutate                                                  |
|---------------------------------|--------------------------------------------------------------------|
| New camera added                | `MesoscopeCameras` (add `<name>_camera_index` etc.)                |
| Camera reindexed                | `MesoscopeCameras.face_camera_index` / `body_camera_index`         |
| Teensy replaced / re-flashed    | `MesoscopeMicroControllers` (port may change)                      |
| Valve recalibrated              | `MesoscopeMicroControllers.valve_calibration_data`                 |
| New running wheel diameter      | `MesoscopeMicroControllers.wheel_diameter_cm`                      |
| Storage relocated               | `MesoscopeFileSystem.<root>_directory` fields                      |
| Google Sheet rotated            | `MesoscopeGoogleSheets.<sheet>_id`                                 |
| Mesoscope acquisition path moved | `MesoscopeExternalAssets.<path>` fields                           |

For any change that adds or removes fields (rather than just changing values), you also need to bump
`sollertia-shared-assets` and regenerate the configuration on every host using it. See the experiment
plugin's `/camera-interface` and `/microcontroller-interface` for the dataclass extension procedure.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] describe_system_configuration_schema_tool was called and used as the source of truth for field names
- [ ] Camera indices were sourced from ataraxis@video:camera-setup discovery, not guessed
- [ ] Microcontroller ports + roles were sourced from ataraxis@communication:microcontroller-setup, not guessed
- [ ] write_system_configuration_tool succeeded without schema errors
- [ ] read_system_configuration_tool returned the expected configuration after the write
- [ ] If new fields were added, sollertia-shared-assets version was bumped and the schema documented
```

---

## Related skills

| Skill                                                 | Relationship                                                       |
|-------------------------------------------------------|--------------------------------------------------------------------|
| `/working-directory`                                  | Required prerequisite — must be run first                          |
| `/configuration-mcp-environment-setup`                | Run first if the MCP server is not connected                       |
| `/experiment-configuration`                           | Authored separately, but consumes system configuration at runtime  |
| experiment plugin `/acquisition-system-setup`         | Validates discovered hardware against the system configuration     |
| experiment plugin `/camera-interface`                 | Dataclass extension procedure for adding new camera fields         |
| experiment plugin `/microcontroller-interface`        | Dataclass extension procedure for adding new microcontroller fields|
| `ataraxis@video:camera-setup`                         | Source of truth for camera indices                                 |
| `ataraxis@communication:microcontroller-setup`        | Source of truth for microcontroller ports                          |
