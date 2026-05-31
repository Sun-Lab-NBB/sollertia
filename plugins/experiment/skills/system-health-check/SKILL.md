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

## MCP server requirements

| Server                  | CLI command | Used for                                                       |
|-------------------------|-------------|----------------------------------------------------------------|
| `sollertia-experiment`  | `sle mcp`   | Mount checks, system-configuration validation, Zaber validation |
| `sollertia-shared-assets` | `slsa mcp` | Platform configuration status snapshot                         |
| ataraxis-video-system   | `axvs mcp`  | Camera discovery and video requirements (via `/acquisition-system-setup`) |
| ataraxis-comm-interface | `axci mcp`  | Microcontroller discovery and MQTT broker check (via `/acquisition-system-setup`) |

If a required server is unavailable, hand off to the owning plugin's MCP environment setup skill
(`/experiment-mcp-environment-setup`, assets plugin `/assets-mcp-environment-setup`,
`ataraxis@video:video-mcp-environment-setup`, `ataraxis@communication:communication-mcp-environment-setup`).

---

## Complete verification workflow

```text
System health check progress:
- [ ] Phase 1: Platform configuration prerequisites verified
- [ ] Phase 2: Network storage mounts accessible
- [ ] Phase 3: Hardware connectivity confirmed
- [ ] Phase 4: Configuration validity confirmed
```

### Phase 1: Platform configuration prerequisites

Take a single-call snapshot of every Sollertia platform configuration component owned by
`sollertia-shared-assets`:

```text
get_platform_environment_status_tool()
```

This read-only `slsa` tool reports data-root readiness, Google credentials, task-templates
directory, and system-configuration presence. If every component reports healthy, advance to
Phase 2. When a component reports unhealthy, hand off to the owning assets-plugin skill to fix it —
data root / credentials / templates are set by assets plugin `/working-directory`; this skill never
writes configuration.

### Phase 2: Network storage mounts

```text
check_system_mounts_tool()
```

Validates `mesoscope_directory` and every configured `filesystem.storage_directories` destination.
For a path that fails, drill in with `check_mount_accessibility_tool(path=...)`:

- `Exists: No` — the path does not exist; check `/etc/fstab` or systemd mount units.
- `Mount: No` — the path exists but is not a mount point (a local directory may be used instead of network storage).
- `Writable: No` — the path exists but the write test failed; check permissions or mount options
  (uid, gid, file_mode, dir_mode).

### Phase 3: Hardware connectivity

| Check                     | Tool                          | Server | Expected result                       |
|---------------------------|-------------------------------|--------|---------------------------------------|
| Cameras detected          | `list_cameras`                | axvs   | Expected camera indices               |
| Video runtime ready       | `check_runtime_requirements`  | axvs   | FFMPEG available, GPU detected        |
| CTI file (Harvesters)     | `get_cti_status`              | axvs   | CTI configured (if using GenICam)     |
| Microcontrollers detected | `list_microcontrollers`       | axci   | ACTOR/SENSOR/ENCODER ports            |
| MQTT broker reachable     | `check_mqtt_broker`           | axci   | Connection successful                 |
| Zaber motors detected     | `get_zaber_devices_tool`      | sle    | All motor groups and axes             |

These read-only discovery tools are owned by `/acquisition-system-setup` (cameras, microcontrollers,
MQTT — via the ataraxis video and communication servers) and `/zaber-interface` (Zaber motors). Hand
off there for full discovery semantics and hardware troubleshooting; this skill calls them read-only
as a pre-flight sweep.

### Phase 4: Configuration validity

| Check                      | Tool                                             | Expected result                       |
|----------------------------|--------------------------------------------------|---------------------------------------|
| System configuration valid | `validate_system_configuration_tool`             | Valid; mounts healthy                 |
| Zaber configuration valid  | `validate_zaber_configuration_tool(port, device_index, expected_settings)` | VALID for each motor |

Project existence (for a session about to be recorded) is verified through the assets plugin
`/project-hierarchy`; there is no project-listing tool on `sle mcp`.

---

## Quick health check

For a rapid pre-session check:

1. `get_platform_environment_status_tool()` — platform configuration healthy.
2. `check_system_mounts_tool()` — all storage accessible.
3. Hand off to `/acquisition-system-setup` — cameras, microcontrollers, Zaber motors, and MQTT broker present.
4. `validate_system_configuration_tool()` — configuration valid.

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
5. **Configuration invalid** — hand off to `/mesoscope-vr` to correct the system configuration.

---

## Related skills

| Skill                                       | Relationship                                                       |
|---------------------------------------------|--------------------------------------------------------------------|
| `/acquisition-system-setup`                 | Owns the full hardware-discovery sweep this skill hands off to     |
| `/mesoscope-vr`                             | Owns system-configuration authoring and `validate_system_configuration_tool` |
| `/experiment-mcp-environment-setup`         | Run first if the `sle mcp` server is not connected                 |
| `/pipeline`                                 | Phase 5 (pre-session health check) is owned by this skill          |
| assets plugin `/working-directory`          | Fixes data-root / credentials / templates prerequisites            |
| assets plugin `/project-hierarchy`          | Confirms the recording project exists                              |
| `ataraxis@video:camera-setup`               | CTI / video runtime requirement deep-dives                         |
| `ataraxis@communication:microcontroller-setup` | Microcontroller manifest / discovery deep-dives                |

---

## Verification checklist

```text
- [ ] sle mcp and slsa mcp connected
- [ ] get_platform_environment_status_tool reported all components healthy
- [ ] check_system_mounts_tool reported all mounts OK
- [ ] Hardware sweep via /acquisition-system-setup confirmed expected hardware
- [ ] validate_system_configuration_tool passed
- [ ] validate_zaber_configuration_tool reported VALID for each motor
- [ ] Did NOT write any configuration from this skill (read-only verification only)
```
