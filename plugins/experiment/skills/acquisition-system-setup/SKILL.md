---
name: discovering-acquisition-system-hardware
description: >-
  Discovers, verifies, and reports the hardware connected to a Sollertia data acquisition PC. Covers cameras,
  microcontrollers, Zaber motors, MQTT brokers, and video runtime requirements via ataraxis-video-system,
  ataraxis-communication-interface, and sollertia-experiment MCP tools. Use when bringing up a new acquisition PC,
  troubleshooting hardware connectivity, or verifying that the discovered hardware matches the recorded system
  configuration. Hands off all configuration authoring and bootstrap state setup to the assets plugin.
user-invocable: true
---

# Discovering acquisition system hardware

Discovers, verifies, and reports the hardware connected to a Sollertia data acquisition PC. Focuses exclusively
on hardware introspection — all configuration file authoring, working directory setup, credential management,
and project / experiment creation are owned by the assets plugin and must be invoked by hand-off.

---

## Scope

**Covers:**
- Discovering connected cameras (OpenCV and Harvesters / GenICam)
- Discovering connected microcontrollers (USB CDC ACM)
- Discovering connected Zaber motors (USB serial)
- Verifying MQTT broker reachability
- Verifying video runtime requirements (FFMPEG, GPU, CTI file)
- Verifying network storage mounts at the OS level
- Reporting discrepancies between discovered hardware and the active system configuration

**Does not cover** (hand off to the assets plugin):
- Setting the working directory, Google credentials, or task templates directory → `/working-directory`
- Reading, writing, or validating system configuration YAML → `/system-configuration`
- Reading, writing, or validating server configuration YAML → `/server-configuration`
- Creating projects → `/project-hierarchy`
- Authoring task templates → `/task-templates`
- Authoring per-project experiment configurations → `/experiment-configuration`
- Reading the `SessionData` marker file → `/session-data`
- Reading session descriptors → `/session-descriptors`
- Reading frozen runtime snapshots → `/session-snapshots`
- Reading subject metadata → `/subject-metadata`
- Reading or curating datasets → `/datasets`

This skill MUST NOT call any `slsa` MCP tool that mutates state. Read-only `read_*` and `discover_*`
tools from the assets plugin's MCP server may be called as a "natural share" only when verifying that
discovered hardware matches the recorded configuration.

---

## MCP Server Requirements

This skill uses MCP tools from libraries other than `sollertia-shared-assets`. The `slsa mcp` server is
required only because hand-off targets in the assets plugin depend on it.

| Server                  | CLI Command        | Used directly by this skill | Purpose                                          |
|-------------------------|--------------------|-----------------------------|--------------------------------------------------|
| ataraxis-video-system   | `axvs mcp`         | yes                         | Camera discovery, runtime requirements, CTI      |
| ataraxis-comm-interface | `axci mcp`         | yes                         | Microcontroller discovery, MQTT broker check     |
| sollertia-experiment           | `sle get mcp`       | yes                         | Zaber motor discovery                            |
| sollertia-shared-assets | `slsa mcp` | no (hand-off targets only)  | Read-only verification of recorded configuration |

If a required MCP server is unavailable, hand off to the appropriate plugin's MCP environment setup
skill: `ataraxis@video:video-mcp-environment-setup`,
`ataraxis@communication:communication-mcp-environment-setup`, or this plugin's
`/experiment-mcp-environment-setup`.

---

## Supported Acquisition Systems

| System      | Description                                 | Schema reference                                                              |
|-------------|---------------------------------------------|-------------------------------------------------------------------------------|
| `mesoscope` | Two-photon mesoscope with VR behavioral rig | this plugin `/system-configuration` (MESOSCOPE_REFERENCE.md companion) |

When working with a specific acquisition system, hand off to `/system-configuration` to read the canonical
field schema. This skill does not duplicate that schema reference.

---

## Network Storage Prerequisites

All acquisition systems require network storage locations to be mounted via SMB before any configuration work
begins. These mounts are managed at the operating system level and are not the responsibility of any skill or
MCP tool.

### Required SMB Mounts (All Systems)

| Mount Purpose      | Configuration Field    | Description                                    |
|--------------------|------------------------|------------------------------------------------|
| Compute server     | `server_directory`     | Long-term hot storage for processed data       |
| NAS backup         | `nas_directory`        | Archival/cold storage backup                   |

### System-Specific Mounts

