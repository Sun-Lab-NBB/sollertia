---
name: subject-metadata
description: >-
  Reads the per-session SurgeryData snapshot via the sollertia-shared-assets MCP server. One
  monolithic file per session bundles subject, procedure, drugs, implants, and injections;
  callers project sections from the returned dict. Read-only at this layer. Use when looking
  up an animal's surgical history, implants, drugs, or injections for a session.
user-invocable: true
---

# Sollertia subject metadata

Reads the per-session `SurgeryData` snapshot via the `slsa mcp` MCP server. This skill is the
**exclusive** owner of `read_subject_surgery_tool` and `describe_surgery_schema_tool`.

Surgery metadata is treated as a **monolithic single-file record**. One `SurgeryData` YAML per
session carries every section (subject, procedure, drugs, implants, injections); there are no
per-section MCP tools — callers project the sections they need from the returned payload dict.

---

## Scope

**Covers:**
- Reading the per-session `SurgeryData` YAML via `read_subject_surgery_tool`
- Schema introspection via `describe_surgery_schema_tool`
- The relationship between the nested sections (`SubjectData`, `ProcedureData`, `DrugData`,
  `ImplantData`, `InjectionData`) and the containing `SurgeryData` record

**Does not cover:**
- Writing or modifying subject records — the source of truth is the upstream Google Sheets and
  reads are read-only at the sollertia-shared-assets MCP layer. To modify a record, edit the
  Google Sheet directly and let the next acquisition run capture the updated snapshot.
