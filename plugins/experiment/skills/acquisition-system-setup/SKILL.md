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
- Verifying any declared network storage mounts at the OS level
- Reporting discrepancies between discovered hardware and the active system configuration

**Does not cover** (hand off to the owning plugin — named per item, since these span three plugins):
- Setting the working directory, Google credentials, or task templates directory → assets plugin `/working-directory`
- Reading, writing, or validating system configuration YAML → the active acquisition system's skill
  (`/mesoscope-vr` for the `mesoscope` system)
- Creating projects → assets plugin `/project-hierarchy`
- Authoring task templates → assets plugin `/task-templates`
- Authoring per-project experiment configurations → assets plugin `/experiment-configuration`
- Reading the `SessionData` marker file → assets plugin `/session-data`
- Reading session descriptors → assets plugin `/session-descriptors`
- Reading frozen runtime snapshots → this plugin `/mesoscope-vr-snapshots`
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
| sollertia-experiment    | `sle mcp`   | yes                         | Zaber motor discovery, storage mount checks      |
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

Network storage locations are optional. An acquisition system is not required to define any — it can operate
entirely on local storage. When a system does define network storage locations, every one of them must be
reachable through a direct-filesystem mount (SMB or an equivalent protocol that exposes the share as a local
path) before any configuration work begins. Establishing these mounts is an operating-system-level task — no
skill or MCP tool creates them. Verifying that the declared mounts are reachable, by contrast, is available
through the `sollertia-experiment` (`sle mcp`) server's `check_system_mounts_tool()` and
`check_mount_accessibility_tool(path=...)`.

### Long-term storage mounts

Long-term storage locations are configured as entries in the system configuration's `storage_directories`
mapping, keyed by destination name (the mapping also accepts additional destinations under arbitrary names).
An entry left as an empty path is treated as not configured and is skipped — only the destinations a system
actually declares need to be mounted:

| Mount Purpose  | `storage_directories` key | Description                              |
|----------------|---------------------------|------------------------------------------|
| Compute server | `Server`                  | Long-term hot storage for processed data |
| NAS backup     | `NAS`                     | Archival/cold storage backup             |

These locations are essentially one-way egress targets: the acquisition PC pushes acquired data to them and
clears it from local disk. Data returns only during a deliberate cross-infrastructure migration — for example,
moving an animal between projects — which is why the `storage_directories` mapping order defines pull-back
preference.

### Within-system shares

Within-system shares connect the separate PCs that together make up one acquisition system. Unlike long-term
storage locations, which are one-way egress targets, a within-system share has no fixed direction — flow may be
unidirectional or bidirectional. Typically, every PC aggregates its data onto the main acquisition PC before the data 
is pushed to the long-term storage destination(s), but the main PC can also write back to a peer.

The Mesoscope-VR system is a concrete example. Its `mesoscope_directory` field declares a within-system share
with the ScanImagePC (a MATLAB workstation):

| Mount Purpose      | Configuration Field    | Description                                    |
|--------------------|------------------------|------------------------------------------------|
| ScanImagePC share  | `mesoscope_directory`  | Data share with the ScanImagePC                |

The share moves data only. The ScanImagePC writes the acquired imaging data — TIFFs,
`MotionEstimator.me`, `fov.roi`, `zstack.tiff`, and the desktop alignment screenshot — into the
share, and the acquisition PC reads it back during preprocessing and session setup. Mesoscope
acquisition control travels over the shared MQTT broker, not through the share, so the ScanImagePC
must expose this directory for the acquisition PC to mount as a direct-filesystem path.

### Mount configuration

For every network storage location the system declares, ensure before invoking this skill:

1. The share is mounted and accessible from the acquisition PC.
2. The mount point has appropriate read/write permissions.
3. The mount persists across reboots (configured via the OS-appropriate mechanism — e.g. `/etc/fstab` or
   systemd mount units on Linux).

To verify that the declared mounts are reachable, use the `sle` MCP server (these tools check the paths; they
do not create the mounts):

```text
check_system_mounts_tool()               # validates mesoscope_directory and every configured storage_directories path
check_mount_accessibility_tool(path=...) # drills into a single path that failed the sweep
```

The OS-level equivalent is a direct listing (Linux example; use the OS-appropriate command on Windows/macOS):

```bash
ls /mnt/server/data
ls /mnt/nas/backup
ls /mnt/mesoscope/data  # mesoscope systems only
```