For mesoscope systems:

| Mount Purpose      | Configuration Field    | Description                                    |
|--------------------|------------------------|------------------------------------------------|
| ScanImagePC share  | `mesoscope_directory`  | Shared directory where ScanImagePC saves TIFFs |

The ScanImagePC (MATLAB workstation) must expose a shared directory that the acquisition PC can access.

### Mount Configuration

Before invoking this skill, ensure:

1. All network shares are mounted and accessible from the acquisition PC.
2. The mount points have appropriate read/write permissions.
3. Mounts persist across reboots (`/etc/fstab` or systemd mount units).

```bash
# Verify mounts are accessible
ls /mnt/server/data
ls /mnt/nas/backup
ls /mnt/mesoscope/data  # mesoscope systems only
```

If mounts are not configured, coordinate with system administrators to set up the SMB shares before proceeding.

---

## Skill Modes

### Discovery Mode

Use when the user wants to identify connected hardware without modifying any configuration.

**When to use:**

- "What cameras are connected?"
- "Which serial ports have microcontrollers?"
- "Are the Zaber motors responding?"
- Verifying hardware accessibility before invoking the assets plugin
- Troubleshooting hardware connectivity

This is the default mode of this skill.

### Verification Mode

Use when the user wants to confirm that the discovered hardware matches the recorded system configuration.

**When to use:**

- After completing a assets plugin authoring workflow
- Before running a runtime acquisition session
- After replacing or relocating a piece of hardware

**Verification steps:**

1. Run hardware discovery (Phases 1–2 below).
2. Hand off to the assets plugin's `/system-configuration` for a read-only `read_system_configuration_tool`
   call to fetch the recorded values.
3. Compare discovered values against recorded values and report any drift to the user.
4. If drift exists, hand off to `/system-configuration` to update the recorded values. Do not edit YAML or call
   `write_system_configuration_tool` from this skill.

---

## Hardware Discovery Workflow

### Phase 1: Runtime Prerequisites

**Check video system runtime requirements:**

```text
check_runtime_requirements()
```

Expected output reports FFMPEG, GPU, and CTI file status. If CTI is not configured and the system uses
Harvesters cameras:

```text
set_cti_file("/path/to/gentl_producer.cti")
```

`set_cti_file` is owned by `ataraxis@video:camera-setup`. This skill is allowed to invoke it because the CTI
path is part of the ataraxis video MCP server's state, not the slsa state. Refer to `ataraxis@video:camera-setup`
for the canonical CTI configuration workflow.

**Check MQTT broker:**

```text
check_mqtt_broker(host="127.0.0.1", port=1883)
```

If the broker is not running, instruct the user to start Mosquitto (or their broker service) before continuing.

### Phase 2: Hardware Discovery

**Cameras:**

```text
list_cameras()
```

Note the camera indices and any model / resolution information returned. For mesoscope systems, identify which
index corresponds to the face camera and which to the body camera.

**Microcontrollers:**

```text
list_microcontrollers()
```

Note the port assignments. The reported microcontroller IDs indicate role:

| Role     | Function                                              |
|----------|-------------------------------------------------------|
| Actor    | Controls outputs (valves, brake, screen triggers)     |
| Sensor   | Monitors inputs (lick, torque, mesoscope frame TTL)   |
| Encoder  | High-precision wheel quadrature encoder               |

**Zaber motors:**

```text
get_zaber_devices_tool()
```

Note the port assignments. For the mesoscope system, the expected motor groups are:

| Group           | Axes              |
|-----------------|-------------------|
| Headbar motors  | Z, Pitch, Roll    |
| Lickport motors | Z, Y, X           |
| Wheel motor     | X (horizontal)    |

**Device path convention:**

- Microcontrollers use `/dev/ttyACM*` (USB CDC ACM)
- Zaber motors use `/dev/ttyUSB*` (USB serial adapters)

Do not confuse these device types when reporting discovered hardware.

### Phase 3: Report and hand off

After discovery completes, report the discovered hardware to the user as a structured table.

**If the user is performing initial bringup**, hand off to the assets plugin in this order:

1. `/working-directory` — set the working directory, Google credentials, and task templates directory.
2. `/system-configuration` — author the host machine's system configuration YAML against the discovered
   hardware values.
3. `/server-configuration` — author the remote storage transfer configuration if the host pushes to a
   compute server.
