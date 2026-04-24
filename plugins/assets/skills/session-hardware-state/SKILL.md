---
name: session-hardware-state
description: >-
  Reads, writes, and validates per-session hardware-state YAMLs (currently
  MesoscopeHardwareState only) via the sollertia-shared-assets MCP server. Owns
  write_session_hardware_state_tool and describe_session_hardware_state_schema_tool. Use when
  inspecting the hardware configuration active at acquisition, repairing a corrupted snapshot,
  or amending hardware-state fields.
user-invocable: true
---

# Sollertia session hardware state

Reads, writes, and validates the per-session, **system-specific** hardware-state YAML file that
the acquisition runtime writes inside a Sollertia session's `raw_data/` directory. Uses the
`slsa mcp` MCP server. This skill is the **exclusive** owner of `write_session_hardware_state_tool`
and `describe_session_hardware_state_schema_tool` — no other skill in the marketplace may call
these.

Every acquisition system stores a per-session snapshot of its active hardware-module parameters
under the same canonical filename (`hardware_state.yaml` inside `raw_data/`). Each system
defines its own schema dataclass; today the only concrete subclass is **`MesoscopeHardwareState`**
(used by the Mesoscope-VR system), so the slsa MCP tools currently bind to that schema. Future
acquisition systems would add their own hardware-state classes following the same shape — the
skill's role would extend to those without changing its conceptual contract. The remainder of
this skill describes the pattern using the mesoscope schema as the worked example.

The file exists for **acquisition session types whose runtime exercises hardware modules**. On
Mesoscope-VR that means mesoscope-experiment, lick-training, and run-training sessions;
window-checking sessions never produce a `hardware_state.yaml` because their code path does not
exercise hardware modules, and `validate_session_tool` does not include this file in its
required-file inventory check. Reading the snapshot for such a session will fail with a
missing-file error — that is expected, not a corrupted session.

The hardware-state dataclass lives in `sollertia-shared-assets` (not the acquisition runtime)
because it is consumed by **both** the acquisition runtime and the downstream processing
pipeline. That is why the read/write/describe tools live on the slsa MCP server even though the
file itself is written by the acquisition runtime at session start.

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
  objective position snapshot (`mesoscope_positions.yaml`) — both are owned by the experiment
  plugin's `/session-snapshots`
