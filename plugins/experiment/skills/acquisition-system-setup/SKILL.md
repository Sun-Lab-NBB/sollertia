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

Discovers, verifies, and reports the hardware connected to a Sollertia acquisition PC. Focuses exclusively on hardware
introspection. Configuration file authoring, working directory setup, credential management, and project and experiment
creation are owned by sibling plugins, named per item in the hand-off list below, and must be invoked by hand-off.

---

## Scope

**Covers:**
- Discovering connected cameras (OpenCV and Harvesters / GenICam)
- Discovering connected microcontrollers (USB CDC ACM)
- Discovering connected Zaber motors (USB serial)
- Verifying MQTT broker reachability
- Verifying Unity Editor MCP bridge reachability
- Verifying video runtime requirements (FFMPEG, GPU, CTI file)
- Verifying any declared network storage mounts at the OS level
- Reporting discrepancies between discovered hardware and the active system configuration

**Does not cover:**
- Setting the working directory, data root, credentials, or task templates directory → `assets:working-directory`
- Reading, writing, or validating system configuration YAML → the active acquisition system's skill
  (`mesoscope:mesoscope-vr` for the current worked example)
- Verifying an out-of-process tool environment the active system declares → that same system's skill
- Extending the platform with a new acquisition system → `/library-extension`
- Creating projects → `assets:project-hierarchy`
- Authoring task templates → `assets:task-templates`
- Authoring per-project experiment configurations → `assets:experiment-configuration`
- Reading the `SessionData` marker file → `assets:session-data`
- Reading session descriptors → `assets:session-descriptors`
- Reading frozen runtime snapshots → `mesoscope:mesoscope-vr-snapshots`
- Reading subject metadata → `assets:data-assets`
- Reading or inspecting forged datasets → `assets:datasets`
- Composing or growing a dataset → `forging:dataset-definition`

You MUST NOT call any `slsa` MCP tool that mutates state. Read-only `read_*` and `discover_*` tools from the assets
plugin's MCP server may be called as a "natural share" only when verifying that discovered hardware matches the
recorded configuration.

---

## MCP server requirements

This skill uses MCP tools from libraries other than `sollertia-shared-assets`. The `slsa mcp` and `slf mcp` servers
are required only because hand-off targets in the assets and forging plugins depend on them.

| Server                           | CLI Command | Used directly by this skill | Purpose                                                      |
|----------------------------------|-------------|-----------------------------|--------------------------------------------------------------|
| ataraxis-video-system            | `axvs mcp`  | yes                         | Camera discovery, runtime requirements, CTI                  |
| ataraxis-communication-interface | `axci mcp`  | yes                         | Microcontroller discovery, MQTT broker check                 |
| sollertia-experiment             | `sle mcp`   | yes                         | Zaber motor discovery, storage mount checks                  |
| sollertia-shared-assets          | `slsa mcp`  | read-only only              | Read-only "natural share" reads during hardware verification |
| sollertia-forgery                | `slf mcp`   | no (hand-off targets only)  | Server configuration and dataset definition hand-offs        |

If a required MCP server is unavailable, hand off to the appropriate plugin's MCP environment setup skill:
`video:video-mcp-environment-setup` (ataraxis marketplace), `communication:communication-mcp-environment-setup`, or
this plugin's `/experiment-mcp-environment-setup`.

### The `sle get` CLI and the agnostic MCP tools

`sle get` carries six commands: `zaber`, `cameras`, `controllers`, `ports`, `unity`, and `checksum`, all registered on
the `get` Click group in `interfaces/get.py`. `sle get controllers` scans every serial port at
`_MICROCONTROLLER_BAUDRATE = 115200` (`interfaces/get.py`).

The `sle mcp` server exposes seven hardware-agnostic tools, the `@mcp.tool()` functions of `interfaces/get_tools.py`.
The CLI commands and those tools do NOT mirror each other. `cameras`, `controllers`, and `ports` have no MCP tool, and
`get_zaber_device_settings_tool`, `set_zaber_device_setting_tool`, `validate_zaber_configuration_tool`, and
`check_mount_accessibility_tool` have no `sle get` command. You MUST NOT infer a tool name from a command name.

