---
name: session-hardware-state
description: >-
  Reads, writes, and validates hardware-state YAMLs via the sollertia-shared-assets MCP server, parsing each file at a
  caller-supplied path with the dataclass that HARDWARE_STATE_REGISTRY maps to the session's AcquisitionSystems value.
  Owns write_session_hardware_state_tool and describe_session_hardware_state_schema_tool. Use when inspecting the
  hardware configuration active at acquisition, repairing a corrupted snapshot, or amending hardware-state fields.
user-invocable: false
---

# Sollertia session hardware state

Reads, writes, and validates the per-acquisition, **system-specific** hardware-state YAML file via the `slsa mcp` MCP
server. This skill is the **exclusive** owner of `write_session_hardware_state_tool` and
`describe_session_hardware_state_schema_tool`, and no other skill in the marketplace may call these.

Hardware-state snapshots are treated as **standalone records** keyed on their schema, meaning the hardware-state
dataclass that matches a given `AcquisitionSystems` value. The tools operate on whatever absolute `file_path` the caller
supplies, and they do not care whether that path points at a raw session snapshot or an ad-hoc location. The caller is
responsible for resolving the path and supplying `acquisition_system` so the right dataclass is used to parse or
validate the file.

Every acquisition system stores its hardware-module parameters under the same canonical filename
(`hardware_state.yaml`). Each system defines its own schema dataclass, registered in `HARDWARE_STATE_REGISTRY` against
its `AcquisitionSystems` value. The slsa MCP tools dispatch on the `acquisition_system` the caller supplies to select
the matching dataclass. Future acquisition systems add their own hardware-state classes following the same shape, and
the skill's contract extends to those without changing. This skill describes the generic pattern only. For
Mesoscope-VR's concrete hardware-state schema, covering field names, types, defaults, and per-session-type
applicability, see `mesoscope:mesoscope-vr-session-schema`.

The hardware-state dataclass lives in `sollertia-shared-assets` rather than in the acquisition runtime because it is
consumed by **both** the acquisition runtime and the downstream processing pipeline. That is why the read, write, and
describe tools live on the slsa MCP server even though the primary on-disk copy is written by the acquisition runtime at
session start.

---

## Scope

**Covers:**
- Reading any `hardware_state.yaml` via `read_session_hardware_state_tool`
- Writing or repairing any `hardware_state.yaml` via `write_session_hardware_state_tool` (validated full-record
  replacement that does not propagate to any sibling copy)
- Schema introspection via `describe_session_hardware_state_schema_tool`
- Guidance on where `hardware_state.yaml` is expected to live and how callers resolve those paths

**Does not cover:**
- Partial or per-field updates. The write tool validates and replaces the **full** payload. To change one field, read
  the current file, mutate the returned dict, then write it back whole.
- Resolving the canonical path for you. The caller, or a collaborating skill, supplies the absolute `file_path`, and
  this skill only reads and writes.
- Reading the `SessionData` marker (see `/session-data`)
- Reading or writing per-session descriptors (see `/session-descriptors`)
- Reading or writing the Zaber motor position snapshot (`zaber_positions.yaml`) or the mesoscope objective position
  snapshot (`mesoscope_positions.yaml`), both owned by `mesoscope:mesoscope-vr-snapshots`
- Reading the frozen experiment configuration captured at session start (see `/experiment-configuration` for
  `read_experiment_configuration_tool`)
- Reading read assets (see `/data-assets`)
- Discovering sessions (see `/project-hierarchy` for `get_data_root_overview_tool`)
- Initial working directory setup (see `/working-directory`)

---

## What is a hardware-state snapshot

A hardware-state snapshot captures the per-rig hardware-module parameters that were active when the acquisition runtime
started the session. The file is written **once at session start** by the acquisition runtime and, by convention, never
modified afterward. Downstream processing pipelines read it to translate raw acquired signals back into physical units
and to know which modules were exercised.

For the canonical field list and types call `describe_session_hardware_state_schema_tool` with the `acquisition_system`
that owns the file, and read the returned payload instead of a handwritten field table that may drift from the slsa
source of truth. The shape of that payload is documented in the `## Response contract` section of
`/assets-mcp-environment-setup`.

