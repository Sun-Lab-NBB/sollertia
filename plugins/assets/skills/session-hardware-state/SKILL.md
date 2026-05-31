---
name: session-hardware-state
description: >-
  Reads, writes, and validates hardware-state YAMLs (currently MesoscopeHardwareState only)
  via the sollertia-shared-assets MCP server. Owns write_session_hardware_state_tool and
  describe_session_hardware_state_schema_tool. Tools are file-path based — the caller supplies
  the path. Use when inspecting the hardware configuration active at acquisition, repairing a
  corrupted snapshot, or amending hardware-state fields.
user-invocable: false
---

# Sollertia session hardware state

Reads, writes, and validates the per-acquisition, **system-specific** hardware-state YAML file
via the `slsa mcp` MCP server. This skill is the **exclusive** owner of
`write_session_hardware_state_tool` and `describe_session_hardware_state_schema_tool` — no
other skill in the marketplace may call these.

Hardware-state snapshots are treated as **standalone records** keyed on their schema (the
hardware-state dataclass that matches a given `AcquisitionSystems` value). The tools operate on
whatever absolute `file_path` the caller supplies; they do not care whether that path points at
a raw session snapshot or an ad-hoc location. The caller is responsible for resolving the path
and supplying `acquisition_system` so the right dataclass is used to parse or validate the file.

Every acquisition system stores its hardware-module parameters under the same canonical filename
(`hardware_state.yaml`). Each system defines its own schema dataclass; today the only concrete
subclass is **`MesoscopeHardwareState`** (used by the Mesoscope-VR system), so the slsa MCP
tools currently bind to that schema. Future acquisition systems would add their own
hardware-state classes following the same shape — the skill's contract would extend to those
without changing. The remainder of this skill describes the pattern using the mesoscope schema
as the worked example.

The hardware-state dataclass lives in `sollertia-shared-assets` (not the acquisition runtime)
because it is consumed by **both** the acquisition runtime and the downstream processing
pipeline. That is why the read/write/describe tools live on the slsa MCP server even though the
primary on-disk copy is written by the acquisition runtime at session start.

---

## Scope

**Covers:**
- Reading any `hardware_state.yaml` via `read_session_hardware_state_tool`
- Writing or repairing any `hardware_state.yaml` via `write_session_hardware_state_tool`
  (validated full-record replacement; does not propagate to any sibling copy)
- Schema introspection via `describe_session_hardware_state_schema_tool`
- Guidance on where `hardware_state.yaml` is expected to live and how callers resolve those
  paths

**Does not cover:**
- Partial/per-field updates — the write tool validates and replaces the **full** payload. To
  change one field, read the current file, mutate the returned dict, then write it back whole.
- Resolving the canonical path for you. The caller (or a collaborating skill) supplies the
  absolute `file_path`; this skill only reads and writes.
- Reading the `SessionData` marker (see `/session-data`)
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing the Zaber motor position snapshot (`zaber_positions.yaml`) or the
  mesoscope objective position snapshot (`mesoscope_positions.yaml`) — both are owned by the
  experiment plugin's `/session-snapshots`
