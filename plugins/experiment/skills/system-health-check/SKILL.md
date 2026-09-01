---
name: system-health-check
description: >-
  Comprehensive pre-flight verification for acquisition systems. Orchestrates MCP tools and skill hand-offs to check
  platform configuration readiness, network storage mounts, hardware connectivity, and configuration validity. Use
  before running acquisition sessions or when troubleshooting system issues.
user-invocable: true
---

# System health check

Orchestrates the `sle mcp` server, the assets plugin's `slsa mcp` server, and hardware-discovery hand-offs to validate
that a host is ready to run a session. This is the lighter-weight pre-session sweep, and full bringup discovery is
owned by `/acquisition-system-setup`.

---

## Scope

**Covers:**
- Pre-session readiness verification: active-system identification, platform-configuration prerequisites,
  network-storage mounts, hardware-connectivity sweep, and configuration validity
- Orchestrating `sle mcp` and `slsa mcp` read-only checks and handing off hardware discovery to the active
  system's skill

**Does not cover:**
- Full system bringup and hardware-discovery deep-dives (see `/acquisition-system-setup`)
- Writing or repairing configuration, since this skill is read-only. Fixes are owned by `assets:working-directory`
  and by the active system's skill
- Per-system hardware mappings or expected device values (owned by `/acquisition-system-setup`)
- The seams a new acquisition system must fill before it can be health-checked (owned by
  `/library-extension`)

---

## MCP server requirements

| Server                             | CLI command | Used for                                                                          |
|------------------------------------|-------------|-----------------------------------------------------------------------------------|
| `sollertia-experiment`             | `sle mcp`   | Active-system identification, mount checks, configuration and Zaber validation    |
| `sollertia-shared-assets`          | `slsa mcp`  | Platform configuration status snapshot, supported-system vocabulary               |
| `ataraxis-video-system`            | `axvs mcp`  | Camera discovery and video requirements (via `/acquisition-system-setup`)         |
| `ataraxis-communication-interface` | `axci mcp`  | Microcontroller discovery and MQTT broker check (via `/acquisition-system-setup`) |

If a required server is unavailable, hand off to the owning plugin's MCP environment setup skill:
`/experiment-mcp-environment-setup`, `assets:assets-mcp-environment-setup`, `video:video-mcp-environment-setup`
(ataraxis marketplace), or `communication:communication-mcp-environment-setup`.

### The `sle` surfaces this skill draws on

`/acquisition-system-setup` owns the `sle get` command set and the way it diverges from the seven hardware-agnostic
MCP tools of `interfaces/get_tools.py`, so read that skill before mapping a command name onto a tool name.

Every agnostic tool returns a plain string, and most report failure with a leading `Error: ` prefix. Two deviate from
that convention, and `/acquisition-system-setup` owns the divergence, so read it before treating a non-`Error:` return
as a fault.

Every other tool this skill calls belongs to the active system's own tool module, `interfaces/<system>_tools.py`, whose
tools register as an import side effect of the `*_tools.py` glob run by `_register_tool_modules()` in
`interfaces/mcp_server.py`. Phase 0 resolves which module that is, and the active system's skill names the tools inside
it.

---

## Complete verification workflow

```text
System health check progress:
- [ ] Phase 0: Active acquisition system identified and resolved to its skill
- [ ] Phase 1: Platform configuration prerequisites verified
- [ ] Phase 2: Network storage mounts accessible
- [ ] Phase 3: Hardware connectivity confirmed
- [ ] Phase 4: Configuration validity confirmed
```

### Phase 0: Identify the active acquisition system

Every later phase hands off to "the active system's skill" for system-specific expectations and for the tools that
read that system's configuration, so resolve that skill first and never assume a particular system.

```text
get_platform_environment_status_tool()    # slsa: locates the working directory
list_supported_acquisition_systems_tool() # slsa: the AcquisitionSystems vocabulary of supported types
```

1. Locate the working directory through `get_platform_environment_status_tool()`. Its `configuration` subdirectory
   holds exactly one system configuration file on a configured host, named `<system>_system_configuration.yaml`
   composed by `_system_configuration_filename()` in `cross_system/system_configuration.py`. A directory holding no
   such file means no system is set up on this host: stop the health check and hand off to `/acquisition-system-setup`
   for bringup and to `assets:working-directory` for the platform prerequisites.
