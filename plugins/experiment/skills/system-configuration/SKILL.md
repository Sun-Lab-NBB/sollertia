---
name: system-configuration
description: >-
  Authors and modifies the MesoscopeSystemConfiguration YAML file for sollertia-experiment via the
  sl-get MCP server. Owns the system configuration write tool and schema introspection. Covers the
  full nested dataclass tree (file system, microcontrollers, cameras, external assets, Google Sheets) and
  the relationship between configuration fields and the binding classes that consume them. Use when
  generating or editing the system configuration for a new Sollertia host or modifying calibration values
  for an existing host. Companion file references/mesoscope-reference.md documents every field in detail.
user-invocable: true
---

# Sollertia system configuration

Authors and modifies the `MesoscopeSystemConfiguration` YAML file for `sollertia-experiment` using
the `sl-get mcp` MCP server. This skill is the **exclusive** owner of `write_system_configuration_tool`
and `describe_system_configuration_schema_tool` — no other skill in the marketplace may call these. The
`list_supported_acquisition_systems_tool` lives on `slsa mcp` (sollertia-shared-assets) since the
enum itself remains the shared vocabulary across the platform.

For the full per-field schema reference, see
[mesoscope-reference.md](references/mesoscope-reference.md).

---

## Scope

**Covers:**
- Authoring `MesoscopeSystemConfiguration` (currently the only supported acquisition system)
- Schema introspection via `describe_system_configuration_schema_tool`
- Read / write / validate workflow for the system YAML file
- Reading the frozen system configuration captured at session start
  (`read_session_system_configuration_tool`)
- Relationship between configuration fields and the binding classes that consume them

**Does not cover:**
- `ServerConfiguration` authoring (forging plugin `/server-configuration`)
- `MesoscopeExperimentConfiguration` authoring (assets plugin `/experiment-configuration`)
- Task template authoring (assets plugin `/task-templates`)
- Session-level data (assets plugin `/session-data`, `/session-descriptors`;
  experiment plugin `/session-snapshots`)
- Initial working directory setup (assets plugin `/working-directory`)
- Binding class semantics for cameras and microcontrollers (sibling skills
  `/camera-interface` and `/microcontroller-interface`)

---

## What lives in the system configuration

The system configuration captures everything that is **host-machine-specific** and **stable across
sessions**. It is read once at the start of every runtime session by `sl-run`. It does not contain
per-session metadata, per-experiment task structure, or remote storage transfer settings.

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

`ServerConfiguration` (remote storage transfer) lives in a sibling YAML file and is owned by the
forging plugin's `/server-configuration` skill.

---

## MCP tool surface

All tools below are hosted on `sl-get mcp` (sollertia-experiment):

| Tool                                        | Purpose                                                               |
|---------------------------------------------|-----------------------------------------------------------------------|
| `describe_system_configuration_schema_tool` | Returns the field schema for MesoscopeSystemConfiguration             |
| `read_system_configuration_tool`            | Reads the active system configuration YAML from the working directory |
| `write_system_configuration_tool`           | Writes a new system configuration YAML (exclusive to this skill)      |
| `validate_system_configuration_tool`        | Validates the loaded system configuration and reports mount status    |
| `check_system_mounts_tool`                  | Checks every filesystem path declared in the configuration            |
| `read_session_system_configuration_tool`    | Reads the frozen system configuration captured at session start       |

The companion enumeration `list_supported_acquisition_systems_tool` (on `slsa mcp`,
sollertia-shared-assets) returns the enum values. The CLI entry point for authoring is
`sl-configure system` hosted by sollertia-experiment.

The write tool accepts the full nested dictionary that maps onto the dataclass tree. It validates the
shape against the schema before writing and refuses partial updates — to change a single field, read
first, mutate the dictionary, then write the whole thing back.

---

## Authoring workflow

### Step 1: Verify prerequisites

- The `sl-get mcp` server (sollertia-experiment) is connected (else hand off to the experiment plugin's
  `/experiment-mcp-environment-setup`).