All seven agnostic tools return a plain string and report failure with a leading `Error: ` prefix. The single
exception to the `Error:` return convention is `check_unity_bridge_tool`, which never returns an `Error:` string.
`check_mount_accessibility_tool` reports an unreachable path as an `OK: False` status line instead, reserving the
leading prefix for a rejected argument. Those two are also the only agnostic tools that carry no `except` handler. The
mount tool's write probe is the exception, because `probe_writable` catches `OSError` itself and reports it as the
trailing `Error:` field, so only a non-`OSError` raised inside either tool propagates across the MCP boundary
(`interfaces/get_tools.py`, `interfaces/mcp_instance.py`).

---

## Supported acquisition systems

The system value column holds the `AcquisitionSystems` member value, which is also the `<system>` segment of the
`<system>_system_configuration.yaml` filename (`sollertia-shared-assets/src/sollertia_shared_assets/enums.py`).

| System value | Description                                                      | Owning schema skill      |
|--------------|------------------------------------------------------------------|--------------------------|
| `mesoscope`  | The current worked example, and the only registered system today | `mesoscope:mesoscope-vr` |

`AcquisitionSystems` currently holds exactly one member, `MESOSCOPE_VR` (`enums.py`), so this table currently holds one
row. A new acquisition system MUST add its row here, and `/library-extension` names this table as a required deliverable
of the registration workflow. Other skills route through this table to resolve a system value to its owning skill, so a
missing row leaves the system unreachable from the pipeline skills. Hand off to the owning skill for the canonical field
schema, which this skill does not duplicate.

---

## Network storage prerequisites

Network storage locations are optional. An acquisition system is not required to define any, and it can operate
entirely on local storage. When a system does define network storage locations, every one of them must be reachable
through a direct-filesystem mount (SMB or an equivalent protocol that exposes the share as a local path) before any
configuration work begins. Establishing these mounts is an operating-system-level task, and no skill or MCP tool
creates them. Verifying that the declared mounts are reachable, by contrast, is available through the
`sollertia-experiment` (`sle mcp`) server's mount tools.

### Long-term storage mounts

A system declares its long-term storage destinations as named entries in its own configuration. An entry left as an
empty path reads as not configured and is skipped, so only the destinations a system actually declares need to be
mounted. Mapping order is the preference order used when data is pulled back, so the fastest source to pull from belongs
first. The shared preprocessing utilities never see the configuration itself: a system resolves its own destinations and
hands over `StorageDestination(name, session_path)` entries collected into a `StorageDestinations` container, with both
dataclasses defined in `cross_system/data_preprocessing.py`. For the concrete destination names of the current worked
example, see `mesoscope:mesoscope-vr`.

These locations are essentially one-way egress targets. The acquisition PC pushes acquired data to them and clears it
from local disk. Data returns only during a deliberate cross-infrastructure migration, for example when moving an
animal between projects, which is why the mapping order defines pull-back preference.

### Within-system shares

Within-system shares connect the separate PCs that together make up one acquisition system. Unlike long-term storage
locations, which are one-way egress targets, a within-system share has no fixed direction, so flow may be
unidirectional or bidirectional. Typically, every PC aggregates its data onto the main acquisition PC before the data
is pushed to the long-term storage destinations, but the main PC can also write back to a peer. A within-system share
connects peer PCs and moves data only. For the within-system shares the current worked example declares, see
`mesoscope:mesoscope-vr`.

### Mount configuration

For every network storage location the system declares, ensure before invoking this skill:

1. The share is mounted and accessible from the acquisition PC.
2. The mount point has appropriate read/write permissions.
3. The mount persists across reboots (configured via the OS-appropriate mechanism, for example `/etc/fstab` or
   systemd mount units on Linux).

To verify that the declared mounts are reachable, use the `sle` MCP server. These tools check the paths, and they do
not create the mounts:

