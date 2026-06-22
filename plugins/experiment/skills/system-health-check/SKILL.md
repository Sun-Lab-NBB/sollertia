---
name: system-health-check
description: >-
  Comprehensive pre-flight verification for acquisition systems. Orchestrates MCP tools and skill
  hand-offs to check platform configuration readiness, network storage mounts, hardware
  connectivity, and configuration validity. Use before running acquisition sessions or when
  troubleshooting system issues.
user-invocable: true
---

# System health check

Comprehensive pre-flight verification for acquisition systems. Orchestrates the `sle mcp` server,
the assets plugin's `slsa mcp` server, and hardware-discovery hand-offs to validate that a host is
ready to run a session. This is the lighter-weight pre-session sweep; full bringup discovery is owned
by `/acquisition-system-setup`.

---

## Scope

**Covers:**
- Pre-session readiness verification: active-system identification, platform-configuration prerequisites,
  network-storage mounts, hardware-connectivity sweep, and configuration validity
- Orchestrating `sle mcp` / `slsa mcp` read-only checks and handing off hardware discovery to the active
  system's skill

**Does not cover:**
- Full system bringup and hardware-discovery deep-dives (see `/acquisition-system-setup`)
- Writing or repairing configuration (read-only verification; fixes are owned by assets plugin
  `assets:working-directory` and the active system's skill)
- Per-system hardware mappings or expected device values (owned by `/acquisition-system-setup`)

---

## MCP server requirements

| Server                             | CLI command | Used for                                                                                |
|------------------------------------|-------------|-----------------------------------------------------------------------------------------|
| `sollertia-experiment`             | `sle mcp`   | Active-system identification, mount checks, system-configuration + subsystem validation |
| `sollertia-shared-assets`          | `slsa mcp`  | Platform configuration status snapshot, supported-system vocabulary                     |
| `ataraxis-video-system`            | `axvs mcp`  | Camera discovery and video requirements (via `/acquisition-system-setup`)               |
| `ataraxis-communication-interface` | `axci mcp`  | Microcontroller discovery and MQTT broker check (via `/acquisition-system-setup`)       |

If a required server is unavailable, hand off to the owning plugin's MCP environment setup skill
(`/experiment-mcp-environment-setup`, `assets:assets-mcp-environment-setup`,
`ataraxis@video:video-mcp-environment-setup`, `ataraxis@communication:communication-mcp-environment-setup`).

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

Every later phase hands off to "the active system's skill" for system-specific expectations, so resolve
that skill first — never assume a particular system.

```text
read_system_configuration_tool()          # sle  — loads the active configuration from the working directory
list_supported_acquisition_systems_tool() # slsa — the AcquisitionSystems vocabulary of supported types
```

1. Call `read_system_configuration_tool()`. If it returns no configuration, no system is set up on this
   host: stop the health check and hand off to `/acquisition-system-setup` (bringup) and assets plugin
   `assets:working-directory`.
2. Determine the active system's **type** from the returned `file_path`: the configuration filename is
   `<system>_system_configuration.yaml` (for the `mesoscope` system, `mesoscope_system_configuration.yaml`),
   whose `<system>` segment is the `AcquisitionSystems` value, confirmable against
   `list_supported_acquisition_systems_tool()`. The configuration payload itself carries no
   acquisition-system field, so the filename is the authoritative type discriminator. Do NOT key off the
   configuration's `name` field: `name` is a free-form, human-readable label that may be customized — and
   even at its default value is not a reliable type discriminator.
3. Resolve that type to its owning skill via `/acquisition-system-setup`'s **Supported acquisition
   systems** registry (this skill does not duplicate that table). The skill name is not derivable from
   the type — `mesoscope` resolves to `mesoscope:mesoscope-vr`, not `/mesoscope`. The phases below refer to the
   resolved skill as "the active system's skill."

### Phase 1: Platform configuration prerequisites

Take a single-call snapshot of every Sollertia platform configuration component owned by
`sollertia-shared-assets`:

```text
get_platform_environment_status_tool()
```

This read-only `slsa` tool reports the readiness of the working directory, data root, task-templates
directory, and one `<category>_credentials` component per supported credentials category (currently
`google_credentials`). Only the working directory is required for `slsa mcp` to function; the others are
optional and gate only the workflows that use them, so the tool's `overall_ok` reflects the required
components only. System-configuration validity is NOT reported here — that is verified in Phase 4.
If every required component reports healthy, advance to Phase 2. When a component reports unhealthy, hand
off to the owning assets-plugin skill to fix it — working directory / data root / credentials / templates
are all set by `assets:working-directory`; this skill never writes configuration.

### Phase 2: Network storage mounts

```text
check_system_mounts_tool()
```

Validates every network storage location the active system configuration declares — the within-system
shares and each configured `filesystem.storage_directories` destination. A system may declare none and
run entirely on local storage, in which case this phase has nothing to verify. The set of locations is
read from the active configuration automatically; for the canonical list of a system's shares and
storage destinations, hand off to the active system's skill resolved in Phase 0 (for the `mesoscope`
system, `mesoscope:mesoscope-vr`).