4. `/project-hierarchy` — create the project (or projects) the host will record under.
5. `/task-templates` — author or import the task templates the project will use.
6. `/experiment-configuration` — author the per-project experiment configuration that wires a template
   to a project.

**If the user is performing verification**, hand off to `/system-configuration` for a read-only fetch of the
recorded values, then report the diff between discovered and recorded.

**If the user is troubleshooting**, use the troubleshooting table below.

This skill MUST NOT call `set_working_directory_tool`, `set_google_credentials_tool`,
`set_task_templates_directory_tool`, `write_system_configuration_tool`, `write_server_configuration_tool`, or
`create_project_tool` directly under any circumstances.

---

## Quick Reference

### Hardware Discovery Commands

| Hardware           | MCP Tool                               | What to look for                   |
|--------------------|----------------------------------------|------------------------------------|
| Cameras            | `list_cameras()`                       | Index, resolution, model name      |
| Microcontrollers   | `list_microcontrollers()`              | Port path, microcontroller ID      |
| Zaber motors       | `get_zaber_devices_tool()`             | Port path, device name, axis count |
| MQTT broker        | `check_mqtt_broker("127.0.0.1", 1883)` | Connection success/failure         |
| Video requirements | `check_runtime_requirements()`         | FFMPEG, GPU, CTI status            |
| CTI file status    | `get_cti_status()`                     | CTI file path or "not set"         |

---

## Troubleshooting

| Error                              | Cause                        | Solution                                  |
|------------------------------------|------------------------------|-------------------------------------------|
| Camera not found at expected index | Wrong camera index           | Re-run `list_cameras()`, hand off to `/system-configuration` to update |
| Microcontroller connection failed  | Wrong port or disconnected   | Re-run `list_microcontrollers()`, check USB cables |
| Zaber motor not responding         | Wrong port or powered off    | Re-run `get_zaber_devices_tool()`, verify power supply |
| MQTT broker unreachable            | Broker not running           | Start Mosquitto or the configured MQTT broker |
| FFMPEG not found                   | FFMPEG not installed         | Install FFMPEG via the OS package manager |
| GPU not detected                   | NVIDIA driver missing        | Install NVIDIA driver and restart |
| CTI file not configured            | GenTL producer not registered| Hand off to `ataraxis@video:camera-setup` to register the CTI file |

For configuration-file-level errors (working directory not set, schema validation failures, missing projects),
hand off to the assets plugin skill that owns the affected asset.

---

## Verification checklist

```text
- [ ] Network storage mounts verified at the OS level
- [ ] Required MCP servers (ataraxis video, ataraxis comm, sollertia-experiment) confirmed reachable
- [ ] check_runtime_requirements() reported FFMPEG and GPU OK
- [ ] CTI file status confirmed (if using Harvesters cameras)
- [ ] check_mqtt_broker() reported broker reachable
- [ ] list_cameras() returned the expected cameras
- [ ] list_microcontrollers() returned the expected microcontrollers and roles
- [ ] get_zaber_devices_tool() returned the expected motor groups
- [ ] Discovered hardware reported to user as a structured table
- [ ] Did NOT call any slsa setter tool from this skill
- [ ] Handed off to /working-directory, /system-configuration, /server-configuration, /project-hierarchy,
      /task-templates, or /experiment-configuration for any state mutation
```

---

## Related skills

| Skill                                            | Relationship                                                       |
|--------------------------------------------------|--------------------------------------------------------------------|
| assets plugin `/working-directory`        | Owns bootstrap state (working dir, credentials, templates dir)     |
| this plugin `/system-configuration`     | Owns `MesoscopeSystemConfiguration` authoring and validation       |
| forging plugin `/server-configuration`           | Owns `ServerConfiguration` authoring and validation                |
| assets plugin `/project-hierarchy`        | Owns project creation                                              |
| assets plugin `/task-templates`           | Owns task template authoring                                       |
| assets plugin `/experiment-configuration` | Owns per-project experiment configuration authoring                |
| this plugin `/system-health-check`               | Lighter-weight pre-session verification sweep                      |
| this plugin `/pipeline`                          | Phase 3 (Hardware bringup) is owned by this skill                  |
| `ataraxis@video:camera-setup`                    | Canonical home for CTI configuration and runtime requirement deep-dives |
| `ataraxis@communication:microcontroller-setup`   | Canonical home for microcontroller manifest and discovery deep-dives |