- Reading the frozen experiment configuration captured at session start (see
  `/experiment-configuration` for `read_experiment_configuration_tool`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering sessions (see `/project-hierarchy` for `discover_sessions_tool`)
- Initial working directory setup (see `/working-directory`)

---

## What is a hardware-state snapshot

A hardware-state snapshot captures the per-rig hardware module parameters that were active when
the acquisition runtime started the session. The file is written **once at session start** by
the acquisition runtime and never modified afterward. Downstream processing pipelines read it to
translate raw acquired signals back into physical units and to know which modules were
exercised.

For the canonical field list and types call `describe_session_hardware_state_schema_tool` — do
not rely on handwritten field tables that may drift from the slsa source of truth. Every field
defaults to `None`, and a `None` value means **"the corresponding hardware module was not used
by the executed runtime"** — not "missing data." That semantic is the load-bearing convention
for downstream pipelines and is preserved when you write or amend a snapshot below.

### Per-session-type field population (Mesoscope-VR example)

Which fields end up populated vs. left `None` is **deterministic per session type**, not a
property of the rig the session ran on. The Mesoscope-VR runtime writes a different subset for
each session type — other acquisition systems will define their own per-session-type
populations against their own hardware-state schema:

| Session type           | Populated fields                                                                                                                                                             | Left as `None`                                                                                                               |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `mesoscope experiment` | All 11 fields. `recorded_mesoscope_ttl=True`. `delivered_gas_puffs` is computed from whether any `GasPuffTrial` exists in the experiment configuration's `trial_structures`. | None — every field is set.                                                                                                   |
| `lick training`        | `torque_per_adc_unit`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.                        | `cm_per_pulse`, `maximum_brake_strength`, `minimum_brake_strength`, `screens_initially_on`, `recorded_mesoscope_ttl`.        |
| `run training`         | `cm_per_pulse`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.                               | `maximum_brake_strength`, `minimum_brake_strength`, `torque_per_adc_unit`, `screens_initially_on`, `recorded_mesoscope_ttl`. |
| `window checking`      | (no file produced — see top of skill)                                                                                                                                        | n/a                                                                                                                          |

When repairing or amending a snapshot via `write_session_hardware_state_tool`, set fields that
the matching session type leaves as `None` to `None` in the payload. Setting them to a numeric
value would imply the hardware module was used when in fact it wasn't, and that misinforms the
processing pipeline's eligibility checks.

---

## File location

```text
<session>/raw_data/hardware_state.yaml
```

---

## MCP tool surface

| Tool                                          | Purpose                                                                                         |
|-----------------------------------------------|-------------------------------------------------------------------------------------------------|
| `read_session_hardware_state_tool`            | Reads a `hardware_state.yaml` file at an explicit path, parsing it with the class for the given `acquisition_system` |
| `write_session_hardware_state_tool`           | Writes (or repairs) `hardware_state.yaml` (exclusive). Defaults to `overwrite=True` — see below |
| `describe_session_hardware_state_schema_tool` | Returns the hardware-state schema for a given acquisition system (exclusive)                    |

Read, write, and describe tools all take an explicit `acquisition_system` — path resolution and
class selection are the caller's responsibility. The canonical on-disk path is always
`<session>/raw_data/hardware_state.yaml`; `acquisition_system` selects the parsing dataclass via
`HARDWARE_STATE_REGISTRY` (today only `mesoscope` is registered). To enumerate valid values call
`list_supported_acquisition_systems_tool`. To discover session roots, hand off to
`/project-hierarchy` for `discover_sessions_tool`.

`describe_session_hardware_state_schema_tool` returns `acquisition_system` (the validated enum
value) and `schema` (the dataclass field schema). `acquisition_system` defaults to `"mesoscope"`.

`read_session_hardware_state_tool` is documented here and other skills should hand off when they
need to read the snapshot. It is read-only and may be called as a natural share by any skill that
needs to inspect hardware state.

---

## Workflows

### Inspecting the hardware state for a session

1. **Verify prerequisites:** MCP server connected (else hand off to
   `/assets-mcp-environment-setup`).
2. **Locate the session.** Hand off to `/project-hierarchy` for `discover_sessions_tool` if needed,
   then construct the file path as `<session>/raw_data/hardware_state.yaml`.
3. **Read the snapshot:**
   ```text
   read_session_hardware_state_tool(
       file_path="<session>/raw_data/hardware_state.yaml",
       acquisition_system="<system>",
   )
   ```
4. **Interpret `None` fields as "module not used by this runtime"**, not as missing data.

### Repairing a corrupted hardware-state snapshot

1. **Verify prerequisites** (same as above).
2. **Read the existing snapshot** (if recoverable) with `read_session_hardware_state_tool`.
3. **Inspect the schema:**
   ```text
   describe_session_hardware_state_schema_tool(acquisition_system="<system>")
   ```
4. **Build the corrected payload** as a JSON-friendly dict matching the schema. Set fields whose
   modules were not active during the session to `None`.
5. **Confirm the planned write with the user.** This file is normally written only by the
   acquisition runtime at session start. `write_session_hardware_state_tool` defaults to
   `overwrite=True`, so it silently clobbers the existing snapshot with no backup. If the user
   wants the call to refuse-on-existing instead, pass `overwrite=False` explicitly in Step 6.
6. **Write the corrected snapshot:**
   ```text
   write_session_hardware_state_tool(
       file_path="<session>/raw_data/hardware_state.yaml",
       acquisition_system="<system>",
       hardware_state_payload={ ... },
       overwrite=True,   # default; set to False to refuse-on-existing
   )
   ```
   The kwargs are `file_path`, `acquisition_system`, `hardware_state_payload`, and the
   keyword-only `overwrite` (default `True`). The tool writes wherever `file_path` points;
   the canonical location is `<session>/raw_data/hardware_state.yaml`.
7. **Re-read to verify:**
   ```text
   read_session_hardware_state_tool(
       file_path="<session>/raw_data/hardware_state.yaml",
       acquisition_system="<system>",
   )
   ```

### Amending a hardware-state snapshot with reconciled data

Use the same workflow as repair, but mutate only the fields the user wants to amend (e.g., a
recalibrated `valve_scale_coefficient` value). Always confirm with the user before writing — every
write replaces the entire file.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] File path was constructed as <session>/raw_data/hardware_state.yaml
- [ ] Confirmed the session type is mesoscope-experiment, lick-training, or run-training
      (window-checking sessions have no hardware_state.yaml by design)
- [ ] describe_session_hardware_state_schema_tool was called before any write
- [ ] Fields the session type leaves as None per the population table were set to None in the payload
- [ ] User confirmed the planned write — including awareness that overwrite defaults to True
- [ ] Payload was passed as hardware_state_payload (the correct kwarg name)
- [ ] If refuse-on-existing semantics were required, overwrite=False was passed explicitly
- [ ] write_session_hardware_state_tool succeeded without schema errors
- [ ] read_session_hardware_state_tool returned the expected content after the write
- [ ] Did not touch SessionData, descriptors, Zaber positions, or mesoscope positions from this skill
```

---

## Related skills

| Skill                                     | Relationship                                                           |
|-------------------------------------------|------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`           | Run first if the MCP server is not connected                           |
| `/session-data`                           | Sibling — owns `SessionData` and the session anatomy                   |
| `/session-descriptors`                    | Sibling — owns the per-session-type descriptor read/write/schema       |
| `/experiment-configuration`               | Owns `read_experiment_configuration_tool` (reads both project source and frozen session snapshot) |
| `/project-hierarchy`                      | Provides `discover_sessions_tool` to locate sessions                   |
| experiment plugin `/session-snapshots`    | Sibling — owns the Zaber and mesoscope-objective position snapshots    |