If a declared mount is not reachable, coordinate with system administrators to set up the SMB share (or an
equivalent direct-filesystem-access protocol) — no skill or MCP tool can create the mount for you.

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
2. Hand off to the active acquisition system's skill (currently `/mesoscope-vr`, for the `mesoscope`
   system) for a read-only `read_system_configuration_tool` call to fetch the recorded values.
3. Compare discovered values against recorded values and report any drift to the user.
4. If drift exists, hand off to that same system skill to update the recorded values. Do not edit YAML or
   call `write_system_configuration_tool` from this skill.

---

## Hardware discovery workflow

Each tool below is owned by one of the three discovery MCP servers. This skill invokes them but hands the
canonical usage detail to the owner skill named after each table. Tool names follow each server's own
convention: `axvs` and `axci` tools carry no suffix, while `sle` tools carry a `_tool` suffix.

### Phase 1: Runtime prerequisites

| Tool                         | Server | Purpose                               |
|------------------------------|--------|---------------------------------------|
| `check_runtime_requirements` | axvs   | FFMPEG, GPU, and CTI file status      |
| `get_cti_status`             | axvs   | CTI (.cti) file path, or "not set"    |
| `set_cti_file`               | axvs   | Sets the .cti path (Harvesters)       |
| `check_mqtt_broker`          | axci   | MQTT broker reachability (host, port) |
| `check_unity_bridge_tool`    | sle    | Unity Editor MCP Bridge reachability  |

`check_runtime_requirements`, `get_cti_status`, and `set_cti_file` are owned by `ataraxis@video:camera-setup`;
`check_mqtt_broker` is owned by `ataraxis@communication:microcontroller-setup`; `check_unity_bridge_tool` is
owned by `/vr-driver-interface` and applies only to systems that drive a Unity VR task.

Invoke `check_runtime_requirements` first. If it reports the CTI file as unconfigured and the system uses
Harvesters cameras, set the path with `set_cti_file` — the CTI path lives in the video MCP server's state, not
slsa state, so this skill may call it; see `ataraxis@video:camera-setup` for the canonical CTI workflow. Then
invoke `check_mqtt_broker`; if the broker is unreachable, instruct the user to start their broker service
(e.g. Mosquitto) before continuing. For a system that drives a Unity VR task, also invoke
`check_unity_bridge_tool` (CLI: `sle get unity`); if it reports the bridge unreachable, instruct the user to
open the Unity project in the editor — its MCP bridge auto-starts — before running an experiment session.

### Phase 2: Hardware discovery

| Tool                     | Server | Discovers                          |
|--------------------------|--------|------------------------------------|
| `list_cameras`           | axvs   | Camera index, model, resolution    |
| `list_microcontrollers`  | axci   | Port path + microcontroller ID     |
| `get_zaber_devices_tool` | sle    | Port path, device name, axis count |

`list_cameras` is owned by `ataraxis@video:camera-setup`, `list_microcontrollers` by
`ataraxis@communication:microcontroller-setup`, and `get_zaber_devices_tool` by this plugin's
`/zaber-interface`.

Invoke each tool and record what it returns. Mapping the discovered IDs and motor layout to fixed hardware
roles is system-specific — hand off to the active acquisition system's skill for the canonical mapping.

For the `mesoscope` system, that mapping is the following concrete example. Each `list_microcontrollers` ID
maps to a fixed role:

| Reported ID | Role    | Function                                            |
|-------------|---------|-----------------------------------------------------|
| `101`       | Actor   | Controls outputs (valves, brake, screen triggers)   |
| `152`       | Sensor  | Monitors inputs (lick, torque, mesoscope frame TTL) |
| `203`       | Encoder | High-precision wheel quadrature encoder             |

The Zaber motors form three groups:

| Group           | Axes           |
|-----------------|----------------|
| Headbar motors  | Z, Pitch, Roll |
| Lickport motors | Z, Y, X        |
| Wheel motor     | X (horizontal) |

The system also exposes two cameras — a face camera and a body camera; match each `list_cameras` index to its
role against the system's recorded configuration.

**Camera GenICam configuration verification.** When the active system records per-camera GenICam configuration
paths (a standard practice for GenTL/GenICam cameras), verify the live cameras against their stored
configurations after discovery: call `verify_camera_configuration_tool` (sle), which dumps each camera's live
GenICam node configuration and diffs it against the stored YAML (reporting `match`, identity match, and per-node
`value_mismatches`). On a mismatch, either restore the known-good configuration onto the camera
(`ataraxis@video:camera-setup`'s `load_genicam_config`), or — if the live configuration is the new desired
baseline — dump it to the stored path (`dump_genicam_config`). The stored paths belong to the active system's
configuration; read them via that system's skill (for `mesoscope`, `/mesoscope-vr` →
`cameras.<role>_camera_configuration_path`). Cameras with no path declared are skipped.