For a path that fails, drill in with `check_mount_accessibility_tool(path=...)`:

- `Exists: False` — the path does not exist; the OS-level mount is not configured. Configure it through the
  host OS's persistent-mount mechanism (see the Mount failures troubleshooting table).
- `Mount: False` — the path exists but is not a mount point (a local directory may be used instead of network storage).
- `Writable: False` — the path exists but the write test failed; check share permissions and the mount's
  ownership and permission options.

### Phase 3: Hardware connectivity

| Check                     | Tool                              | Server | Expected result                      |
|---------------------------|-----------------------------------|--------|--------------------------------------|
| Cameras detected          | `list_cameras_tool`               | `axvs` | Expected camera indices              |
| Video runtime ready       | `check_runtime_requirements_tool` | `axvs` | FFMPEG available, GPU detected       |
| CTI file (Harvesters)     | `get_cti_status_tool`             | `axvs` | CTI configured (if using GenICam)    |
| Microcontrollers detected | `list_microcontrollers_tool`      | `axci` | Expected microcontroller IDs / roles |
| MQTT broker reachable     | `check_mqtt_broker_tool`          | `axci` | Connection successful                |
| Zaber devices detected    | `get_zaber_devices_tool`          | `sle`  | Expected Zaber devices (if any)      |

The tools above are domain-general discovery utilities, system-agnostic across Sollertia acquisition systems
(cameras and microcontrollers come from the `axvs` / `axci` dependency servers; Zaber discovery is a general
`sle` tool). Which of these a given system composes — and the values each should return (camera indices,
microcontroller IDs and roles, Zaber devices) — is system-specific and is NOT enumerated here: hand off to
`/acquisition-system-setup` for the per-system hardware mapping and the discovery and troubleshooting
semantics, and to `/zaber-interface` for the per-device Zaber semantics. This skill calls these tools
read-only as a pre-flight sweep.

For a session that runs the corridor task, additionally confirm the shared Unity Editor MCP Bridge is
reachable with `check_unity_bridge_tool` (`sle`; CLI `sle get unity`). Unity is a shared asset with its own
driver, not specific to any one acquisition system. The bridge starts automatically inside the Unity Editor,
so an unreachable bridge means the editor is not open — the runtime cannot open the scene or arm the VR task.
Training and window-checking sessions run no task and skip this check. Hand off to `/vr-driver-interface` for
the bridge contract.

If the active system drives a system-specific instrument control interface beyond the domain-general stack and
the shared Unity bridge, additionally confirm that interface is reachable. Its check tool, expected state, and
remediation are owned by the active system's skill — do NOT assume a tool here; hand off. (For the `mesoscope`
system: the ScanImage control bridge that arms and commands the Mesoscope over MQTT; see `mesoscope:mesoscope-vr`.)

### Phase 4: Configuration validity

| Check                        | Tool                                 | Expected result                                |
|------------------------------|--------------------------------------|------------------------------------------------|
| System configuration valid   | `validate_system_configuration_tool` | Valid; mounts healthy                          |
| Camera GenICam configs match | `verify_camera_configuration_tool`   | Each declared camera matches its stored config |

`validate_system_configuration_tool` is the platform-universal check — it validates whatever system
configuration is active against the live mounts. Each third-party-SDK subsystem additionally exposes its
own validator, which is system-specific: run one per subsystem the active system composes, and hand off
to the active system's skill for its parameters and expected settings. (For the `mesoscope` system: the
Zaber motors' `validate_zaber_configuration_tool(port, device_index)`; see
`/zaber-interface`.) A system that composes no such subsystem has nothing further to validate here.

When the active system records per-camera GenICam configuration paths (standard for GenTL/GenICam
cameras), `verify_camera_configuration_tool` dumps each camera's live node configuration and diffs it
against the stored YAML, reporting per-camera `match` and `value_mismatches`. Cameras with no path
declared are skipped, and a system with no GenICam cameras has nothing to check here. On a mismatch, hand
off to `/acquisition-system-setup` to restore or re-baseline, and to `ataraxis@video:camera-setup` for the
GenICam dump/restore mechanics.

Project existence (for a session about to be recorded) is verified through the assets plugin
`assets:project-hierarchy`; there is no project-listing tool on `sle mcp`.

