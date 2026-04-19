---
name: session-hardware-state
description: >-
  Reads, writes, and validates the per-session MesoscopeHardwareState YAML file via the slsa MCP
  server. Owns write_session_hardware_state_tool and describe_session_hardware_state_schema_tool.
  Use when inspecting the hardware configuration that was active when a session was acquired,
  repairing a corrupted hardware-state snapshot, or amending hardware-state fields after a manual
  reconciliation. The Zaber and mesoscope-objective position snapshots are owned by the experiment
  plugin's /session-snapshots skill.
user-invocable: true
---

# Sollertia session hardware state

Reads, writes, and validates the per-session `MesoscopeHardwareState` YAML file (`hardware_state.yaml`)
that lives inside every Sollertia session's `raw_data/` directory. Uses the `slsa mcp` MCP server.
This skill is the **exclusive** owner of `write_session_hardware_state_tool` and
`describe_session_hardware_state_schema_tool` — no other skill in the marketplace may call these.

`MesoscopeHardwareState` lives in `sollertia-shared-assets` (not the acquisition runtime) because it
is consumed by both the runtime (`sl-experiment`) and the processing pipeline (`sollertia-forgery`).
That is why the read/write/describe tools live on the slsa MCP server even though the file is
written by `sl-run` at session start.

---

## Scope

**Covers:**
- Reading `hardware_state.yaml` for a session (`read_session_hardware_state_tool`)
- Writing or repairing `hardware_state.yaml` (`write_session_hardware_state_tool`)
- Schema introspection for `MesoscopeHardwareState`
  (`describe_session_hardware_state_schema_tool`)

**Does not cover:**
- Reading the `SessionData` marker (see `/session-data`)
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing the Zaber motor position snapshot (`zaber_positions.yaml`) or the mesoscope
  objective position snapshot (`mesoscope_positions.yaml`) — both live on the `sl-get` MCP server
  and are owned by the experiment plugin's `/session-snapshots`
- Reading the frozen experiment configuration captured at session start (see
  `/experiment-configuration` for `read_session_experiment_configuration_tool`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering sessions (see `/project-hierarchy` for `discover_sessions_tool`)
- Initial working directory setup (see `/working-directory`)

---

## What is `MesoscopeHardwareState`

`MesoscopeHardwareState` is the snapshot of every per-rig hardware module parameter that was active
when `sl-run` started the session. The file is written **once at session start** by the acquisition
runtime and never modified afterward. The processing pipeline reads it to translate raw acquired
signals back into physical units and to know which modules were used.

| Field                           | Captures                                                                |
|---------------------------------|-------------------------------------------------------------------------|
| `cm_per_pulse`                  | Encoder pulses → centimeters conversion                                 |
| `maximum_brake_strength`        | Brake torque (N·cm) when fully engaged                                  |
| `minimum_brake_strength`        | Brake torque (N·cm) when fully disengaged                               |
| `lick_threshold`                | 12-bit ADC threshold for lick detection                                 |
| `valve_scale_coefficient`       | Power-law scale for water-valve open-time → dispensed volume            |
| `valve_nonlinearity_exponent`   | Power-law exponent for the same relationship                            |
| `torque_per_adc_unit`           | Torque sensor 12-bit ADC → N·cm conversion                              |
| `screens_initially_on`          | Initial state of the VR screens at session start                        |
| `recorded_mesoscope_ttl`        | Whether the session recorded mesoscope brain-activity data              |
| `delivered_gas_puffs`           | Whether the session delivered any gas puffs to the animal               |
| `system_state_codes`            | Mapping of integer system-state codes to human-readable state names     |

Every field defaults to `None`. A `None` value means "the corresponding hardware module was not used
by the executed runtime" — not "missing data."

---

## File location

```text
<session>/raw_data/hardware_state.yaml
```

The filename is `hardware_state.yaml` (no `mesoscope_` prefix). The file lives **inside `raw_data/`**,
not at the session root.

---

## MCP tool surface

| Tool                                            | Purpose                                                               |
|-------------------------------------------------|-----------------------------------------------------------------------|
| `read_session_hardware_state_tool`              | Reads `hardware_state.yaml` for a session                             |
| `write_session_hardware_state_tool`             | Writes (or repairs) `hardware_state.yaml` (exclusive to this skill)   |
| `describe_session_hardware_state_schema_tool`   | Returns the `MesoscopeHardwareState` schema (exclusive to this skill) |

`read_session_hardware_state_tool` is documented here and other skills should hand off when they
need to read the snapshot. It is read-only and may be called as a natural share by any skill that
needs to inspect hardware state.

---

## Workflows

### Inspecting the hardware state for a session

1. **Verify prerequisites:**
   - MCP server connected (else hand off to `/assets-mcp-environment-setup`).
   - Working directory set (else hand off to `/working-directory`).
2. **Locate the session.** Hand off to `/project-hierarchy` for `discover_sessions_tool` if needed.
3. **Read the snapshot:**
   ```text
   read_session_hardware_state_tool(session_path="<absolute>")
   ```
4. **Interpret `None` fields as "module not used by this runtime"**, not as missing data.

### Repairing a corrupted hardware-state snapshot

1. **Verify prerequisites** (same as above).
2. **Read the existing snapshot** (if recoverable) with `read_session_hardware_state_tool`.
3. **Inspect the schema:**
   ```text
   describe_session_hardware_state_schema_tool()
   ```
4. **Build the corrected payload** as a JSON-friendly dict matching the schema. Set fields whose
   modules were not active during the session to `None`.
5. **Confirm the planned write with the user.** This file is normally written only by `sl-run` at
   session start; manual edits are irreversible without a backup.
6. **Write the corrected snapshot:**
   ```text
   write_session_hardware_state_tool(
       session_path="<absolute>",
       hardware_state_payload={ ... },
   )
   ```
   The kwarg is `hardware_state_payload`. The destination file is always
   `<session>/raw_data/hardware_state.yaml` (not configurable).
7. **Re-read to verify:**
   ```text
   read_session_hardware_state_tool(session_path="<absolute>")
   ```

### Amending a hardware-state snapshot with reconciled data

Use the same workflow as repair, but mutate only the fields the user wants to amend (e.g., a
recalibrated `valve_scale_coefficient` value). Always confirm with the user before writing — every
write replaces the entire file.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] describe_session_hardware_state_schema_tool was called before any write
- [ ] User confirmed the planned write before it was executed
- [ ] Payload was passed as hardware_state_payload (the correct kwarg name)
- [ ] write_session_hardware_state_tool succeeded without schema errors
- [ ] read_session_hardware_state_tool returned the expected content after the write
- [ ] Did not touch SessionData, descriptors, Zaber positions, or mesoscope positions from this skill
```

---

## Related skills

| Skill                                     | Relationship                                                            |
|-------------------------------------------|-------------------------------------------------------------------------|
| `/working-directory`                      | Required prerequisite — must be run first                               |
| `/assets-mcp-environment-setup`           | Run first if the MCP server is not connected                            |
| `/session-data`                           | Sibling — owns `SessionData` and the session anatomy                    |
| `/session-descriptors`                    | Sibling — owns the per-session-type descriptor read/write/schema        |
| `/experiment-configuration`               | Owns `read_session_experiment_configuration_tool` (frozen experiment)   |
| `/project-hierarchy`                      | Provides `discover_sessions_tool` to locate sessions                    |
| experiment plugin `/session-snapshots`    | Sibling — owns the Zaber and mesoscope-objective position snapshots     |