- Reading the frozen experiment configuration captured at session start (see
  `/experiment-configuration` for `read_experiment_configuration_tool`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering sessions (see `/project-hierarchy` for `get_data_root_overview_tool`)
- Initial working directory setup (see `/working-directory`)

---

## What is a hardware-state snapshot

A hardware-state snapshot captures the per-rig hardware-module parameters that were active
when the acquisition runtime started the session. The file is written **once at session
start** by the acquisition runtime and, by convention, never modified afterward. Downstream
processing pipelines read it to translate raw acquired signals back into physical units and to
know which modules were exercised.

For the canonical field list and types call `describe_session_hardware_state_schema_tool` — do
not rely on handwritten field tables that may drift from the slsa source of truth. Every field
defaults to `None`, and a `None` value means **"the corresponding hardware module was not used
by the executed runtime"** — not "missing data." That semantic is the load-bearing convention
for downstream pipelines and is preserved when you write or amend a snapshot below.

### Session-type applicability (Mesoscope-VR example)

The file only exists for **acquisition session types whose runtime exercises hardware
modules**. On Mesoscope-VR that means mesoscope-experiment, lick-training, and run-training
sessions; window-checking sessions never produce a `hardware_state.yaml` because their code
path does not exercise hardware modules, and `inspect_sessions_tool` does not include this
file in the `required_assets` inventory check. Reading the snapshot for a window-checking session
will fail with a missing-file error — that is expected, not a corrupted session.

### Per-session-type field population (Mesoscope-VR example)

Which fields end up populated vs. left `None` is **deterministic per session type**, not a
property of the rig the session ran on. The Mesoscope-VR runtime writes a different subset for
each session type — other acquisition systems will define their own per-session-type
populations against their own hardware-state schema. The table below is a runtime-side
convention owned by `sollertia-experiment` (the acquisition runtime that writes the snapshot);
sollertia-shared-assets only defines the dataclass schema — see the experiment plugin for the
authoritative producer:

| Session type           | Populated fields                                                                                                                                                             | Left as `None`                                                                                                               |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `mesoscope experiment` | All 11 fields. `recorded_mesoscope_ttl=True`. `delivered_gas_puffs` is computed from whether any `GasPuffTrial` exists in the experiment configuration's `trial_structures`. | None — every field is set.                                                                                                   |
| `lick training`        | `torque_per_adc_unit`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.                        | `cm_per_pulse`, `maximum_brake_strength`, `minimum_brake_strength`, `screens_initially_on`, `recorded_mesoscope_ttl`.        |
| `run training`         | `cm_per_pulse`, `lick_threshold`, `valve_scale_coefficient`, `valve_nonlinearity_exponent`, `delivered_gas_puffs=False`, `system_state_codes`.                               | `maximum_brake_strength`, `minimum_brake_strength`, `torque_per_adc_unit`, `screens_initially_on`, `recorded_mesoscope_ttl`. |
| `window checking`      | (no file produced — see above)                                                                                                                                               | n/a                                                                                                                          |

When repairing or amending a snapshot via `write_session_hardware_state_tool`, set fields that
the matching session type leaves as `None` to `None` in the payload. Setting them to a numeric
value would imply the hardware module was used when in fact it wasn't, and that misinforms the
processing pipeline's eligibility checks.

### Known file locations

`hardware_state.yaml` (`RawDataFiles.HARDWARE_STATE`) is written by the acquisition runtime
into a single canonical location today. Additional locations may emerge as downstream
pipelines start copying the file alongside their outputs. All such copies hold the same schema
and are read/written by the same tools; this skill does not distinguish between them beyond
helping the caller resolve the right path.

| Location                                 | Populated by                                                                 | Discovery path                                                                                                                        |
|------------------------------------------|------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `<session>/raw_data/hardware_state.yaml` | Acquisition runtime at session start (primary and only automatic copy today) | Session root from `/session-discovery`; `inspect_sessions_tool` (`/session-data`) confirms presence in its `raw_data_files` inventory |

Other locations are possible — the tools take any absolute path. The snapshot is **frozen at
the moment the runtime wrote it**; downstream pipelines expect it immutable. Amendment via
`write_session_hardware_state_tool` is local to the one file whose path was passed and does
not flow back to any sibling copy that may exist.

---

## MCP tool surface

| Tool                                          | Purpose                                                                                                              |
|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| `read_session_hardware_state_tool`            | Loads a hardware-state file at an explicit `file_path`, parsing it with the class for the given `acquisition_system` |
| `write_session_hardware_state_tool`           | Writes a validated full hardware-state payload to a `file_path` (exclusive). Defaults to `overwrite=True`            |
| `describe_session_hardware_state_schema_tool` | Returns the hardware-state schema for a given acquisition system (exclusive). `acquisition_system` defaults to `"mesoscope"` |

All three tools take an explicit `acquisition_system` — class selection is the caller's
responsibility. `read_session_hardware_state_tool` and `write_session_hardware_state_tool`
additionally take `file_path`; path resolution is also the caller's responsibility. To
enumerate valid `acquisition_system` values call `list_supported_acquisition_systems_tool`;
today `HARDWARE_STATE_REGISTRY` only registers `mesoscope`.

Path-resolution hand-offs:
- Raw session snapshot → `/project-hierarchy` + `/session-discovery` for session roots;
  `/session-data` (`inspect_sessions_tool`) to confirm the file is present.
- Ad-hoc location → the user supplies the path directly.

If you don't know the acquisition system for a given file and it's a raw session snapshot,
hand off to `/session-data`. Call `inspect_sessions_tool` on the session root and read
`identity.acquisition_system` from the report, or read the marker directly with
`read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For ad-hoc paths,
the caller supplies `acquisition_system` directly.

`write_session_hardware_state_tool` also accepts `hardware_state_payload: dict[str, Any]` (the
full record) and a keyword-only `overwrite: bool = True`. The payload is validated against the
hardware-state dataclass before anything is written, so a bad payload fails without touching
the file. There is no partial-update tool.

`describe_session_hardware_state_schema_tool` returns `acquisition_system` (the validated enum
value) and `schema` (the dataclass field schema). `acquisition_system` defaults to `"mesoscope"`.

---

## Workflows

All workflows reduce to: **resolve the path → pick the acquisition system → read / write**.
The resolution step changes depending on where the file lives; the read/write step is the same
everywhere.

### Reading a hardware-state snapshot (generic)

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`); the
   target `hardware_state.yaml` exists at the path you are about to pass. Confirm the session
   type is one that produces the file (for Mesoscope-VR, anything other than window checking).