---

## Quick health check

For a rapid pre-session check:

1. `get_platform_environment_status_tool()` — platform configuration healthy.
2. `check_system_mounts_tool()` — all storage accessible.
3. Hand off to `/acquisition-system-setup` — the hardware the active system declares (cameras,
   microcontrollers, any third-party-SDK subsystems such as Zaber motors, MQTT broker) is present.
4. `validate_system_configuration_tool()` — configuration valid.
5. For a session that runs the corridor task, `check_unity_bridge_tool()` — the Unity Editor is open and its
   MCP bridge is reachable (training and window-checking sessions run no task and skip it).
6. For a session that drives a system-specific instrument control interface, confirm it through the active
   system's skill (for the `mesoscope` system, the ScanImage control bridge — see `mesoscope:mesoscope-vr`).

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

The symptoms and likely causes are OS-independent; the resolution mechanics are OS-specific. Resolve the
concrete commands for the host OS before suggesting fixes — for example, `/etc/fstab` entries or systemd
mount units and `mount`/`umount` on Linux, mapped network drives (`net use`) on Windows, and
`mount`/automount or Finder's "Connect to Server" on macOS.

### Hardware not detected / MQTT failures

Delegate to `/acquisition-system-setup`'s troubleshooting tables — it owns the discovery tooling and
the camera/microcontroller/Zaber/MQTT failure modes.

---

## Post-check actions

1. **All checks pass** — the system is ready for session execution.
2. **Configuration prerequisite unhealthy** — hand off to `assets:working-directory`.
3. **Mount failures** — resolve OS-level mount issues before proceeding.
4. **Hardware missing** — hand off to `/acquisition-system-setup`.
5. **Configuration invalid** — hand off to the active acquisition system's skill (currently
   `mesoscope:mesoscope-vr`, for the `mesoscope` system) to correct the system configuration.
6. **Unity bridge unreachable** (sessions that run the corridor task) — open the Unity project in the editor so
   its MCP bridge auto-starts (confirm with `sle get unity`); hand off to `/vr-driver-interface`.
7. **System-specific control interface unreachable** — hand off to the active system's skill to bring it up
   (for the `mesoscope` system, the ScanImage control bridge; see `mesoscope:mesoscope-vr`).

---

## Related skills

| Skill                                          | Relationship                                                                                     |
|------------------------------------------------|--------------------------------------------------------------------------------------------------|
| `/acquisition-system-setup`                    | Owns the full hardware-discovery sweep this skill hands off to                                   |
| `mesoscope:mesoscope-vr`                       | Active acquisition system's skill (`mesoscope`); owns config/validation and the ScanImage bridge |
| `/vr-driver-interface`                         | Owns the shared Unity editor bridge check (`check_unity_bridge_tool`) for corridor-task sessions |
| `/experiment-mcp-environment-setup`            | Run first if the `sle mcp` server is not connected                                               |
| `/pipeline`                                    | Phase 5 (pre-session health check) is owned by this skill                                        |
| `assets:working-directory`                     | Fixes data-root / credentials / templates prerequisites                                          |
| `assets:project-hierarchy`                     | Confirms the recording project exists                                                            |
| `ataraxis@video:camera-setup`                  | CTI / video runtime requirement deep-dives                                                       |
| `ataraxis@communication:microcontroller-setup` | Microcontroller manifest / discovery deep-dives                                                  |
| `/zaber-interface`                             | Owns per-device Zaber discovery / validation semantics for systems that compose Zaber motors     |

---

## Verification checklist

```text
- [ ] sle mcp and slsa mcp connected
- [ ] Active system identified via read_system_configuration_tool (by the configuration filename's
      AcquisitionSystems type, not the free-form name) and resolved to its owning skill
- [ ] get_platform_environment_status_tool reported all components healthy
- [ ] check_system_mounts_tool reported all mounts OK
- [ ] Hardware sweep via /acquisition-system-setup confirmed expected hardware
- [ ] validate_system_configuration_tool passed
- [ ] Per-subsystem discovery/validation done for each subsystem the active system composes (see that
      system's skill; for mesoscope, Zaber via /zaber-interface) — skip if it composes none
- [ ] For a session that runs the corridor task, check_unity_bridge_tool reported the shared Unity editor
      bridge reachable (training and window-checking sessions run no task and skip it)
- [ ] For a session that drives a system-specific instrument control interface, confirmed reachable via the
      active system's skill (for mesoscope, the ScanImage bridge via mesoscope:mesoscope-vr) — skip if none
- [ ] Did NOT write any configuration from this skill (read-only verification only)
```