- Discovering subjects (see `/project-hierarchy` for `discover_subjects_tool`)
- Reading session-level data (see `/session-data`, `/session-descriptors`,
  `/session-hardware-state`, and the experiment plugin's `/session-snapshots`)

---

## SurgeryData model

A single `SurgeryData` YAML aggregates every surgery-time record for one animal. The library
defines five nested dataclasses that become sections of that one file — none of them is an
independent on-disk artifact:

| Section      | Nested dataclass  | Captures                                                                                        |
|--------------|-------------------|-------------------------------------------------------------------------------------------------|
| `subject`    | `SubjectData`     | id, ear-punch, sex, genotype, date of birth, pre-surgery weight, cage, housing location, status |
| `procedure`  | `ProcedureData`   | Surgery start/end timestamps, surgeon, protocol, surgery + post-op notes, quality               |
| `drugs`      | `DrugData`        | Peri-surgical drugs (LRS, ketoprofen, buprenorphine, dexamethasone), each with volume and code  |
| `implants`   | `ImplantData[]`   | Per-implant: name, target region, manufacturer code, AP/ML/DV stereotactic coordinates          |
| `injections` | `InjectionData[]` | Per-injection: name, target, volume (nL), manufacturer code, AP/ML/DV stereotactic coordinates  |

Subject metadata is **animal-scoped**, not session-scoped. A single animal accumulates records
across its lifetime within whichever project currently owns it (each animal belongs to exactly
one project at a time — see `/project-hierarchy` for the constraint). The authoritative source
is the upstream Google Sheets; the sollertia-shared-assets MCP layer **does not query Google
Sheets at runtime** and only ever reads on-disk per-session snapshots.

### On-disk location

The canonical — and only — location this library reads is the per-session snapshot at

```text
<session>/raw_data/surgery_metadata.yaml
```

(`RawDataFiles.SURGERY_METADATA`, exposed as `SessionData.surgery_metadata_path`). The
acquisition runtime writes this file into each session's `raw_data/` at session start, copying
the upstream Google Sheets record as it stood when the session began.
`discover_session_descriptors_tool` (owned by `/session-data`) classifies the file as kind
`surgery_metadata`.

Reads of this file are **per-session snapshots**: the value is the animal's surgery record at
the moment the session was acquired, not a live view of the Google Sheet. To see a newer
revision, acquire a new session for the same animal and read that session's snapshot.

---

## MCP tool surface

| Tool                            | Purpose                                                                       |
|---------------------------------|-------------------------------------------------------------------------------|
| `read_subject_surgery_tool`     | Loads the full `SurgeryData` payload for a session (exclusive to this skill)  |
| `describe_surgery_schema_tool`  | Returns the `SurgeryData` schema with nested section schemas (exclusive)      |

The discovery side of "which subjects exist" is owned by `/project-hierarchy` via
`discover_subjects_tool`; call it as a natural share when you need to enumerate animals before
reading their surgery snapshots.

`read_subject_surgery_tool` takes a `session_path` and reads
`<session>/raw_data/surgery_metadata.yaml`. Accepts either the session root or its `raw_data/`
subdirectory — the helper normalizes both forms.

Section projection is a **caller-side** operation. The returned `data` dict has the five
top-level keys `subject`, `procedure`, `drugs`, `implants`, `injections` — pick the one you
need. Examples:

```python
# After read_subject_surgery_tool(session_path="/data/MyProject/A01/2026-04-20-15-30-00-000000") succeeds:
response["data"]["subject"]      # SubjectData section (dict)
response["data"]["procedure"]    # ProcedureData section (dict)
response["data"]["drugs"]        # DrugData section (dict)
response["data"]["implants"]     # list[ImplantData dict]
response["data"]["injections"]   # list[InjectionData dict]
```

---

## Workflows

### Reading an animal's surgical history for a specific session

1. **Verify prerequisites:**
   - MCP server connected (else `/assets-mcp-environment-setup`).
   - The target session exists and `raw_data/surgery_metadata.yaml` is present inside it.
     Confirm via `discover_session_descriptors_tool` (owned by `/session-data`) if in doubt.
2. **Read the full SurgeryData record:**
   ```text
   read_subject_surgery_tool(session_path="<absolute path to session root>")
   ```
3. **Project the section(s) the user asked about** from `response["data"]`. For implants,
   read `response["data"]["implants"]`; for drugs, `response["data"]["drugs"]`; and so on.
4. **Report to the user.**

### Auditing all subjects across a project

1. **Hand off to `/project-hierarchy`** for
   `discover_subjects_tool(project="<project>", root_directory="<absolute>")` to enumerate
   animals.
2. **For each subject**, hand off to `/project-hierarchy` /`/session-discovery` to pick one of
   its sessions (typically the most recent), then call
   `read_subject_surgery_tool(session_path=<session path>)`.
3. **Project and aggregate** the sections you need across subjects, then report.

### Cross-referencing drug administration with session timing

1. **Pick a representative session** for the animal (usually the most recent one) via
   `/session-discovery`.
2. **Read the surgery snapshot:**
   ```text
   read_subject_surgery_tool(session_path="<absolute>")
   ```
   Extract `response["data"]["drugs"]`.
3. **Hand off to `/project-hierarchy`** for `discover_sessions_tool(animal_id="<id>", ...)` to
   enumerate the animal's sessions.
4. **Hand off to `/session-data`** to read individual session timestamps.
5. **Aggregate the cross-reference and report.**

---

## Modifying subject records

This skill does **not** support writing subject records. The sollertia-shared-assets MCP
server has no `write_subject_*_tool` because the source of truth is the upstream Google
Sheets.

If a record needs to be added or corrected:

1. Direct the user to the relevant upstream Google Sheet.
2. Edit the row in the sheet directly.
3. The correction will appear in the next session acquired for that animal (the acquisition
   runtime copies the Sheets record into each session's `surgery_metadata.yaml` at session
   start). Existing sessions keep the snapshot that was valid at their acquisition time.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] The target session's raw_data/surgery_metadata.yaml exists (confirmed via
      discover_session_descriptors_tool or by inspecting the session directory)
- [ ] read_subject_surgery_tool was called with a session_path (not with subject_id)
- [ ] Section extraction was done caller-side from response["data"]["<section>"] — did not
      look for a per-section MCP tool (none exist)
- [ ] discover_subjects_tool was called via /project-hierarchy when enumeration was needed
- [ ] Did not attempt to write any subject record from this skill (not supported)
```

---

## Related skills

| Skill                           | Relationship                                                                         |
|---------------------------------|--------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup` | Run first if the MCP server is not connected                                         |
| `/project-hierarchy`            | Owns `discover_subjects_tool` and the project tree walk                              |
| `/session-data`                 | Owns `discover_session_descriptors_tool` that classifies `surgery_metadata.yaml`     |
| `/session-descriptors`          | Sibling — descriptors capture per-session runtime state (separate from surgery data) |