The library enforces only the flat filename `hardware_state.yaml` and the `HARDWARE_STATE_REGISTRY` dispatch. No
import-time check inspects hardware-state fields. Each system's hardware-state class defines its own field set and its
own null semantics, so you MUST take both from `describe_session_hardware_state_schema_tool` and the owning system's
schema skill before interpreting or writing any value. Which fields a given session type populates, and whether a
session type produces a `hardware_state.yaml` at all, are system-specific for the same reason. Route Mesoscope-VR's rule
to `mesoscope:mesoscope-vr-session-schema`.

### Known file locations

`hardware_state.yaml` (`RawDataFiles.HARDWARE_STATE`) is written by the acquisition runtime into a single canonical
location today, `<session>/raw_data/hardware_state.yaml`. Additional locations may emerge as downstream pipelines start
copying the file alongside their outputs. All such copies hold the same schema and are read and written by the same
tools, and this skill does not distinguish between them beyond helping the caller resolve the right path.

Resolve a raw session snapshot from the session inventory rather than by hand. Call `inspect_sessions_tool`
(`/session-data`) on the session root, take the `raw_data_files` entry whose `field` is `hardware_state_path`, and read
that entry's `path` and `exists` directly instead of concatenating the session root with a filename.
`<session>/raw_data/hardware_state.yaml` is the fallback shape when no inventory is available, and the session roots
themselves come from `/session-discovery` or `/project-hierarchy`.

Other locations are possible, because the tools take any absolute path. The snapshot is **frozen at the moment the
runtime wrote it**, and downstream pipelines expect it immutable. Amendment via `write_session_hardware_state_tool` is
local to the one file whose path was passed and does not flow back to any sibling copy that may exist.

---

## MCP tool surface

| Tool                                          | Purpose                                                                                                              |
|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| `read_session_hardware_state_tool`            | Loads a hardware-state file at an explicit `file_path`, parsing it with the class for the given `acquisition_system` |
| `write_session_hardware_state_tool`           | Writes a validated full hardware-state payload to a `file_path` (exclusive). Defaults to `overwrite=True`            |
| `describe_session_hardware_state_schema_tool` | Returns the hardware-state schema for a given acquisition system (exclusive). `acquisition_system` is required       |

All three tools take an explicit `acquisition_system` and dispatch on it through `HARDWARE_STATE_REGISTRY` to select the
matching dataclass, so class selection is the caller's responsibility. `read_session_hardware_state_tool` and
`write_session_hardware_state_tool` additionally take `file_path`, and path resolution is also the caller's
responsibility.

`acquisition_system` is the `AcquisitionSystems` string value and never the enum member name, and the two differ, as in
`MESOSCOPE_VR = "mesoscope"`. `list_supported_acquisition_systems_tool` returns `{value, name}` pairs of which only
`value` is accepted, so passing `"MESOSCOPE_VR"` returns:

```text
Unable to resolve the hardware-state class. The acquisition_system 'MESOSCOPE_VR' is not a member of
AcquisitionSystems. Valid values: mesoscope.
```

To enumerate the valid values, hand off to `/experiment-configuration`, which owns
`list_supported_acquisition_systems_tool`. That tool enumerates the `AcquisitionSystems` members rather than the
registry, and the import-time coverage check guarantees that every member carries a `HARDWARE_STATE_REGISTRY` entry, so
the two lists match today. The resolver still carries a separate "No class is registered for" branch for the case where
they stop matching.

Path-resolution hand-offs:
- Raw session snapshot: `/project-hierarchy` plus `/session-discovery` for session roots, then `/session-data`
  (`inspect_sessions_tool`) for the `raw_data_files` entry.
- Ad-hoc location: the user supplies the path directly.

If you do not know the acquisition system for a given file and it is a raw session snapshot, hand off to
`/session-data`. Call `inspect_sessions_tool` on the session root and read `identity.acquisition_system` from the
report, or read the marker directly with `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For
ad-hoc paths, the caller supplies `acquisition_system` directly.

`write_session_hardware_state_tool` also accepts `hardware_state_payload: dict[str, Any]` (the full record) and a
keyword-only `overwrite: bool = True`. What a write tool actually validates is documented in the `## Response contract`
section of `/assets-mcp-environment-setup`, and hardware state is the sharpest case in the plugin.
`MesoscopeHardwareState` declares no `__post_init__` and gives all 11 of its fields a `None` default, so a misspelled or
omitted field name is silently dropped and persisted as `None`. Under that schema's own semantics a `None` reads
downstream as "this hardware module was not used", which means a single typo writes a false claim onto a frozen
acquisition record and nothing rejects it. You MUST re-read the file after every write and diff the returned `data`
against the payload you intended before reporting success. There is no partial-update tool.

