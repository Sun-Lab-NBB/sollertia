---
name: subject-metadata
description: >-
  Reads subject records (SubjectData, SurgeryData, ImplantData, InjectionData, DrugData) via the
  sl-configure MCP server. Subject metadata is sourced from Google Sheets and is read-only at the slsa
  layer. Use when looking up an animal's surgical history, implant details, drug administration, or
  injection records, or when building tooling that needs subject-level introspection.
user-invocable: true
---

# Sollertia subject metadata

Reads subject records that live in the project hierarchy and are sourced from Google Sheets via the
`sl-configure mcp` MCP server. This skill is the **exclusive** owner of the subject read tools and
`describe_surgery_schema_tool`.

---

## Scope

**Covers:**
- Reading core subject metadata (`SubjectData`)
- Reading surgical history (`SurgeryData`)
- Reading implant records (`ImplantData`)
- Reading injection records (`InjectionData`)
- Reading drug administration records (`DrugData`)
- Schema introspection for surgery records

**Does not cover:**
- Writing or modifying subject records — these are sourced from Google Sheets and are read-only at
  the slsa MCP layer. To modify a subject record, edit the upstream Google Sheet directly.
- Discovering subjects (see `/project-hierarchy` for `discover_subjects_tool`)
- Reading session-level data (see `/session-data` for `SessionData`, `/session-descriptors` for
  per-session descriptors, `/session-snapshots` for runtime snapshots)
- Initial working directory setup (see `/working-directory`)

---

## Subject metadata model

Subject metadata is **animal-scoped**, not session-scoped. A single animal accumulates records across
its lifetime and across multiple projects:

| Record type  | Dataclass       | Captures                                                        |
|--------------|-----------------|-----------------------------------------------------------------|
| Core subject | `SubjectData`   | Subject ID, sex, date of birth, genotype, current weight        |
| Surgery      | `SurgeryData`   | Surgical procedures (craniotomy, headbar, window implant, etc.) |
| Implant      | `ImplantData`   | Implanted hardware (electrodes, optical fibers, headbars)       |
| Injection    | `InjectionData` | Viral / tracer / drug injections                                |
| Drug         | `DrugData`      | Drug administration records (water restriction, antibiotics)    |

The records live in the upstream Google Sheets (configured via `/working-directory`'s
`set_google_credentials_tool`) and are projected through the slsa MCP server as read-only views.

---

## MCP tool surface

| Tool                              | Returns                                                       |
|-----------------------------------|---------------------------------------------------------------|
| `read_subject_tool`               | Core subject metadata for a given subject ID                  |
| `read_subject_surgery_tool`       | All surgery records for the subject                           |
| `read_subject_implants_tool`      | All implant records for the subject                           |
| `read_subject_injections_tool`    | All injection records for the subject                         |
| `read_subject_drugs_tool`         | All drug administration records for the subject               |
| `describe_surgery_schema_tool`    | Returns the schema for `SurgeryData`                          |

The discovery side of "which subjects exist" is owned by `/project-hierarchy` via
`discover_subjects_tool`. Call it as a natural share when you need to enumerate before reading.

---

## Workflows

### Looking up an animal's surgical history before a session

1. **Verify prerequisites:**
   - MCP server connected (else `/mcp-environment-setup`).
   - Working directory and Google credentials set (else `/working-directory`).
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
2. **Hand off to `/project-hierarchy`** for `discover_sessions_tool(animal="<id>")` to enumerate
   sessions for the same animal.
3. **Hand off to `/session-data`** to read individual session timestamps.
4. **Aggregate the cross-reference and report.**

---

## Modifying subject records

This skill does **not** support writing subject records. The slsa MCP server has no
`write_subject_*_tool` because the source of truth is the upstream Google Sheets.

If a record needs to be added or corrected:

1. Direct the user to the relevant Google Sheet (configured via `/working-directory`).
2. Edit the row in the sheet directly.
3. Re-call `read_subject_*_tool` to confirm the change is visible to slsa.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] Google credentials are set (else read tools fail with "credentials not set")
- [ ] sollertia-shared-assets MCP server is connected
- [ ] discover_subjects_tool was called via /project-hierarchy when enumeration was needed
- [ ] Did not attempt to write any subject record from this skill (not supported)
```

---

## Related skills

| Skill                       | Relationship                                                       |
|-----------------------------|--------------------------------------------------------------------|
| `/working-directory`        | Required prerequisite — owns Google credentials path               |
| `/mcp-environment-setup`    | Run first if the MCP server is not connected                       |
| `/project-hierarchy`        | Owns `discover_subjects_tool` and the project tree walk            |
| `/session-data`             | Sibling — owns session-scoped data, this skill owns animal-scoped  |
| `/session-descriptors`      | Sibling — descriptors capture per-session animal state             |