**Device path convention:**

On Linux, microcontrollers enumerate as `/dev/ttyACM*` (USB CDC ACM) and Zaber motors as `/dev/ttyUSB*`
(USB serial adapters). On Windows and macOS the path format differs (e.g. `COMx` on Windows), but the
CDC-ACM (microcontroller) vs USB-serial (Zaber) device-type distinction still holds.

Do not confuse these device types when reporting discovered hardware.

### Phase 3: Report and hand off

After discovery completes, report the discovered hardware to the user as a structured table.

**If the user is performing initial bringup**, hand off in this order (owning plugin named per step):

1. assets plugin `/working-directory` — set the working directory (the only universally required item).
   Also set the Google credentials path, but only if the system reads animal metadata from Google Sheets,
   and the task templates directory, but only if the system runs Unity VR tasks; both are otherwise optional.
2. the active acquisition system's skill (this plugin's `/mesoscope-vr` for the `mesoscope` system) —
   author the host machine's system configuration YAML against the discovered hardware values.
3. assets plugin `/project-hierarchy` — create the project (or projects) the host will record under.
4. assets plugin `/task-templates` — author or import the task templates the project will use.
5. assets plugin `/experiment-configuration` — author the per-project experiment configuration that
   wires a template to a project.

Steps 4–5 (and the task templates directory in step 1) apply only to systems that run Unity VR tasks, such
as `mesoscope`; a non-VR acquisition system skips them.

**If the user is performing verification**, hand off to the active acquisition system's skill
(`/mesoscope-vr` for the `mesoscope` system) for a read-only fetch of the recorded values, then report
the diff between discovered and recorded.

**If the user is troubleshooting**, use the troubleshooting table below.

You MUST NOT call `set_working_directory_tool`, `set_google_credentials_tool`,
`set_task_templates_directory_tool`, `write_system_configuration_tool`, or
`write_server_configuration_tool` directly under any circumstances.

---

## Troubleshooting

| Error                                  | Cause                                  | Solution                                                                        |
|----------------------------------------|----------------------------------------|---------------------------------------------------------------------------------|
| Camera not found at expected index     | Wrong camera index                     | Re-run `list_cameras()`, hand off to the active system's skill                  |
| Microcontroller connection failed      | Wrong port or disconnected             | Re-run `list_microcontrollers()`, check USB cables                              |
| Zaber motor not responding             | Wrong port or powered off              | Re-run `get_zaber_devices_tool()`, verify power supply                          |
| MQTT broker unreachable                | Broker not running                     | Start Mosquitto or the configured MQTT broker                                   |
| Unity bridge unreachable               | Unity Editor not open                  | Open the Unity project in the editor; its MCP bridge auto-starts                |
| FFMPEG not found                       | FFMPEG not installed                   | Install FFMPEG via the OS package manager                                       |
| GPU not detected                       | NVIDIA driver missing                  | Install NVIDIA driver and restart                                               |
| CTI file not configured                | GenTL producer not registered          | Hand off to `ataraxis@video:camera-setup` to register the CTI file              |
| Live camera config differs from stored | Camera drifted or reconfigured         | Restore via `load_genicam_config`, or re-baseline via `dump_genicam_config`     |
| Stored camera config file not found    | Declared path points at a missing file | Dump a baseline with `dump_genicam_config`, or fix the path via `/mesoscope-vr` |

For configuration-file-level errors (working directory not set, schema validation failures, missing projects),
hand off to the assets plugin skill that owns the affected asset.

---

## Verification checklist

```text
- [ ] Any declared network storage mounts verified via check_system_mounts_tool() (skip if the system declares none)
- [ ] Required MCP servers (ataraxis video, ataraxis comm, sollertia-experiment) confirmed reachable
- [ ] check_runtime_requirements() reported FFMPEG and GPU OK
- [ ] CTI file status confirmed (if using Harvesters cameras)
- [ ] check_mqtt_broker() reported broker reachable
- [ ] check_unity_bridge_tool() reported the Unity Editor bridge reachable (only for systems driving a Unity VR task)
- [ ] list_cameras() returned the expected cameras
- [ ] Camera GenICam configs verified against stored configs via verify_camera_configuration_tool() (if the system declares config paths)
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