```text
<the active system's mount-sweep tool>    # sweeps the data root and every path the active configuration declares
check_mount_accessibility_tool(path=...) # drills into a failed storage or data root, not a read-only input file
```

The mount-sweep tool reads the active system configuration, so it belongs to the active system's own tool group. For the
current worked example, see `mesoscope:mesoscope-vr`. Resolve that system first, then call the sweep tool that system's
skill names. The sweep reports the platform data root alongside the system's own paths and counts it in the returned
`ok` and `failed` tallies. An unset or unreachable data root therefore fails the sweep even when every declared mount is
healthy. Hand off to `assets:working-directory` to set the data root, then re-run the sweep.

`check_mount_accessibility_tool` is agnostic and takes a single path. It rejects an empty or relative path outright and
demands an absolute one. For a path that exists it reports `Exists`, `Mount`, `Writable`, and `OK`, adding an `Error`
segment when the write probe fails (`interfaces/get_tools.py`).

The OS-level equivalent is a direct listing (Linux example, use the OS-appropriate command on Windows and macOS):

```bash
ls /mnt/<destination_name>   # one listing per declared storage destination
ls /mnt/<peer_share_name>    # one listing per declared within-system share
```

If a declared mount is not reachable, coordinate with system administrators to set up the SMB share (or an equivalent
direct-filesystem-access protocol), because no skill or MCP tool can create the mount for you.

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

1. Run hardware discovery (Phases 1 and 2 below).
2. Hand off to the active acquisition system's skill for a read-only read of the recorded configuration values,
   because that system owns its configuration reader tool.
3. Compare discovered values against recorded values and report any drift to the user.
4. If drift exists, hand off to that same system skill to update the recorded values. Do not edit YAML and do not call
   a configuration writer from this skill.

---

## Hardware discovery workflow

Each tool below is owned by one of the three discovery MCP servers. This skill invokes them but hands the canonical
usage detail to the owner skill named after each table. Every server's MCP tools carry a `_tool` suffix.

### Phase 1: Runtime prerequisites

| Tool                              | Server | Purpose                                                       |
|-----------------------------------|--------|---------------------------------------------------------------|
| `check_runtime_requirements_tool` | axvs   | FFMPEG, GPU, and CTI file status                              |
| `get_cti_status_tool`             | axvs   | CTI (.cti) path, "CTI: Not configured", or "CTI: Unavailable" |
| `set_cti_file_tool`               | axvs   | Sets the .cti path (Harvesters)                               |
| `check_mqtt_broker_tool`          | axci   | MQTT broker reachability (host, port)                         |
| `check_unity_bridge_tool`         | sle    | Unity Editor MCP Bridge reachability                          |

`check_runtime_requirements_tool`, `get_cti_status_tool`, and `set_cti_file_tool` are owned by `video:camera-setup`.
`check_mqtt_broker_tool` is owned by `communication:microcontroller-setup`. `check_unity_bridge_tool` is owned by
`/vr-driver-interface`.

Invoke `check_runtime_requirements_tool` first. If it reports the CTI file as unconfigured and the system uses
Harvesters cameras, set the path with `set_cti_file_tool`. The CTI path lives in the video MCP server's state rather
than slsa state, so this skill may call it. See `video:camera-setup` for the canonical CTI workflow. Then invoke
`check_mqtt_broker_tool`. If the broker is unreachable, instruct the user to start their broker service (for example
Mosquitto) before continuing. Then invoke `check_unity_bridge_tool` (CLI: `sle get unity`). If it reports the bridge
unreachable, instruct the user to open the Unity project in the editor, whose MCP bridge auto-starts, before running a
session that runs the corridor task. If the Editor is already open and the bridge is still unreachable, see
`unity:unity-mcp-environment-setup` for the Editor-side listener diagnostic.

**Out-of-process tool environments.** When the active system configuration declares an out-of-process tool, confirm
that tool's environment through that system's skill, because the environment name sits outside the mount report and
needs a manual check.

### Phase 2: Hardware discovery