2. Read the active system's **type** from that filename. The `<system>` segment is the `AcquisitionSystems` value,
   confirmable against `list_supported_acquisition_systems_tool()`. The configuration payload itself carries no
   acquisition-system field, so the filename is the authoritative type discriminator. Do NOT key off the
   configuration's `name` field, which is a free-form human-readable label that may be customized and is not a
   reliable type discriminator even at its default value.
3. Resolve that type to its owning skill through `/acquisition-system-setup`'s **Supported acquisition systems**
   registry, which this skill does not duplicate. The skill name is not derivable from the type, so read the table.
   The phases below refer to the resolved skill as "the active system's skill."

A system's configuration loader verifies that the host belongs to that system and raises `TypeError` when it does not,
and its tools catch that exception and return it as an `{"error": ...}` payload rather than letting it cross the MCP
boundary. An `error` payload naming a different acquisition system therefore identifies the host as belonging to that
other system. The current worked example implements this check in `get_system_configuration()`
(`mesoscope_vr/system.py`), and its tools wrap the loader in `try`/`except Exception`
(`interfaces/mesoscope_vr_tools.py`).

### Phase 1: Platform configuration prerequisites

Take a single-call snapshot of every Sollertia platform configuration component owned by `sollertia-shared-assets`:

```text
get_platform_environment_status_tool()
```

This read-only `slsa` tool reports the readiness of the working directory, data root, task-templates directory, and
one `<category>_credentials` component per supported credentials category (currently `google_credentials`). Only the
working directory is required for `slsa mcp` to function. The others are optional and gate only the workflows that use
them, so the tool's `overall_ok` reflects the required components only. System-configuration validity is reported in
Phase 4 instead. If every required component reports healthy, advance to Phase 2. When a component reports unhealthy,
hand off to `assets:working-directory`, which owns the working directory, data root, credentials, and templates
directory. This skill never writes configuration. `assets:session-data` owns the session and project hierarchy that
the data root anchors.

### Phase 2: Network storage mounts

Run the active system's mount sweep, which the active system's skill names. The sweep reads the active configuration and
validates every network storage location that configuration declares, both the within-system shares and each configured
long-term storage destination. A system may declare none and run entirely on local storage, in which case this phase has
nothing to verify. The sweep also reports the platform data root and the read-only input files the configuration
declares, and counts both in its own tallies, so an unset data root fails the sweep even when every declared mount is
healthy.

For a path that fails, drill in with the agnostic `check_mount_accessibility_tool(path=...)`. Apply it only to a
directory the sweep reports. It rejects an empty or relative path outright and demands an absolute one
(`interfaces/get_tools.py`):

- `Exists: False` means the path does not exist, so the OS-level mount is not configured. Configure it through the
  host OS's persistent-mount mechanism (see the Mount failures troubleshooting table).
- `Mount: False` means the path exists but is not a mount point, so a local directory may be in use instead of
  network storage.
- `Writable: False` means the write probe failed, and the reported `Error` segment carries the underlying OS error.
  Check share permissions and the mount's ownership and permission options.

The read-only input files the sweep covers, each stored camera GenICam configuration `.yaml` and the DeepLabCut project,
are checked for existence and read access instead of being write-probed. `check_mount_accessibility_tool` write-probes
by creating a temporary file inside the path it is given, so pointing it at one of those files reports a failure for a
perfectly healthy file:

```text
Writable: False | OK: False | Error: [Errno 20] Not a directory: <file>/.sollertia_experiment_probe_...
```

Diagnose a failing input file from the sweep's own `error` segment, which reads `File does not exist` or `Not readable`,
rather than with this tool.

### Phase 3: Hardware connectivity

| Check                     | Tool                              | Server | Expected result                      |
|---------------------------|-----------------------------------|--------|--------------------------------------|
| Cameras detected          | `list_cameras_tool`               | `axvs` | Expected camera indices              |
| Video runtime ready       | `check_runtime_requirements_tool` | `axvs` | FFMPEG available, GPU detected       |
| CTI file (Harvesters)     | `get_cti_status_tool`             | `axvs` | CTI configured (if using GenICam)    |
| Microcontrollers detected | `list_microcontrollers_tool`      | `axci` | Expected microcontroller IDs / roles |
| MQTT broker reachable     | `check_mqtt_broker_tool`          | `axci` | Connection successful                |
| Zaber devices detected    | `get_zaber_devices_tool`          | `sle`  | Expected Zaber devices (if any)      |

