---
name: acquisition-system-setup
description: >-
  Discovers, verifies, and reports the hardware connected to a Sollertia acquisition PC
  (cameras, microcontrollers, Zaber motors, MQTT brokers) via the video, communication, and
  experiment MCP servers. Use when bringing up a new acquisition PC, troubleshooting hardware
  connectivity, or verifying discovered hardware against the recorded system configuration.
user-invocable: false
---

# Discovering acquisition system hardware

Discovers, verifies, and reports the hardware connected to a Sollertia data acquisition PC. Focuses exclusively
on hardware introspection — all configuration file authoring, working directory setup, credential management,
and project / experiment creation are owned by sibling plugins (named per item in the hand-off list below)
and must be invoked by hand-off.

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

**Does not cover** (hand off to the owning plugin — named per item, since these span three plugins):
- Setting the working directory, Google credentials, or task templates directory → assets plugin `/working-directory`
- Reading, writing, or validating system configuration YAML → this plugin `/mesoscope-vr`
- Creating projects → assets plugin `/project-hierarchy`
- Authoring task templates → assets plugin `/task-templates`
- Authoring per-project experiment configurations → assets plugin `/experiment-configuration`
- Reading the `SessionData` marker file → assets plugin `/session-data`
- Reading session descriptors → assets plugin `/session-descriptors`
- Reading frozen runtime snapshots → this plugin `/session-snapshots`
- Reading subject metadata → assets plugin `/subject-metadata`
- Reading or curating datasets → forging plugin `/datasets`

You MUST NOT call any `slsa` MCP tool that mutates state. Read-only `read_*` and `discover_*`
tools from the assets plugin's MCP server may be called as a "natural share" only when verifying that
discovered hardware matches the recorded configuration.

---

## MCP server requirements

This skill uses MCP tools from libraries other than `sollertia-shared-assets`. The `slsa mcp` server is
required only because hand-off targets in the assets plugin depend on it.

| Server                  | CLI Command | Used directly by this skill | Purpose                                          |
|-------------------------|-------------|-----------------------------|--------------------------------------------------|
| ataraxis-video-system   | `axvs mcp`  | yes                         | Camera discovery, runtime requirements, CTI      |
| ataraxis-comm-interface | `axci mcp`  | yes                         | Microcontroller discovery, MQTT broker check     |
| sollertia-experiment    | `sle mcp`   | yes                         | Zaber motor discovery                            |
| sollertia-shared-assets | `slsa mcp`  | no (hand-off targets only)  | Read-only verification of recorded configuration |

If a required MCP server is unavailable, hand off to the appropriate plugin's MCP environment setup
skill: `ataraxis@video:video-mcp-environment-setup`,
`ataraxis@communication:communication-mcp-environment-setup`, or this plugin's
`/experiment-mcp-environment-setup`.

---

## Supported acquisition systems

| System      | Description                                 | Schema reference                                                |
|-------------|---------------------------------------------|-----------------------------------------------------------------|
| `mesoscope` | Two-photon mesoscope with VR behavioral rig | this plugin `/mesoscope-vr` (configuration-fields.md companion) |

When working with a specific acquisition system, hand off to that system's skill (e.g. `/mesoscope-vr`)
to read the canonical field schema. This skill does not duplicate that schema reference.

---

## Network storage prerequisites

All acquisition systems require network storage locations to be mounted via SMB before any configuration work
begins. These mounts are managed at the operating system level and are not the responsibility of any skill or
MCP tool.

### Required SMB mounts (all systems)

Both long-term storage locations are configured as entries in the system configuration's
`storage_directories` mapping, keyed by destination name (the mapping also accepts additional
destinations under arbitrary names):

| Mount Purpose  | `storage_directories` key | Description                              |
|----------------|---------------------------|------------------------------------------|
| Compute server | `Server`                  | Long-term hot storage for processed data |
| NAS backup     | `NAS`                     | Archival/cold storage backup             |

### System-specific mounts

For mesoscope systems:

| Mount Purpose      | Configuration Field    | Description                                    |
|--------------------|------------------------|------------------------------------------------|
| ScanImagePC share  | `mesoscope_directory`  | Shared directory where ScanImagePC saves TIFFs |

The ScanImagePC (MATLAB workstation) must expose a shared directory that the acquisition PC can access.

### Mount configuration

Before invoking this skill, ensure:

1. All network shares are mounted and accessible from the acquisition PC.
2. The mount points have appropriate read/write permissions.
3. Mounts persist across reboots (configured via the OS-appropriate mechanism — e.g. `/etc/fstab` or
   systemd mount units on Linux).

```bash
# Verify mounts are accessible (Linux example; use the OS-appropriate listing on Windows/macOS)
ls /mnt/server/data
ls /mnt/nas/backup
ls /mnt/mesoscope/data  # mesoscope systems only
```

If mounts are not configured, coordinate with system administrators to set up the SMB shares before proceeding.

---

## Skill modes

### Discovery mode

Use when the user wants to identify connected hardware without modifying any configuration.

**When to use:**

- "What cameras are connected?"
- "Which serial ports have microcontrollers?"
- "Are the Zaber motors responding?"
- Verifying hardware accessibility before invoking the assets plugin
- Troubleshooting hardware connectivity

