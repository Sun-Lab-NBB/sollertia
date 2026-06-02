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
  `/working-directory` and the active system's skill)
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
(`/experiment-mcp-environment-setup`, assets plugin `/assets-mcp-environment-setup`,
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
   `/working-directory`.
2. Determine the active system's **type** from the returned `file_path`: the configuration filename is
   `<system>_system_configuration.yaml` (for the `mesoscope` system, `mesoscope_system_configuration.yaml`),
   whose `<system>` segment is the `AcquisitionSystems` value, confirmable against
   `list_supported_acquisition_systems_tool()`. The configuration payload itself carries no
   acquisition-system field, so the filename is the authoritative type discriminator. Do NOT key off the
   configuration's `name` field: `name` is a free-form, human-readable label that may be customized — and
   even at its default value is not a reliable type discriminator.
3. Resolve that type to its owning skill via `/acquisition-system-setup`'s **Supported acquisition
   systems** registry (this skill does not duplicate that table). The skill name is not derivable from
   the type — `mesoscope` resolves to `/mesoscope-vr`, not `/mesoscope`. The phases below refer to the
   resolved skill as "the active system's skill."

### Phase 1: Platform configuration prerequisites

Take a single-call snapshot of every Sollertia platform configuration component owned by
`sollertia-shared-assets`:

```text
get_platform_environment_status_tool()
```

This read-only `slsa` tool reports the readiness of the working directory, data root, task-templates
directory, and Google credentials. Only the working directory is required for `slsa mcp` to function; the
others are optional and gate only the workflows that use them, so the tool's `overall_ok` reflects the
required components only. System-configuration validity is NOT reported here — that is verified in Phase 4.
If every required component reports healthy, advance to Phase 2. When a component reports unhealthy, hand
off to the owning assets-plugin skill to fix it — working directory / data root / credentials / templates
are all set by assets plugin `/working-directory`; this skill never writes configuration.

### Phase 2: Network storage mounts

```text
check_system_mounts_tool()
```

Validates every network storage location the active system configuration declares — the within-system
shares and each configured `filesystem.storage_directories` destination. A system may declare none and
run entirely on local storage, in which case this phase has nothing to verify. The set of locations is
read from the active configuration automatically; for the canonical list of a system's shares and
storage destinations, hand off to the active system's skill resolved in Phase 0 (for the `mesoscope`
system, `/mesoscope-vr`).

For a path that fails, drill in with `check_mount_accessibility_tool(path=...)`:

- `Exists: False` — the path does not exist; check `/etc/fstab` or systemd mount units.
- `Mount: False` — the path exists but is not a mount point (a local directory may be used instead of network storage).
- `Writable: False` — the path exists but the write test failed; check permissions or mount options
  (uid, gid, file_mode, dir_mode).

### Phase 3: Hardware connectivity

| Check                     | Tool                         | Server | Expected result                      |
|---------------------------|------------------------------|--------|--------------------------------------|
| Cameras detected          | `list_cameras`               | `axvs` | Expected camera indices              |
| Video runtime ready       | `check_runtime_requirements` | `axvs` | FFMPEG available, GPU detected       |
| CTI file (Harvesters)     | `get_cti_status`             | `axvs` | CTI configured (if using GenICam)    |
| Microcontrollers detected | `list_microcontrollers`      | `axci` | Expected microcontroller IDs / roles |
| MQTT broker reachable     | `check_mqtt_broker`          | `axci` | Connection successful                |

The tools above are the platform-universal discovery stack — the same for any Sollertia acquisition
system. What each should return (which camera indices, which microcontroller IDs and their roles) is
system-specific and is NOT enumerated here: hand off to `/acquisition-system-setup`, which owns the
concrete per-system hardware mapping and the full discovery and troubleshooting semantics, to learn and
confirm the expected values. This skill calls these tools read-only as a pre-flight sweep.

For every third-party-SDK subsystem the active system composes, additionally run that subsystem's
discovery tool and confirm the devices against the recorded configuration. The subsystem set, its tools,
and expected values are system-specific — hand off to the active system's skill. (For the `mesoscope`
system: Zaber motors via `get_zaber_devices_tool`; see `/zaber-interface` and `/acquisition-system-setup`.)