The tools above are domain-general discovery utilities, system-agnostic across Sollertia acquisition systems. Cameras
and microcontrollers come from the `axvs` and `axci` dependency servers, and Zaber discovery is an agnostic `sle`
tool. Which of these a given system composes, and the values each should return, is system-specific and is not
enumerated here. Hand off to `/acquisition-system-setup` for the per-system hardware mapping and the discovery and
troubleshooting semantics, and to `/zaber-interface` for the per-device Zaber semantics. This skill calls these tools
read-only as a pre-flight sweep.

For a session whose type runs the corridor task, additionally confirm the shared Unity Editor MCP Bridge is reachable
with `check_unity_bridge_tool` (`sle`, CLI `sle get unity`). Unity is a shared asset with its own driver rather than a
per-system one. The bridge starts automatically inside the Unity Editor, so an unreachable bridge means the editor is
not open and the runtime can neither open the scene nor arm the VR task. The session types that run the task are the
members of `SESSION_TYPES_USING_VR_TASK` (`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`), and
every other session type skips this check. Hand off to `/vr-driver-interface` for the bridge contract.

If the active system drives a system-specific instrument control interface beyond the domain-general stack and the
shared Unity bridge, additionally confirm that interface is reachable. Its check tool, expected state, and remediation
are owned by the active system's skill, so hand off rather than assuming a tool here. For the current worked example,
see `mesoscope:mesoscope-vr`.

### Phase 4: Configuration validity

The system configuration validator and the camera configuration verifier both read the active configuration, so both
belong to the active system's tool group and the active system's skill names them. Run the pair that Phase 0 resolved,
and expect an `{"error": ...}` payload carrying the loader's `TypeError` message if the host turns out to belong to a
different acquisition system. A validator reports whether the configuration is internally valid and whether every path
it declares resolves on the live filesystem. A camera verifier dumps each declared camera's live GenICam node
configuration and diffs it against the stored YAML. On a mismatch, hand off to `/acquisition-system-setup` to restore
or re-baseline, and to `video:camera-setup` for the GenICam dump and restore mechanics.

Each third-party-SDK subsystem additionally exposes its own device-level validator.
`validate_zaber_configuration_tool(port, device_index)` sits in the agnostic tool group and is shared across
acquisition systems, validating one Zaber device against the settings the binding library requires. It returns
`Status: VALID|INVALID | Checksum: OK|FAIL | Positions: OK|FAIL`, plus `Errors:` and `Warnings:` segments when either
is present (`interfaces/get_tools.py`). Which subsystems the active system composes, and the ports, device
indices, and expected settings each should report, are system-specific, so hand off to the active system's skill and
to `/zaber-interface` for the per-device Zaber semantics. A system that composes no such subsystem has nothing further
to validate here.

For a session that is about to be recorded, confirm the raw assets that session will be required to carry.
`SessionData.required_raw_assets()` is the single source of truth
(`sollertia-shared-assets/src/sollertia_shared_assets/data_hierarchy/session_data.py`). Every session requires the
session descriptor and the system configuration snapshot. An experiment session additionally requires the experiment
configuration snapshot, and a session whose type sits in `SESSION_TYPES_USING_VR_TASK` additionally requires the
`vr_configuration.yaml` task template snapshot. That snapshot is copied out of the task templates directory at session
creation, so an unset templates directory blocks a corridor-task session even when every mount is healthy. Hand off to
`assets:library-extension` for the required-asset policy, to `assets:session-data` for the session hierarchy, and to
`assets:project-hierarchy` to confirm the recording project exists, because `sle mcp` carries no project-listing tool.

---

## Quick health check

For a rapid pre-session check:

1. `get_platform_environment_status_tool()` reports the platform configuration healthy.
2. The active system's mount sweep reports all storage accessible.
3. `/acquisition-system-setup` confirms the hardware the active system declares is present, covering cameras,
   microcontrollers, any third-party-SDK subsystems such as Zaber motors, and the MQTT broker.
4. The active system's configuration validator reports the configuration valid.
5. For a session whose type runs the corridor task, `check_unity_bridge_tool()` reports the Unity Editor open and its
   MCP bridge reachable.
6. For a session that drives a system-specific instrument control interface, confirm it through the active system's
   skill.

If all pass, the system is ready for acquisition.

---

## Troubleshooting guide

### Mount failures

| Symptom                | Likely cause         | Resolution                                                     |
|------------------------|----------------------|----------------------------------------------------------------|
| Path does not exist    | Mount not configured | Configure a persistent mount via the host OS's mount mechanism |
| Exists but not a mount | Local directory used | Query the host OS's mount table to confirm the path is a mount |
| Not writable           | Permission issue     | Check share permissions and ownership/permission mount options |
| Stale mount            | Network disruption   | Unmount and remount the share with the host OS's mount tooling |
| Input file unusable    | Missing or no access | Fix the configured path or read permissions, not the mount     |

