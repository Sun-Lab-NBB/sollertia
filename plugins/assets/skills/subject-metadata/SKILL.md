---
name: subject-metadata
description: >-
  Reads subject records (SubjectData, SurgeryData, ImplantData, InjectionData, DrugData) via the
  slsa MCP server. Subject metadata is sourced from Google Sheets and is read-only at the slsa
  layer. Use when looking up an animal's surgical history, implant details, drug administration, or
  injection records, or when building tooling that needs subject-level introspection.
user-invocable: true
---

# Sollertia subject metadata

Reads subject records that live in the project hierarchy and are sourced from Google Sheets via the
`slsa mcp` MCP server. This skill is the **exclusive** owner of the subject read tools and
`describe_surgery_schema_tool`.

---

## Scope

**Covers:**
- Reading core subject metadata (`SubjectData`)
- Reading surgical history (`SurgeryData`)
- Reading implant records (`ImplantData`)
- Reading injection records (`InjectionData`)
- Reading drug administration records (`DrugData`)
- Reading procedure records (`ProcedureData`)
- Schema introspection for surgery records

**Does not cover:**
- Writing or modifying subject records — these are sourced from Google Sheets and are read-only at
  the slsa MCP layer. To modify a subject record, edit the upstream Google Sheet directly.
- Discovering subjects (see `/project-hierarchy` for `discover_subjects_tool`)
- Reading session-level data (see `/session-data` for `SessionData`, `/session-descriptors` for
  per-session descriptors, `/session-snapshots` for runtime snapshots)

---

## Subject metadata model

Subject metadata is **animal-scoped**, not session-scoped. A single animal accumulates records
across its lifetime within whichever project currently owns it (each animal belongs to exactly
one project at a time — see `/project-hierarchy` for the constraint). When an animal is
migrated between projects, its metadata moves with it.

| Record type  | Dataclass       | Captures                                                                                        |
|--------------|-----------------|-------------------------------------------------------------------------------------------------|
| Core subject | `SubjectData`   | id, ear-punch, sex, genotype, date of birth, pre-surgery weight, cage, housing location, status |
| Surgery      | `SurgeryData`   | Aggregate surgery record: subject + procedure + drugs + implants + injections sections          |
| Implant      | `ImplantData`   | Per-implant: name, target region, manufacturer code, AP/ML/DV stereotactic coordinates          |
| Injection    | `InjectionData` | Per-injection: name, target, volume (nL), manufacturer code, AP/ML/DV stereotactic coordinates  |
| Drug         | `DrugData`      | Peri-surgical drugs (LRS, ketoprofen, buprenorphine, dexamethasone), each with volume and code  |
| Procedure    | `ProcedureData` | Surgery metadata: start/end timestamps, surgeon, protocol, surgery + post-op notes, quality     |

The authoritative source for these records is the upstream Google Sheets, but the slsa MCP
layer **does not query Google Sheets at runtime**. It reads the cached `SurgeryData` YAML
files written to disk by the acquisition runtime's preprocessing step. To modify a record,
edit the row in the source Google Sheet and let the next preprocessing run refresh the
cached YAML.

---

## MCP tool surface

| Tool                              | Returns                                                       |
|-----------------------------------|---------------------------------------------------------------|
| `read_subject_tool`               | Core subject metadata for a given subject ID                  |
| `read_subject_surgery_tool`       | All surgery records for the subject                           |
| `read_subject_implants_tool`      | All implant records for the subject                           |
| `read_subject_injections_tool`    | All injection records for the subject                         |
| `read_subject_drugs_tool`         | All drug administration records for the subject               |
| `read_subject_procedure_tool`     | Non-surgical procedure records for the subject                |
| `describe_surgery_schema_tool`    | Returns the schema for `SurgeryData`                          |

The discovery side of "which subjects exist" is owned by `/project-hierarchy` via
`discover_subjects_tool`. Call it as a natural share when you need to enumerate before reading.

---

## Workflows

### Looking up an animal's surgical history before a session

1. **Verify prerequisites:**
   - MCP server connected (else `/assets-mcp-environment-setup`).
   - Cached `SurgeryData` YAMLs exist on disk for the target subject under either
     `<root>/<project>/<subject>/`, `<root>/<project>/surgery_data/`, `<root>/surgery_data/`,
     or the slsa working directory's `surgery_data/`. The slsa subject reads do not call
     Google Sheets — they read from these on-disk caches, which are populated by the
     acquisition runtime's preprocessing step.
2. **Read the subject record:**
   ```text
   read_subject_tool(subject_id="<id>")
   ```
3. **Read the surgical history:**
   ```text
   read_subject_surgery_tool(subject_id="<id>")
   ```
4. **Read implants if relevant:**
   ```text
   read_subject_implants_tool(subject_id="<id>")
   ```
5. **Report to the user.**

### Auditing all subjects across a project

1. **Hand off to `/project-hierarchy`** for `discover_subjects_tool(project="<project>")` to enumerate.
2. **Loop:** For each subject ID, call the read tools in this skill.
3. **Aggregate and report.**

### Cross-referencing drug administration with session timing

1. **Read the drug records:**
   ```text
   read_subject_drugs_tool(subject_id="<id>")
   ```
2. **Hand off to `/project-hierarchy`** for `discover_sessions_tool(animal_id="<id>")` to enumerate
   sessions for the same animal.
3. **Hand off to `/session-data`** to read individual session timestamps.
4. **Aggregate the cross-reference and report.**

---

## Modifying subject records

This skill does **not** support writing subject records. The slsa MCP server has no
`write_subject_*_tool` because the source of truth is the upstream Google Sheets.

If a record needs to be added or corrected:

1. Direct the user to the relevant upstream Google Sheet.
2. Edit the row in the sheet directly.
3. Re-run preprocessing for a session belonging to that animal so the cached `SurgeryData`
   YAML is refreshed (the acquisition runtime's preprocessing step is what writes the cache).
4. Re-call `read_subject_*_tool` to confirm the change is visible to slsa.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] Cached SurgeryData YAML exists for the target subject under one of the resolver's
      candidate paths (project subtree, project surgery_data/, root surgery_data/, or slsa
      working directory's surgery_data/)
- [ ] discover_subjects_tool was called via /project-hierarchy when enumeration was needed
- [ ] Did not attempt to write any subject record from this skill (not supported)
```

---

## Related skills

| Skill                           | Relationship                                                      |
|---------------------------------|-------------------------------------------------------------------|
| `/assets-mcp-environment-setup` | Run first if the MCP server is not connected                      |
| `/project-hierarchy`            | Owns `discover_subjects_tool` and the project tree walk           |
| `/session-data`                 | Sibling — owns session-scoped data, this skill owns animal-scoped |
| `/session-descriptors`          | Sibling — descriptors capture per-session animal state            |