On success, `read_session_hardware_state_tool` and `write_session_hardware_state_tool` both return the same four payload
keys: `data` (the serialized record), `file_path`, `acquisition_system` (the echoed input), and `hardware_state_class`
(the resolved dataclass name, useful for confirming the dispatch picked the class you expected).
`describe_session_hardware_state_schema_tool` returns `acquisition_system` (the echoed input) and `schema` (the
dataclass field schema), and `acquisition_system` is required with no default.

### Failure modes

The envelope these tools return is documented in the `## Response contract` section of `/assets-mcp-environment-setup`.
Four error messages are specific to this surface:

- **`acquisition_system` is not an `AcquisitionSystems` member.** The message quoted above. Recover by taking a `value`
  from `list_supported_acquisition_systems_tool` rather than guessing the spelling.
- **`acquisition_system` is a valid member with no registered class.** `Unable to resolve the hardware-state class. No
  class is registered for '<value>'. Registered systems: <list>.` The import-time coverage check makes this unreachable
  in a correctly imported library, so route it to `/library-extension`.
- **The file does not exist.** `Unable to read <Class> from <path>: the file does not exist.` This is how you learn a
  session carries no snapshot.
- **The write was refused.** `Unable to write <Class> to <path>: a file already exists at this path. Pass overwrite=True
  to replace it.` Only reachable when the caller passed `overwrite=False`, because the default is `True`.

---

## Workflows

All workflows reduce to: **resolve the path, pick the acquisition system, then read or write**. The resolution step
changes depending on where the file lives, and the read and write step is the same everywhere.

### Reading a hardware-state snapshot (generic)

1. **Verify the MCP server is connected**, else `/assets-mcp-environment-setup`. Do not pre-check that the file exists.
   Some session types exercise no hardware modules and produce no snapshot, and the read tool answers that case
   authoritatively with the file-does-not-exist error listed under Failure modes above.
2. **Resolve the `file_path`** using the hand-off that matches the container:
   - **Raw session snapshot**: session root from `/project-hierarchy` or `/session-discovery`, then the `raw_data_files`
     entry whose `field` is `hardware_state_path` from `inspect_sessions_tool` (`/session-data`).
   - **Ad-hoc location**: the user supplies the path directly.