This is the default mode of this skill.

### Verification mode

Use when the user wants to confirm that the discovered hardware matches the recorded system configuration.

**When to use:**

- After completing an assets plugin authoring workflow
- Before running a runtime acquisition session
- After replacing or relocating a piece of hardware

**Verification steps:**

1. Run hardware discovery (Phases 1–2 below).
2. Hand off to this plugin's `/mesoscope-vr` for a read-only `read_system_configuration_tool`
   call to fetch the recorded values.
3. Compare discovered values against recorded values and report any drift to the user.
4. If drift exists, hand off to `/mesoscope-vr` to update the recorded values. Do not edit YAML or call
   `write_system_configuration_tool` from this skill.

---

## Hardware discovery workflow

### Phase 1: Runtime prerequisites

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

### Phase 2: Hardware discovery

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

Note the port assignments. `list_microcontrollers()` reports a numeric microcontroller ID per port;
in the Mesoscope-VR system each ID maps to a fixed role:

| Reported ID | Role    | Function                                            |
|-------------|---------|-----------------------------------------------------|
| `101`       | Actor   | Controls outputs (valves, brake, screen triggers)   |
| `152`       | Sensor  | Monitors inputs (lick, torque, mesoscope frame TTL) |
| `203`       | Encoder | High-precision wheel quadrature encoder             |

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

On Linux, microcontrollers enumerate as `/dev/ttyACM*` (USB CDC ACM) and Zaber motors as `/dev/ttyUSB*`
(USB serial adapters). On Windows and macOS the path format differs (e.g. `COMx` on Windows), but the
CDC-ACM (microcontroller) vs USB-serial (Zaber) device-type distinction still holds.

Do not confuse these device types when reporting discovered hardware.

### Phase 3: Report and hand off

After discovery completes, report the discovered hardware to the user as a structured table.

**If the user is performing initial bringup**, hand off in this order (owning plugin named per step):

1. assets plugin `/working-directory` — set the working directory, Google credentials, and task
   templates directory.
2. this plugin `/mesoscope-vr` — author the host machine's system configuration YAML against the
   discovered hardware values.
3. forging plugin `/server-configuration` — author the remote storage transfer configuration if the
   host pushes to a compute server.
4. assets plugin `/project-hierarchy` — create the project (or projects) the host will record under.
5. assets plugin `/task-templates` — author or import the task templates the project will use.
6. assets plugin `/experiment-configuration` — author the per-project experiment configuration that
   wires a template to a project.

**If the user is performing verification**, hand off to this plugin's `/mesoscope-vr` for a read-only
fetch of the recorded values, then report the diff between discovered and recorded.

**If the user is troubleshooting**, use the troubleshooting table below.

You MUST NOT call `set_working_directory_tool`, `set_google_credentials_tool`,
`set_task_templates_directory_tool`, `write_system_configuration_tool`, or
`write_server_configuration_tool` directly under any circumstances.

---

## Quick reference

### Hardware discovery commands

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

| Error                              | Cause                         | Solution                                                           |
|------------------------------------|-------------------------------|--------------------------------------------------------------------|
| Camera not found at expected index | Wrong camera index            | Re-run `list_cameras()`, hand off to `/mesoscope-vr` to update     |
| Microcontroller connection failed  | Wrong port or disconnected    | Re-run `list_microcontrollers()`, check USB cables                 |
| Zaber motor not responding         | Wrong port or powered off     | Re-run `get_zaber_devices_tool()`, verify power supply             |
| MQTT broker unreachable            | Broker not running            | Start Mosquitto or the configured MQTT broker                      |
| FFMPEG not found                   | FFMPEG not installed          | Install FFMPEG via the OS package manager                          |
| GPU not detected                   | NVIDIA driver missing         | Install NVIDIA driver and restart                                  |
| CTI file not configured            | GenTL producer not registered | Hand off to `ataraxis@video:camera-setup` to register the CTI file |

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
- [ ] Handed off to /working-directory, /mesoscope-vr, /server-configuration, /project-hierarchy,
      /task-templates, or /experiment-configuration for any state mutation
```

---

## Related skills

| Skill                                          | Relationship                                                            |
|------------------------------------------------|-------------------------------------------------------------------------|
| assets plugin `/working-directory`             | Owns bootstrap state (working dir, credentials, templates dir)          |
| this plugin `/mesoscope-vr`                    | Owns `MesoscopeSystemConfiguration` authoring and validation            |
| forging plugin `/server-configuration`         | Owns `ServerConfiguration` authoring and validation                     |
| assets plugin `/project-hierarchy`             | Owns project creation                                                   |
| assets plugin `/task-templates`                | Owns task template authoring                                            |
| assets plugin `/experiment-configuration`      | Owns per-project experiment configuration authoring                     |
| this plugin `/system-health-check`             | Lighter-weight pre-session verification sweep                           |
| this plugin `/pipeline`                        | Phase 3 (Hardware bringup) is owned by this skill                       |
| `ataraxis@video:camera-setup`                  | Canonical home for CTI configuration and runtime requirement deep-dives |
| `ataraxis@communication:microcontroller-setup` | Canonical home for microcontroller manifest and discovery deep-dives    |