- The working directory is set (else hand off to the assets plugin's `/working-directory`).

### Step 2: Determine whether to create or modify

Call `read_system_configuration_tool`. If it returns a configuration, you are modifying. If it returns
an empty / not-found response, you are creating from scratch.

### Step 3: Inspect the schema

```text
describe_system_configuration_schema_tool()
```

The schema response gives you the canonical field names, types, defaults, and units. Use this **and**
[mesoscope-reference.md](references/mesoscope-reference.md) as the source of truth — do not
guess field names.

### Step 4: Gather values from the user (creation case)

For a new host, the values you cannot guess are:

- **Camera indices** — must come from the experiment plugin's `/acquisition-system-setup` skill, which
  in turn uses `ataraxis@video:camera-setup` for hardware discovery. Do not call camera discovery tools
  from this skill.
- **Microcontroller ports** — must come from the experiment plugin's `/acquisition-system-setup` skill,
  which uses `ataraxis@communication:microcontroller-setup`. The user must confirm which physical
  Teensy plays the actor / sensor / encoder role.
- **File system roots** — `local_root_directory`, `nas_directory`, `mesoscope_directory`, etc. Ask the
  user for the absolute paths.
- **Google Sheet IDs** — ask the user for the sheet IDs (the long alphanumeric segment in the sheet URL).
- **Calibration data** — the defaults in [mesoscope-reference.md](references/mesoscope-reference.md) are
  reasonable starting points. Only override if the user has freshly measured values.

### Step 5: Write the configuration

```text
write_system_configuration_tool(
    configuration_payload={ ... full nested dict ... },
    overwrite=True,
)
```

The tool validates the dictionary against the schema and writes the YAML atomically.

### Step 6: Verify

Call `read_system_configuration_tool` and confirm the returned configuration matches what you wrote.
For modification cases, diff the relevant fields against the previous read.

### Step 7: Hand off for server configuration

If the user is also setting up remote storage transfer, hand off to the forging plugin's
`/server-configuration` skill. This skill does not write the server configuration file.

---

## Modifying an existing system configuration

Common modification cases:

| Change                           | Section to mutate                                          |
|----------------------------------|------------------------------------------------------------|
| New camera added                 | `MesoscopeCameras` (add `<name>_camera_index` etc.)        |
| Camera reindexed                 | `MesoscopeCameras.face_camera_index` / `body_camera_index` |
| Teensy replaced / re-flashed     | `MesoscopeMicroControllers` (port may change)              |
| Valve recalibrated               | `MesoscopeMicroControllers.valve_calibration_data`         |
| New running wheel diameter       | `MesoscopeMicroControllers.wheel_diameter_cm`              |
| Storage relocated                | `MesoscopeFileSystem.<root>_directory` fields              |
| Google Sheet rotated             | `MesoscopeGoogleSheets.<sheet>_id`                         |
| Mesoscope acquisition path moved | `MesoscopeExternalAssets.<path>` fields                    |

For any change that adds or removes fields (rather than just changing values), the dataclass extension
itself is a code change to `sollertia-experiment` (the package that now owns MesoscopeSystemConfiguration
and its nested hardware dataclasses). The sibling `/camera-interface` and `/microcontroller-interface`
skills document the dataclass extension procedure and explicitly hand off back to this skill for the YAML
regeneration step.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host (assets plugin)
- [ ] sl-get mcp server (sollertia-experiment) is connected
- [ ] describe_system_configuration_schema_tool was called and used as the source of truth for field names
- [ ] references/mesoscope-reference.md was consulted for field semantics
- [ ] Camera indices and microcontroller ports were sourced from /acquisition-system-setup, not guessed
- [ ] write_system_configuration_tool succeeded without schema errors
- [ ] read_system_configuration_tool returned the expected configuration after the write
- [ ] Did not call write_server_configuration_tool — handed off to forging plugin /server-configuration if needed
```

---

## Related skills

| Skill                                              | Relationship                                                        |
|----------------------------------------------------|---------------------------------------------------------------------|
| assets plugin `/working-directory`          | Required prerequisite — must be run first                           |
| this plugin `/experiment-mcp-environment-setup`               | Run first if sl-get mcp is not connected                            |
| forging plugin `/server-configuration`             | Owns `ServerConfiguration` (moved out of the assets plugin)  |
| assets plugin `/experiment-configuration`   | Authored separately, consumes system configuration at runtime       |
| assets plugin `/session-data`               | Session-level data                                                  |
| this plugin `/session-snapshots`                   | Sibling — per-session ZaberPositions and MesoscopePositions         |
| this plugin `/acquisition-system-setup`            | Source of camera indices and microcontroller ports via discovery    |
| this plugin `/camera-interface`                    | Dataclass extension procedure; hands off here for YAML regeneration |
| this plugin `/microcontroller-interface`           | Dataclass extension procedure; hands off here for YAML regeneration |