| Tool                         | Server | Discovers                          |
|------------------------------|--------|------------------------------------|
| `list_cameras_tool`          | axvs   | Camera index, model, resolution    |
| `list_microcontrollers_tool` | axci   | Port path + microcontroller ID     |
| `get_zaber_devices_tool`     | sle    | Port path, device name, axis count |

`list_cameras_tool` is owned by `video:camera-setup`, `list_microcontrollers_tool` by
`communication:microcontroller-setup`, and `get_zaber_devices_tool` by this plugin's `/zaber-interface`.

Invoke each tool and record what it returns. Mapping the discovered IDs and motor layout to fixed hardware roles is
system-specific, so hand off to the active acquisition system's skill for the canonical mapping. For the current worked
example, see `mesoscope:mesoscope-vr`.

**Camera GenICam configuration verification.** When the active system records per-camera GenICam configuration paths
(a standard practice for GenTL/GenICam cameras), verify the live cameras against their stored configurations after
discovery. The verification tool reads the active system configuration, so it belongs to that system's tool group and
this skill hands the call off. On a reported mismatch, either restore the known-good configuration onto the camera
(`video:camera-setup`'s `load_genicam_config_tool`), or, if the live configuration is the new desired baseline, dump it
to the stored path (`dump_genicam_config_tool`). Cameras with no configuration path declared are skipped.

**Device path convention:**

On Linux, microcontrollers enumerate as `/dev/ttyACM*` (USB CDC ACM) and Zaber motors as `/dev/ttyUSB*` (USB serial
adapters). On Windows and macOS the path format differs (for example `COMx` on Windows), but the CDC-ACM
(microcontroller) versus USB-serial (Zaber) device-type distinction still holds.

Do not confuse these device types when reporting discovered hardware.

### Phase 3: Report and hand off

After discovery completes, report the discovered hardware to the user as a structured table.

**If the user is performing initial bringup**, hand off in this order (owning plugin named per step):

1. `assets:working-directory` sets the working directory, the data root, and the task templates directory. Also
   configure the `google` category credentials, but only if the system reads animal metadata from Google Sheets.
2. The active acquisition system's skill (`mesoscope:mesoscope-vr` for the current worked example) authors the host
   machine's system configuration YAML against the discovered hardware values.
3. `assets:project-hierarchy` creates the project (or projects) the host will record under.
4. `assets:task-templates` authors or imports the task templates the project will use.
5. `assets:experiment-configuration` authors the per-project experiment configuration that wires a template to a
   project.

**If the user is performing verification**, hand off to the active acquisition system's skill for a read-only fetch of
the recorded values, then report the diff between discovered and recorded.

**If the user is troubleshooting**, use the troubleshooting table below.

You MUST NOT call `set_working_directory_tool`, `set_credentials_tool`, `set_task_templates_directory_tool`, the active
system's system-configuration write tool, or `write_server_configuration_tool` directly under any circumstances. The
system-configuration write tool is owned exclusively by the active system's skill, `mesoscope:mesoscope-vr` for the
current worked example, and `write_server_configuration_tool` is owned exclusively by `forging:server-configuration`
on the `slf mcp` server.

---

## Troubleshooting

| Error                                  | Cause                                  | Solution                                                                              |
|----------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------|
| Camera not found at expected index     | Wrong camera index                     | Re-run `list_cameras_tool()`, hand off to the active system's skill                   |
| Microcontroller connection failed      | Wrong port or disconnected             | Re-run `list_microcontrollers_tool()`, check USB cables                               |
| Zaber motor not responding             | Wrong port or powered off              | Re-run `get_zaber_devices_tool()`, verify power supply                                |
| MQTT broker unreachable                | Broker not running                     | Start Mosquitto or the configured MQTT broker                                         |
| Unity bridge unreachable               | Unity Editor not open                  | Open the Unity project in the editor, whose MCP bridge auto-starts                    |
| FFMPEG not found                       | FFMPEG not installed                   | Install FFMPEG via the OS package manager                                             |
| GPU not detected                       | NVIDIA driver missing                  | Install NVIDIA driver and restart                                                     |
| CTI file not configured                | GenTL producer not registered          | Hand off to `video:camera-setup` to register the CTI file                             |
| Out-of-process tool fails to start     | Declared environment or path missing   | Create the environment, or fix the path through the active system's skill             |
| Live camera config differs from stored | Camera drifted or reconfigured         | Restore via `load_genicam_config_tool`, or re-baseline via `dump_genicam_config_tool` |
| Stored camera config file not found    | Declared path points at a missing file | Dump a baseline with `dump_genicam_config_tool`, or fix the path through that skill   |

For configuration-file-level errors (working directory not set, schema validation failures, missing projects), hand off
to the assets plugin skill that owns the affected asset.

---

## Related skills

Entries prefixed `video:` and `communication:` resolve through the ataraxis marketplace.

| Skill                                               | Relationship                                                               |
|-----------------------------------------------------|----------------------------------------------------------------------------|
| `assets:working-directory`                          | Owns bootstrap state (working dir, credentials, templates dir)             |
| `mesoscope:mesoscope-vr`                            | Owns the current worked example's configuration authoring and validation   |
| `/library-extension`                                | Owns the seam catalog and the registration deliverables of a new system    |
| `forging:server-configuration`                      | Owns `ServerConfiguration` authoring and validation                        |
| `assets:project-hierarchy`                          | Owns project creation (`create_project_tool`)                              |
| `assets:task-templates`                             | Owns task template authoring                                               |
| `assets:experiment-configuration`                   | Owns per-project experiment configuration authoring                        |
| `/system-health-check`                              | Lighter-weight pre-session verification sweep                              |
| `/pipeline`                                         | Phase 3 (Hardware bringup) is owned by this skill                          |
| `/vr-driver-interface`                              | Owns the `check_unity_bridge_tool` contract this skill calls in Phase 1    |
| `/zaber-interface`                                  | Owns `get_zaber_devices_tool` and the per-device Zaber semantics           |
| `video:camera-setup`                                | Canonical home for CTI configuration and runtime requirement deep-dives    |
| `video:video-mcp-environment-setup`                 | Diagnoses an unreachable `axvs mcp` server                                 |
| `communication:microcontroller-setup`               | Canonical home for microcontroller manifest and discovery deep-dives       |
| `communication:communication-mcp-environment-setup` | Diagnoses an unreachable `axci mcp` server                                 |
| `unity:unity-mcp-environment-setup`                 | Editor-side McpBridge listener diagnostic behind `check_unity_bridge_tool` |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Prerequisites:
- [ ] Required MCP servers (ataraxis video, ataraxis comm, sollertia-experiment) confirmed reachable
- [ ] Active acquisition system resolved to its owning skill through the supported-systems table
- [ ] The active system's mount sweep reported the platform data root and every declared mount reachable (run after
      the system configuration exists, which on a fresh bringup follows Phase 3 step 2)

Runtime prerequisites:
- [ ] check_runtime_requirements_tool() reported FFMPEG and GPU OK
- [ ] CTI file status confirmed (if using Harvesters cameras)
- [ ] check_mqtt_broker_tool() reported broker reachable
- [ ] check_unity_bridge_tool() reported the Unity Editor bridge reachable
- [ ] Any out-of-process tool environment the active system declares confirmed through that system's skill

Discovery:
- [ ] list_cameras_tool() returned the expected cameras
- [ ] Camera GenICam configs verified against stored configs through the active system's skill (if paths are declared)
- [ ] list_microcontrollers_tool() returned the expected microcontrollers and roles
- [ ] get_zaber_devices_tool() returned the expected motor groups
- [ ] Discovered hardware reported to user as a structured table

Boundaries:
- [ ] Did NOT call any slsa setter tool from this skill
- [ ] Handed off every state mutation to assets:working-directory, the active system's skill,
      forging:server-configuration, assets:project-hierarchy, assets:task-templates, or
      assets:experiment-configuration
```
