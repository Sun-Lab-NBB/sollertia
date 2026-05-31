---
name: subject-metadata
description: >-
  Reads and amends `surgery_metadata.yaml` (SurgeryData) files via the sollertia-shared-assets
  MCP server. One monolithic file bundles subject, procedure, drugs, implants, and injections;
  callers project sections from the returned dict. Tools are file-path based — the caller
  supplies the path. Use when looking up or fixing an animal's surgical history, implants,
  drugs, or injections.
user-invocable: false
---

# Sollertia subject metadata

Reads and amends `SurgeryData` YAML files via the `slsa mcp` MCP server. This skill is the
**exclusive** owner of `read_surgery_data_tool`, `write_surgery_data_tool`, and
`describe_surgery_data_schema_tool`.

Surgery metadata is treated as a **monolithic single-file record**. One `SurgeryData` YAML
carries every section (subject, procedure, drugs, implants, injections); there are no
per-section MCP tools — callers project the sections they need from the returned payload dict.
The tools operate on whatever absolute `file_path` the caller supplies; they do not care
whether that path points at a session snapshot, a dataset-level copy, or an ad-hoc location.

---

## Scope

**Covers:**
- Reading any `surgery_metadata.yaml` (full `SurgeryData` payload) via `read_surgery_data_tool`
- Amending any `surgery_metadata.yaml` in place via `write_surgery_data_tool` (validated
  full-record replacement; does not propagate to the upstream Google Sheet or to any other
  copy of the same file)
- Schema introspection via `describe_surgery_data_schema_tool`
- The relationship between the nested sections (`SubjectData`, `ProcedureData`, `DrugData`,
  `ImplantData`, `InjectionData`) and the containing `SurgeryData` record
- Guidance on where `surgery_metadata.yaml` is expected to live (session snapshot and dataset
  per-animal copy) and how callers resolve those paths

**Does not cover:**
- Durable corrections to an animal's surgical record. The authoritative source of truth is the
  upstream Google Sheet; `write_surgery_data_tool` only amends the one file at the path the
  caller supplies. For a correction that should apply to every future session, edit the
  Google Sheet directly and let the next acquisition run capture the updated snapshot.
- Propagating an amendment from one copy of the file to another. Session snapshots and the
  dataset-level copy are independent files on disk — writing to one does not update any other.
- Partial/per-section updates — the write tool validates and replaces the **full** `SurgeryData`
  payload. To change one field, read the current file, mutate the returned dict, then write
  it back whole.
- Resolving the canonical path for you. The caller (or a collaborating skill) supplies the
  absolute `file_path`; this skill only reads and writes.