The symptoms and likely causes are OS-independent, and the resolution mechanics are OS-specific. Resolve the concrete
commands for the host OS before suggesting fixes, for example `/etc/fstab` entries or systemd mount units and
`mount`/`umount` on Linux, mapped network drives (`net use`) on Windows, and `mount`/automount or Finder's "Connect to
Server" on macOS.

### Hardware not detected and MQTT failures

Delegate to `/acquisition-system-setup`'s troubleshooting tables, which own the discovery tooling and the camera,
microcontroller, Zaber, and MQTT failure modes.

---

## Post-check actions

1. **All checks pass.** The system is ready for session execution.
2. **Configuration prerequisite unhealthy.** Hand off to `assets:working-directory`.
3. **Mount failures.** Resolve OS-level mount issues before proceeding.
4. **Hardware missing.** Hand off to `/acquisition-system-setup`.
5. **Configuration invalid.** Hand off to the active acquisition system's skill to correct the system configuration.
6. **Unity bridge unreachable**, for a session type that runs the corridor task. Open the Unity project in the editor
   so its MCP bridge auto-starts, and confirm with `sle get unity`. Hand off to `/vr-driver-interface`, or to
   `unity:unity-mcp-environment-setup` when the Editor is already open and the listener is still unreachable.
7. **Missing required raw asset.** Hand off to `assets:library-extension` for the required-asset policy, and to
   `assets:working-directory` when the task templates directory is the blocker.
8. **System-specific control interface unreachable.** Hand off to the active system's skill to bring it up.

---

## Related skills

Entries prefixed `video:` and `communication:` resolve through the ataraxis marketplace.

| Skill                                 | Relationship                                                                          |
|---------------------------------------|---------------------------------------------------------------------------------------|
| `/cli-reference`                      | Reference: the `sle get` commands behind these hardware checks                        |
| `/acquisition-system-setup`           | Owns the full hardware-discovery sweep and the supported-systems registry             |
| `mesoscope:mesoscope-vr`              | The current worked example's skill, resolved by Phase 0 on a host running that system |
| `/library-extension`                  | Owns the seams a new acquisition system fills before this sweep can resolve it        |
| `/vr-driver-interface`                | Owns the shared Unity editor bridge check (`check_unity_bridge_tool`)                 |
| `unity:unity-mcp-environment-setup`   | Editor-side McpBridge listener diagnostic when the Editor is open but unreachable     |
| `/experiment-mcp-environment-setup`   | Run first if the `sle mcp` server is not connected                                    |
| `/pipeline`                           | Phase 5 (pre-session health check) is owned by this skill                             |
| `assets:working-directory`            | Fixes data-root, credentials, and templates-directory prerequisites                   |
| `assets:session-data`                 | Owns the session hierarchy and the `SessionData` marker Phase 4 reads                 |
| `assets:library-extension`            | Owns the `required_raw_assets` policy and the `SESSION_TYPES_USING_VR_TASK` claim     |
| `assets:project-hierarchy`            | Confirms the recording project exists                                                 |
| `video:camera-setup`                  | CTI and video runtime requirement deep-dives                                          |
| `communication:microcontroller-setup` | Microcontroller manifest and discovery deep-dives                                     |
| `/zaber-interface`                    | Owns per-device Zaber discovery and validation semantics                              |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Servers and resolution:
- [ ] sle mcp and slsa mcp connected
- [ ] Active system identified from the <system>_system_configuration.yaml filename, not the free-form name field,
      and resolved to its owning skill through /acquisition-system-setup's registry

Platform and storage:
- [ ] get_platform_environment_status_tool reported all required components healthy
- [ ] The active system's mount sweep reported the data root and every declared mount reachable

Hardware:
- [ ] Hardware sweep via /acquisition-system-setup confirmed the expected hardware
- [ ] Per-subsystem discovery and validation done for each subsystem the active system composes, skipped when none
- [ ] For a session type in SESSION_TYPES_USING_VR_TASK, check_unity_bridge_tool reported the shared Unity editor
      bridge reachable, skipped for every other session type
- [ ] For a session driving a system-specific instrument control interface, confirmed reachable through the active
      system's skill, skipped when none

Configuration:
- [ ] The active system's configuration validator passed
- [ ] Required raw assets for the session about to be recorded confirmed against SessionData.required_raw_assets
- [ ] Did NOT write any configuration from this skill, since verification is read-only
```