If the active system drives a Virtual Reality task through the Unity game engine (for the `mesoscope`
system, when running an `experiment` session), additionally confirm the Unity Editor MCP Bridge is
reachable with `check_unity_bridge_tool` (`sle`). The bridge starts automatically inside the Unity Editor,
so an unreachable bridge means the editor is not open — the runtime cannot open the scene or arm the VR
task without it. Hand off to `/vr-driver-interface` for the bridge contract.

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
`/project-hierarchy`; there is no project-listing tool on `sle mcp`.

---

## Quick health check

For a rapid pre-session check:

1. `get_platform_environment_status_tool()` — platform configuration healthy.
2. `check_system_mounts_tool()` — all storage accessible.
3. Hand off to `/acquisition-system-setup` — the hardware the active system declares (cameras,
   microcontrollers, any third-party-SDK subsystems such as Zaber motors, MQTT broker) is present.
4. `validate_system_configuration_tool()` — configuration valid.
5. For a VR/Unity experiment session, `check_unity_bridge_tool()` — the Unity Editor is open and its MCP
   bridge is reachable.

If all pass, the system is ready for acquisition.

---

## Troubleshooting guide

### Mount failures

| Symptom                | Likely cause         | Resolution                                                  |
|------------------------|----------------------|-------------------------------------------------------------|
| Path does not exist    | Mount not configured | Add an entry to `/etc/fstab` or create a systemd mount unit |
| Exists but not a mount | Local directory used | Check mount status: `mount \| grep <path>`                  |
| Not writable           | Permission issue     | Check mount options (uid, gid, file_mode, dir_mode)         |
| Stale mount            | Network disruption   | Remount: `sudo umount -l <path> && sudo mount <path>`       |

The remediation commands above are Linux examples; on Windows or macOS use the OS-appropriate mount tooling
(the symptom and likely cause are OS-independent).

### Hardware not detected / MQTT failures

Delegate to `/acquisition-system-setup`'s troubleshooting tables — it owns the discovery tooling and
the camera/microcontroller/Zaber/MQTT failure modes.

---

## Post-check actions

1. **All checks pass** — the system is ready for session execution.
2. **Configuration prerequisite unhealthy** — hand off to assets plugin `/working-directory`.
3. **Mount failures** — resolve OS-level mount issues before proceeding.
4. **Hardware missing** — hand off to `/acquisition-system-setup`.
5. **Configuration invalid** — hand off to the active acquisition system's skill (currently
   `/mesoscope-vr`, for the `mesoscope` system) to correct the system configuration.
6. **Unity bridge unreachable** (VR/Unity experiment sessions) — open the Unity project in the editor so
   its MCP bridge auto-starts (confirm with `sle get unity`); hand off to `/vr-driver-interface`.

---

## Related skills

| Skill                                          | Relationship                                                                                        |
|------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `/acquisition-system-setup`                    | Owns the full hardware-discovery sweep this skill hands off to                                      |
| `/mesoscope-vr`                                | Active acquisition system's skill (currently `mesoscope`); owns its config authoring and validation |
| `/vr-driver-interface`                         | Owns the Unity editor bridge check (`check_unity_bridge_tool`) for VR/Unity systems                 |
| `/experiment-mcp-environment-setup`            | Run first if the `sle mcp` server is not connected                                                  |
| `/pipeline`                                    | Phase 5 (pre-session health check) is owned by this skill                                           |
| assets plugin `/working-directory`             | Fixes data-root / credentials / templates prerequisites                                             |
| assets plugin `/project-hierarchy`             | Confirms the recording project exists                                                               |
| `ataraxis@video:camera-setup`                  | CTI / video runtime requirement deep-dives                                                          |
| `ataraxis@communication:microcontroller-setup` | Microcontroller manifest / discovery deep-dives                                                     |

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
- [ ] For a VR/Unity experiment session, check_unity_bridge_tool reported the editor bridge reachable
      (skip for non-Unity systems and non-experiment sessions)
- [ ] Did NOT write any configuration from this skill (read-only verification only)
```