- Discovering animals (see `/project-hierarchy` for `get_data_root_overview_tool`)
- Reading session-level data (see `/session-data`, `/session-descriptors`,
  `/session-hardware-state`, and the experiment plugin's `/session-snapshots`)
- Dataset assembly and dataset-level surgery path resolution (see the forging plugin's
  `/datasets` skill, which owns `DatasetData.surgery_paths`)

---

## SurgeryData model

### What's in the file

A `surgery_metadata.yaml` file is the complete record of a **surgical intervention performed
on a research animal prior to data acquisition** — what was done to the animal, by whom, when, 
with which drugs, and what was implanted or injected where. One file per animal-surgery, 
aggregating every surgery-time fact into five sections.

The library defines five nested dataclasses that become sections of that one file — none of
them is an independent on-disk artifact:

| Section      | Nested dataclass  | Captures                                                                                        |
|--------------|-------------------|-------------------------------------------------------------------------------------------------|
| `subject`    | `SubjectData`     | id, ear-punch, sex, genotype, date of birth, pre-surgery weight, cage, housing location, status |
| `procedure`  | `ProcedureData`   | Surgery start/end timestamps, surgeon, protocol, surgery + post-op notes, quality               |
| `drugs`      | `DrugData`        | Peri-surgical drugs (LRS, ketoprofen, buprenorphine, dexamethasone), each with volume and code  |
| `implants`   | `ImplantData[]`   | Per-implant: name, target region, manufacturer code, AP/ML/DV stereotactic coordinates          |
| `injections` | `InjectionData[]` | Per-injection: name, target, volume (nL), manufacturer code, AP/ML/DV stereotactic coordinates  |

`implants` and `injections` are **lists** (0..N entries each); a surgery may record none, one,
or many of either. The other three sections are singletons — exactly one `subject`, one
`procedure`, one `drugs` block per file.

### Units and encodings

The section table is a human summary; the on-disk fields carry specific units and encodings
that callers must respect when reading values or constructing a write payload.

- **Timestamps** (`date_of_birth_us`, `surgery_start_us`, `surgery_end_us`) — `int`,
  microseconds elapsed since the UTC epoch.
- **Weight** (`weight_g`) — `float`, grams (pre-surgery).
- **Drug volumes** (`lactated_ringers_solution_volume_ml`, `ketoprofen_volume_ml`,
  `buprenorphine_volume_ml`, `dexamethasone_volume_ml`) — `float`, millilitres.
- **Injection volume** (`injection_volume_nl`) — `float`, nanolitres (note the unit mismatch
  vs. drugs: drugs are mL, injections are nL).
- **Stereotactic coordinates** (`*_ap_coordinate_mm`, `*_ml_coordinate_mm`,
  `*_dv_coordinate_mm` on implants and injections) — `float`, millimetres relative to bregma.
- **Surgery quality** (`surgery_quality`) — `int`, range 0–3 inclusive:
  0 = unusable, 1 = usable but below publication threshold, 2 = publication-grade,
  3 = high-tier publication-grade. Defaults to 0.
- **Cage** (`cage`) — `int`, the cage number.
- **Identifiers and codes** (manufacturer codes, subject id (`int`), ear_punch, sex, genotype,
  status, location_housed, protocol, surgeon, surgery_notes, post_op_notes) — strings except
  `id` and `cage` which are integers.

For the full field list with types, call `describe_surgery_data_schema_tool`; the runtime schema is
always authoritative over this summary.

### Scope and provenance

Subject metadata is **animal-scoped** conceptually. A single animal accumulates records across
its lifetime within whichever project currently owns it (each animal belongs to exactly one
project at a time — see `/project-hierarchy`). The authoritative source is the upstream Google
Sheet; the sollertia-shared-assets MCP layer **does not query Google Sheets at runtime** and
only ever reads whatever YAML file the caller points at.

### Known file locations

`surgery_metadata.yaml` (`RawDataFiles.SURGERY_METADATA`) is written by different pipelines
into several canonical locations. All of them hold the same schema and are read/written by the
same two tools; this skill does not distinguish between them beyond helping the caller resolve
the right path.

| Location                                        | Populated by                                                        | Discovery path                                                                                                                          |
|-------------------------------------------------|---------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| `<session>/raw_data/surgery_metadata.yaml`      | Acquisition runtime at session start (snapshot of the Google Sheet) | `SessionData.raw_data.surgery_metadata_path`; `inspect_sessions_tool` lists it as the `surgery_metadata_path` entry in `raw_data_files` |
| `<dataset_root>/<animal>/surgery_metadata.yaml` | Forging pipeline (`shutil.copy2` from the animal's latest session)  | `DatasetData.surgery_paths` (owned by the forging plugin's `/datasets` skill)                                                           |

Other locations are possible — the tools take any absolute path — but these are the two
populated automatically. All copies are **snapshots** of the Google Sheet state at the moment
their pipeline ran; none is a live view.

Snapshots are not strictly immutable: `write_surgery_data_tool` can amend any one file in
place. That amendment is local to the one file whose path was passed. It does not flow back
to the Google Sheet, and it does not update any sibling copy (e.g., writing the session
snapshot does not update the dataset copy, and vice versa). For a correction that should
apply everywhere, edit the Google Sheet and let downstream pipelines re-capture it.

---

## MCP tool surface

| Tool                                | Purpose                                                                         |
|-------------------------------------|---------------------------------------------------------------------------------|
| `read_surgery_data_tool`            | Loads the full `SurgeryData` payload from a file path (exclusive to this skill) |
| `write_surgery_data_tool`           | Writes a validated full `SurgeryData` payload to a file path (exclusive)        |
| `describe_surgery_data_schema_tool` | Returns the `SurgeryData` schema with nested section schemas (exclusive)        |

Both `read_surgery_data_tool` and `write_surgery_data_tool` take an explicit `file_path`
— path resolution is the caller's responsibility. See **Known file locations** above for the
two canonical paths and which neighboring skill owns each.

Path-resolution hand-offs:
- Session snapshot → `/project-hierarchy` + `/session-discovery` for session roots;
  `/session-data` (`inspect_sessions_tool`) to confirm the file is present.
- Dataset per-animal copy → the forging plugin's `/datasets` skill, which owns
  `DatasetData.surgery_paths` and can hand back the per-animal path mapping.
- Enumerating animals → `/project-hierarchy` (`get_data_root_overview_tool`).

`write_surgery_data_tool` also accepts `surgery_payload: dict[str, Any]` (the full record —
all five sections) and a keyword-only `overwrite: bool = True`. The payload is validated against
`SurgeryData` before anything is written, so a bad payload fails without touching the file.
There is no partial-update tool; to change a single field, read the current file, mutate the
returned dict, and write it back whole.

Section projection is a **caller-side** operation. The returned `data` dict has the five
top-level keys `subject`, `procedure`, `drugs`, `implants`, `injections` — pick the one you
need. Examples:

```python
# After read_surgery_data_tool(file_path="<absolute path to surgery_metadata.yaml>")
# succeeds:
response["data"]["subject"]      # SubjectData section (dict)
response["data"]["procedure"]    # ProcedureData section (dict)
response["data"]["drugs"]        # DrugData section (dict)
response["data"]["implants"]     # list[ImplantData dict]
response["data"]["injections"]   # list[InjectionData dict]
```

---

## Workflows

All workflows reduce to: **resolve the path → read / write**. The resolution step changes
depending on where the file lives; the read/write step is the same everywhere.

### Reading a surgery record (generic)

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`); the
   target `surgery_metadata.yaml` exists at the path you are about to pass.
2. **Resolve the `file_path`** using the hand-off that matches the container:
   - **Session snapshot** → session root from `/project-hierarchy` or `/session-discovery`;
     optionally confirm the file is present via `inspect_sessions_tool`
     (`/session-data`). Path is `<session>/raw_data/surgery_metadata.yaml`.
   - **Dataset per-animal copy** → hand off to the forging plugin's `/datasets` skill for the
     mapping from animal id → path (exposed as `DatasetData.surgery_paths`). Path is
     `<dataset_root>/<animal>/surgery_metadata.yaml`.
   - **Ad-hoc location** → the user supplies the path directly.
3. **Read the full SurgeryData record:**
   ```text
   read_surgery_data_tool(file_path="<absolute path>")
   ```
4. **Project the section(s) the user asked about** from `response["data"]`. For implants,
   read `response["data"]["implants"]`; for drugs, `response["data"]["drugs"]`; and so on.
5. **Report to the user.** If the read came from a snapshot (session or dataset copy), remind
   the user the values reflect the Google Sheet state at the moment that copy was written.

### Amending a surgery record in place

Use this when a single copy has an error and a durable upstream correction is either not
warranted or not available yet. The amendment affects only the one file whose path is passed.

1. **Resolve the `file_path`** as in the read workflow.
2. **Read the current record** so you mutate a validated baseline rather than constructing one
   from scratch:
   ```text
   read_surgery_data_tool(file_path="<absolute path>")
   ```
3. **Mutate the returned `response["data"]` dict** in memory. Change only the fields that
   need correcting; keep every other section intact — the write tool validates and replaces
   the full record.
4. **Write the corrected payload back to the same path:**
   ```text
   write_surgery_data_tool(
       file_path="<absolute path>",
       surgery_payload=<mutated dict>,
   )
   ```
   The tool validates against `SurgeryData` before overwriting, so a malformed edit fails
   without damaging the file.
5. **Tell the user which copy was amended** and that the change does not propagate to any
   sibling copy (session ↔ dataset) or to the upstream Google Sheet. If other copies should
   match, amend each explicitly, and/or edit the Google Sheet for future captures.

### Auditing subjects across a project

1. **Hand off to `/project-hierarchy`** for
   `get_data_root_overview_tool(root_directory="<absolute>")` and pick the animals from
   `response["projects"][<project>].animals`.
2. **For each animal**, resolve a surgery file for it (typically its most recent session's
   snapshot via `/project-hierarchy` / `/session-discovery`, or a dataset copy via the
   forging plugin's `/datasets`).
3. **Read the record:** `read_surgery_data_tool(file_path=<resolved path>)`.
4. **Project and aggregate** the sections you need, then report.

### Cross-referencing drug administration with session timing

1. **Resolve one surgery file** for the animal (session snapshot via `/session-discovery`, or
   dataset copy via `/datasets`).
2. **Read it:** `read_surgery_data_tool(file_path="<absolute path>")`; extract
   `response["data"]["drugs"]`.
3. **Hand off to `/project-hierarchy`** for `get_data_root_overview_tool(root_directory="<absolute>")`
   and filter the flat `sessions` list by `animal == "<id>"` to enumerate the animal's sessions.
4. **Hand off to `/session-data`** to read individual session timestamps.
5. **Aggregate the cross-reference and report.**

---

## Amendment vs. upstream correction

The two write paths are not interchangeable — pick the one that matches the durability you
need.

| Scenario                                                                            | Use                                                                                                      |
|-------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| One session's snapshot has a data-entry error; re-acquiring the session is overkill | `write_surgery_data_tool` on the session file                                                            |
| A dataset's per-animal copy is wrong (e.g., it picked up a bad session snapshot)    | `write_surgery_data_tool` on the dataset file                                                            |
| The same field is wrong in both the session snapshot and the dataset copy           | `write_surgery_data_tool` against each file separately — there is no propagation                         |
| A field is wrong for the animal itself and should be right for every future capture | Edit the upstream Google Sheet; next session acquisition (and downstream dataset forge) captures the fix |
| Both a past file and future captures need fixing                                    | Do both — write tool for the existing file(s), sheet for future captures                                 |

The sollertia-shared-assets MCP layer does not query Google Sheets at runtime and will never
push an amendment back upstream; copies stay separate until the next capture.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] file_path was resolved via the right owner: /session-data or /session-discovery for
      session snapshots, forging plugin's /datasets for dataset per-animal copies, or the
      user directly for ad-hoc paths
- [ ] The target surgery_metadata.yaml exists at the resolved path (confirmed via
      inspect_sessions_tool for session snapshots, DatasetData.surgery_paths for
      dataset copies, or direct inspection otherwise)
- [ ] file_path passed to the tool is absolute
- [ ] Section extraction was done caller-side from response["data"]["<section>"] — did not
      look for a per-section MCP tool (none exist)
- [ ] Animal enumeration was done via /project-hierarchy's get_data_root_overview_tool when needed
- [ ] If writing: the full SurgeryData payload (all five sections) was supplied to
      write_surgery_data_tool — no partial-update path exists
- [ ] If writing: the user was told which single copy was amended and that the change does
      not propagate to sibling copies or to the upstream Google Sheet
```

---

## Related skills

| Skill                           | Relationship                                                                                           |
|---------------------------------|--------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup` | Run first if the MCP server is not connected                                                           |
| `/working-directory`            | Required prerequisite — bootstraps the working directory AND the Google credentials path read here     |
| `/project-hierarchy`            | Owns `get_data_root_overview_tool` and the project tree walk                                           |
| `/session-discovery`            | Resolves session roots for session-snapshot paths                                                      |
| `/session-data`                 | Owns `inspect_sessions_tool` that classifies `surgery_metadata.yaml` under a session                   |
| `/session-descriptors`          | Sibling — descriptors capture per-session runtime state (separate from surgery data)                   |
| forging plugin `/datasets`      | Resolves dataset per-animal `surgery_metadata.yaml` paths (`DatasetData.surgery_paths`)                |