3. **Determine `acquisition_system`.** For a raw session snapshot, hand off to `/session-data`. Either call
   `inspect_sessions_tool(session_paths=["<session root>"])` and read `identity.acquisition_system` from the report, or
   read the marker directly with `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For ad-hoc
   paths, the caller supplies it directly.
4. **Read the snapshot:**
   ```text
   read_session_hardware_state_tool(
       file_path="<absolute path to hardware_state.yaml>",
       acquisition_system="<system>",
   )
   ```
5. **Read the record from `response["data"]`** and interpret its null fields against the owning system's null semantics
   rather than assuming they mean missing data.
6. **Report to the user.** Values reflect the rig state at session start. They are frozen and do not live-track the rig.

### Repairing or amending a hardware-state snapshot in place

Use this when a single copy has an error. The amendment affects only the one file whose path is passed. Snapshots are
conventionally immutable once written, so confirm with the user before every write.

1. **Resolve the `file_path` and `acquisition_system`** as in the read workflow.
2. **Read the current record** so you mutate a validated baseline rather than constructing one from scratch. Skip this
   step only when the file is unreadable and must be reconstructed:
   ```text
   read_session_hardware_state_tool(file_path="<absolute path>", acquisition_system="<system>")
   ```
3. **Inspect the schema** if you need to add fields or double-check types:
   ```text
   describe_session_hardware_state_schema_tool(acquisition_system="<system>")
   ```
4. **Mutate `response["data"]` from step 2 into the new payload.** Change only the fields that need correcting and carry
   every other field through unchanged. The write tool replaces the full record, so a field you drop is written back at
   its dataclass default. The population rules and the null semantics are system-specific.
5. **Confirm the planned write with the user.** This file is normally written only by the acquisition runtime at session
   start. `write_session_hardware_state_tool` defaults to `overwrite=True`, so it silently clobbers the existing
   snapshot with no backup. If the user wants the call to refuse-on-existing instead, pass `overwrite=False` explicitly.
   Confirm that the parent directory of `file_path` already exists as well, because the tool creates any missing parent
   directories and a mistyped path therefore produces a stray file instead of an error.
6. **Write the corrected payload back to the same path:**
   ```text
   write_session_hardware_state_tool(
       file_path="<absolute path>",
       acquisition_system="<system>",
       hardware_state_payload=<payload dict>,
       overwrite=True,  # the default, set to False to refuse-on-existing
   )
   ```
   The hardware-state class defines no `__post_init__`, so this write is a shape check only and a dropped or misspelled
   key persists as `None` instead of failing.
7. **Re-read and diff. This step is mandatory, not a formality:**
   ```text
   read_session_hardware_state_tool(file_path="<absolute path>", acquisition_system="<system>")
   ```
   Compare the returned `data` field by field against the payload you intended. A key the write dropped surfaces here as
   `None`, and this diff is the only thing that catches it. Report success only after the diff is clean.
8. **Tell the user which copy was amended** and that the change does not propagate to any other copy that may exist.

---

## Related skills

| Skill                                   | Relationship                                                                                                                                          |
|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`         | Run first if the MCP server is not connected. Owns the plugin-wide response contract                                                                  |
| `/working-directory`                    | Required prerequisite that bootstraps the local working directory used to resolve project roots                                                       |
| `/session-discovery`                    | Resolves raw session roots                                                                                                                            |
| `/session-data`                         | Sibling that owns `SessionData` and the session anatomy, and surfaces `acquisition_system`                                                            |
| `/session-descriptors`                  | Sibling that owns the per-session-type descriptor read, write, and schema tools                                                                       |
| `/data-assets`                          | Sibling that owns read assets such as surgery records                                                                                                 |
| `/experiment-configuration`             | Owns `read_experiment_configuration_tool` and `list_supported_acquisition_systems_tool`                                                               |
| `/project-hierarchy`                    | Provides `get_data_root_overview_tool` to locate sessions                                                                                             |
| `mesoscope:mesoscope-vr-session-schema` | Owns Mesoscope-VR's concrete hardware-state field schema, its null semantics, and its per-session-type population                                     |
| `mesoscope:mesoscope-vr-snapshots`      | Sibling that owns the Zaber and mesoscope-objective position snapshots                                                                                |
| `/library-extension`                    | Adds a new `AcquisitionSystems` member and its `HARDWARE_STATE_REGISTRY` dataclass. A new `SessionTypes` member touches `DESCRIPTOR_REGISTRY` instead |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] file_path was resolved via the right owner: /session-discovery or /session-data for raw
      session snapshots, or the user directly for ad-hoc paths
- [ ] file_path passed to the tool is absolute
- [ ] acquisition_system was resolved from /session-data (raw snapshot) or supplied directly
      (ad-hoc), and passed to every tool call
- [ ] acquisition_system was passed as the AcquisitionSystems value, such as "mesoscope", and
      not as the enum member name
- [ ] describe_session_hardware_state_schema_tool was called before any write when the payload
      structure was not already known
- [ ] The payload carried every field of the record rather than only the fields being changed
- [ ] User confirmed the planned write, including awareness that overwrite defaults to True
- [ ] The parent directory of file_path already existed, since the write tool creates missing
      parents instead of failing on a mistyped path
- [ ] Payload was passed as hardware_state_payload (the correct kwarg name)
- [ ] If refuse-on-existing semantics were required, overwrite=False was passed explicitly
- [ ] After writing: the file was re-read and the returned data was diffed field by field
      against the intended payload before success was reported
- [ ] If writing: the user was told which single copy was amended and that the change does not
      propagate to any sibling copy that may exist
- [ ] Did not touch SessionData, descriptors, Zaber positions, or mesoscope positions from
      this skill
```