2. **Resolve the `file_path`** using the hand-off that matches the container:
   - **Raw session snapshot** → session root from `/project-hierarchy` or
     `/session-discovery`; optionally confirm the file is present via
     `inspect_sessions_tool` (`/session-data`). Path is
     `<session>/raw_data/hardware_state.yaml`.
   - **Ad-hoc location** → the user supplies the path directly.
3. **Determine `acquisition_system`.** For a raw session snapshot, hand off to `/session-data`
   — either call `inspect_sessions_tool(session_paths=["<session root>"])` and read
   `identity.acquisition_system` from the report, or read the marker directly with
   `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For ad-hoc paths,
   the caller supplies it directly.
4. **Read the snapshot:**
   ```text
   read_session_hardware_state_tool(
       file_path="<absolute path to hardware_state.yaml>",
       acquisition_system="<system>",
   )
   ```
5. **Interpret `None` fields as "module not used by this runtime"**, not as missing data.
6. **Report to the user.** Values reflect the rig state at session start; they are frozen and
   do not live-track the rig.

### Repairing or amending a hardware-state snapshot in place

Use this when a single copy has an error. The amendment affects only the one file whose path
is passed. Snapshots are conventionally immutable once written; confirm with the user before
every write.

1. **Resolve the `file_path` and `acquisition_system`** as in the read workflow.
2. **Read the current record** so you mutate a validated baseline rather than constructing one
   from scratch (skip only when the file is unreadable and must be reconstructed from scratch):
   ```text
   read_session_hardware_state_tool(file_path="<absolute path>", acquisition_system="<system>")
   ```
3. **Inspect the schema** if you need to add fields or double-check types:
   ```text
   describe_session_hardware_state_schema_tool(acquisition_system="<system>")
   ```
4. **Build or mutate the payload.** Set fields whose hardware modules were not active during
   the session to `None` per the population table; change only the fields that need
   correcting and keep every other field intact — the write tool validates and replaces the
   full record.
5. **Confirm the planned write with the user.** This file is normally written only by the
   acquisition runtime at session start. `write_session_hardware_state_tool` defaults to
   `overwrite=True`, so it silently clobbers the existing snapshot with no backup. If the
   user wants the call to refuse-on-existing instead, pass `overwrite=False` explicitly.
6. **Write the corrected payload back to the same path:**
   ```text
   write_session_hardware_state_tool(
       file_path="<absolute path>",
       acquisition_system="<system>",
       hardware_state_payload=<payload dict>,
       overwrite=True,  # default; set to False to refuse-on-existing
   )
   ```
   The tool validates against the hardware-state dataclass before overwriting, so a malformed
   edit fails without damaging the file.
7. **Re-read to verify:**
   ```text
   read_session_hardware_state_tool(file_path="<absolute path>", acquisition_system="<system>")
   ```
8. **Tell the user which copy was amended** and that the change does not propagate to any
   other copy that may exist.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] file_path was resolved via the right owner: /session-discovery or /session-data for raw
      session snapshots, or the user directly for ad-hoc paths
- [ ] acquisition_system was resolved from /session-data (raw snapshot) or supplied directly
      (ad-hoc), and passed to every tool call
- [ ] file_path passed to the tool is absolute
- [ ] Confirmed the session type produces a hardware_state.yaml (Mesoscope-VR window-checking
      sessions have no file by design)
- [ ] describe_session_hardware_state_schema_tool was called before any write when the payload
      structure was not already known
- [ ] Fields the session type leaves as None per the population table were set to None in the
      payload
- [ ] User confirmed the planned write — including awareness that overwrite defaults to True
- [ ] Payload was passed as hardware_state_payload (the correct kwarg name)
- [ ] If refuse-on-existing semantics were required, overwrite=False was passed explicitly
- [ ] If writing: the user was told which single copy was amended and that the change does not
      propagate to any sibling copy that may exist
- [ ] Did not touch SessionData, descriptors, Zaber positions, or mesoscope positions from
      this skill
```

---

## Related skills

| Skill                                  | Relationship                                                                                                                                                                    |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`        | Run first if the MCP server is not connected                                                                                                                                    |
| `/working-directory`                   | Required prerequisite — bootstraps the local working directory the agent uses to resolve project roots                                                                          |
| `/session-data`                        | Sibling — owns `SessionData` and the session anatomy; surfaces `acquisition_system`                                                                                             |
| `/session-descriptors`                 | Sibling — owns the per-session-type descriptor read/write/schema                                                                                                                |
| `/experiment-configuration`            | Owns `read_experiment_configuration_tool` (reads both project source and frozen session snapshot)                                                                               |
| `/project-hierarchy`                   | Provides `get_data_root_overview_tool` to locate sessions                                                                                                                       |
| experiment plugin `/session-snapshots` | Sibling — owns the Zaber and mesoscope-objective position snapshots                                                                                                             |
| `/library-extension`                   | Cross-cutting recipe to add a new `AcquisitionSystems` (or `SessionTypes`) member; lists the per-session-type field population table here that needs cloning for the new system |
