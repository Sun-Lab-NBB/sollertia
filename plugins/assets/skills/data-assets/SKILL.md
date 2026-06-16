---
name: data-assets
description: >-
  Reads, writes, and describes on-disk read-asset dataclasses via the sollertia-shared-assets MCP
  server's generic data-asset tools, dispatched by a `data_asset` identifier resolved from the
  platform's read-asset registry. Surgery data (`surgery_data` → `SurgeryData`) is the current
  worked example. Use when looking up or amending an animal's surgical history, or any other
  registered read asset.
user-invocable: false
---

# Sollertia data assets

Reads, writes, and describes the platform's **read assets** via the `slsa mcp` MCP server. A read
asset is metadata the platform reads from an external source (e.g., a Google Sheet) and caches on
disk as a typed dataclass; the canonical set is the `READ_ASSET_REGISTRY` in `sollertia-shared-assets`
(see assets `/library-extension`). This skill is the **exclusive** owner of `read_data_asset_tool`,
`write_data_asset_tool`, and `describe_data_asset_schema_tool`.

Every tool is keyed by a **`data_asset`** identifier (a `ReadAssets` value such as `"surgery_data"`)
that selects the on-disk dataclass. The interface is generic — one tool set serves every registered
read asset — so new assets become available here automatically once registered, with no new tools.
**Surgery data is the current worked example** and the only registered read asset today; the rest of
this skill uses it to illustrate the generic flow.

---

## Scope

**Covers:**
- Reading any read-asset YAML (full payload) via `read_data_asset_tool(file_path, data_asset)`
- Amending any read-asset YAML in place via `write_data_asset_tool(file_path, data_asset, payload)`
  (validated full-record replacement; does not propagate to the upstream source or to any other copy)
- Schema introspection via `describe_data_asset_schema_tool(data_asset)`
- Enumerating the registered read assets via `list_supported_data_assets_tool`
- Resolving the `data_asset` identifier (prefer automatic resolution; see below)
- The surgery-data worked example: its `SurgeryData` model, provenance, and file locations

**Does not cover:**
- Adding a **new** read asset (the dataclass + `ReadAssets` member + registry entry) — owned by
  assets `/library-extension` ("Adding a new read asset")
- Durable corrections to an asset's upstream source. For read assets captured from a Google Sheet,
  the authoritative source is the sheet; the MCP layer **does not query it at runtime** and writes
  here amend only the one on-disk file the caller points at. Edit the upstream source for a fix that
  should apply to every future capture.
- Propagating an amendment from one copy of the file to another — copies are independent on disk
- Partial/per-section updates — the write tool validates and replaces the **full** payload. To change
  one field, read the current file, mutate the returned dict, then write it back whole.
- Resolving the canonical `file_path` for you. The caller (or a collaborating skill) supplies the
  absolute path; this skill only reads and writes.
- Discovering animals (see `/project-hierarchy`)
- Reading session-level data (see `/session-data`, `/session-descriptors`, `/session-hardware-state`)

---

## MCP tool surface

| Tool                              | Purpose                                                                                             |
|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| `read_data_asset_tool`            | Loads a read-asset YAML, parsing it with the dataclass for the given `data_asset` (exclusive)       |
| `write_data_asset_tool`           | Writes a validated full payload to a path, validated against the `data_asset` dataclass (exclusive) |
| `describe_data_asset_schema_tool` | Returns the schema (nested dataclasses recursed inline) for the given `data_asset` (exclusive)      |
| `list_supported_data_assets_tool` | Enumerates the registered read assets — `value`, `name`, and `data_asset_class` for each            |

Signatures: `read_data_asset_tool(file_path, data_asset)`,
`write_data_asset_tool(file_path, data_asset, data_asset_payload, *, overwrite=True)`,
`describe_data_asset_schema_tool(data_asset)`. The `file_path` is always the caller's responsibility;
the `data_asset` selects the schema. The write payload is validated against the resolved dataclass
before anything is written, so a bad payload fails without touching the file. There is no
partial-update tool — to change a single field, read the current file, mutate the returned dict, and
write it back whole.

---

## Resolving the `data_asset`

`data_asset` is **required** — there is no default. Resolve it **automatically** wherever possible and
only fall back to asking the user:

1. **Infer from context first.** A `surgery_metadata.yaml` file, or a request about an animal's
   surgical history / implants / drugs, resolves to `data_asset="surgery_data"`. The thing being read
   or the user's intent almost always determines the asset.
2. **Enumerate when unsure.** Call `list_supported_data_assets_tool` to see the registered assets
   (`value` is the `data_asset` argument; `data_asset_class` is the dataclass it resolves to). With one
   registered asset today, the choice is unambiguous.
3. **Prompt the user only as a fallback** — when the asset genuinely cannot be inferred and the list
   does not disambiguate, or when the user is explicitly choosing. Prefer automatic resolution to
   prompting.

An invalid `data_asset` returns an error naming the valid `ReadAssets` values, so a wrong guess fails
loudly rather than silently mis-parsing.

---

## Worked example: surgery data (`data_asset="surgery_data"`)

Surgery data is the platform's current read asset: one `SurgeryData` YAML (`surgery_metadata.yaml`) is
the complete record of a **surgical intervention performed on a research animal prior to data
acquisition** — what was done, by whom, when, with which drugs, and what was implanted or injected
where. One file per animal-surgery, aggregating every surgery-time fact into five sections.

### The `SurgeryData` model

`SurgeryData` is a **monolithic single-file record** with five nested sections; none is an independent
on-disk artifact. Callers project the sections they need from the returned `data` dict.

| Section      | Nested dataclass  | Captures                                                                                        |
|--------------|-------------------|-------------------------------------------------------------------------------------------------|
| `subject`    | `SubjectData`     | id, ear-punch, sex, genotype, date of birth, pre-surgery weight, cage, housing location, status |
| `procedure`  | `ProcedureData`   | Surgery start/end timestamps, surgeon, protocol, surgery + post-op notes, quality               |
| `drugs`      | `DrugData[]`      | Per-drug: descriptive name, volume (mL), manufacturer or reference code                         |
| `implants`   | `ImplantData[]`   | Per-implant: name, target region, manufacturer code, AP/ML/DV stereotactic coordinates          |
| `injections` | `InjectionData[]` | Per-injection: name, target, volume (nL), manufacturer code, AP/ML/DV stereotactic coordinates  |

`drugs`, `implants`, and `injections` are **lists** (0..N entries); `subject` and `procedure` are singletons.
Section projection is **caller-side**: the returned `data` dict has top-level keys `subject`, `procedure`,
`drugs`, `implants`, `injections` — pick the one you need. There are no per-section MCP tools.

**Units and encodings** (call `describe_data_asset_schema_tool(data_asset="surgery_data")` for the
authoritative schema): timestamps (`*_us`) are `int` microseconds since the UTC epoch; `weight_g` is
`float` grams; drug volumes are `float` mL; `injection_volume_nl` is `float` nL (note the mL/nL
mismatch); stereotactic coordinates (`*_ap/ml/dv_coordinate_mm`) are `float` mm relative to bregma;
`surgery_quality` is `int` 0–3 (0 = unusable … 3 = high-tier publication-grade, default 0); `cage` and
subject `id` are `int`; all other identifiers/codes/notes are strings.

### Provenance and file locations

Surgery data is **animal-scoped** — an animal accumulates records within whichever project owns it
(each animal belongs to exactly one project; see `/project-hierarchy`). The authoritative source is the
upstream Google Sheet; the MCP layer **does not query it at runtime** and only reads the YAML file the
caller points at. `surgery_metadata.yaml` (`RawDataFiles.SURGERY_METADATA`) is materialized into:

| Location                                        | Populated by                                                        | Discovery path                                                                                        |
|-------------------------------------------------|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `<session>/raw_data/surgery_metadata.yaml`      | Acquisition runtime at session start (snapshot of the Google Sheet) | `SessionData.raw_data.surgery_metadata_path`; `inspect_sessions_tool` lists it under `raw_data_files` |
| `<dataset_root>/<animal>/surgery_metadata.yaml` | Forging pipeline (from the animal's latest session)                 | `DatasetData.surgery_paths` (owned by `forging:datasets` skill)                         |

All copies are **snapshots** of the Google Sheet state when their pipeline ran; none is a live view,
and a write to one does not update any sibling copy or flow back to the sheet.

---

## Workflows

Every workflow reduces to: **resolve the `data_asset` and the `file_path` → read / write**. The
`data_asset` is resolved per [Resolving the data_asset](#resolving-the-data_asset); the `file_path`
resolution depends on where the file lives.

### Reading a read asset

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`); the target
   file exists at the path you will pass.
2. **Resolve the `data_asset`** (infer from context; `list_supported_data_assets_tool` if unsure).
3. **Resolve the `file_path`** via the owning hand-off — for surgery: session snapshot via
   `/project-hierarchy` / `/session-discovery` (confirm with `inspect_sessions_tool` in
   `/session-data`), or the dataset per-animal copy via `forging:datasets`
   (`DatasetData.surgery_paths`), or an ad-hoc path the user supplies.
4. **Read:** `read_data_asset_tool(file_path="<absolute path>", data_asset="<asset>")`.
5. **Project the section(s)** the user asked about from `response["data"]`, then report. If the read
   came from a snapshot, remind the user the values reflect the upstream state when the copy was written.

### Amending a read asset in place

Use when a single copy has an error and a durable upstream correction is not warranted or not yet
available. The amendment affects only the one file whose path is passed.

1. **Resolve `data_asset` and `file_path`** as above.
2. **Read the current record** so you mutate a validated baseline:
   `read_data_asset_tool(file_path="<absolute path>", data_asset="<asset>")`.
3. **Mutate `response["data"]`** in memory — change only the fields that need correcting; keep every
   other section intact (the write replaces the full record).
4. **Write back to the same path:**
   `write_data_asset_tool(file_path="<absolute path>", data_asset="<asset>", data_asset_payload=<mutated dict>)`.
   The payload is validated against the resolved dataclass before overwriting, so a malformed edit
   fails without damaging the file.
5. **Tell the user which copy was amended** and that the change does not propagate to sibling copies or
   to the upstream source.

### Amendment vs. upstream correction (surgery example)

| Scenario                                                                            | Use                                                                                      |
|-------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| One session's snapshot has a data-entry error; re-acquiring is overkill             | `write_data_asset_tool` on the session file                                              |
| A dataset's per-animal copy is wrong                                                | `write_data_asset_tool` on the dataset file                                              |
| The same field is wrong in both the session snapshot and the dataset copy           | `write_data_asset_tool` against each file separately — there is no propagation           |
| A field is wrong for the animal itself and should be right for every future capture | Edit the upstream Google Sheet; next acquisition (and downstream forge) captures the fix |
| Both a past file and future captures need fixing                                    | Do both — write tool for the existing file(s), the source for future captures            |

The MCP layer never pushes an amendment back upstream; copies stay separate until the next capture.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] data_asset resolved automatically where possible (inferred from context or list_supported_data_assets_tool);
      the user was prompted only when it could not be inferred
- [ ] file_path resolved via the right owner (/session-data or /session-discovery for session snapshots,
      forging:datasets for dataset copies, or the user directly for ad-hoc paths) and is absolute
- [ ] The target file exists at the resolved path
- [ ] Section extraction was done caller-side from response["data"]["<section>"] — no per-section MCP tool exists
- [ ] If writing: the full payload was supplied to write_data_asset_tool — no partial-update path exists
- [ ] If writing: the user was told which single copy was amended and that the change does not propagate
      to sibling copies or to the upstream source
```

---

## Related skills

| Skill                                         | Relationship                                                                                   |
|-----------------------------------------------|------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`               | Run first if the MCP server is not connected                                                   |
| `/working-directory`                          | Required prerequisite — bootstraps the working directory AND the Google credentials path       |
| `/library-extension`                          | Adds a **new** read asset (dataclass + `ReadAssets` member + `READ_ASSET_REGISTRY` entry)      |
| `/project-hierarchy`                          | Owns `get_data_root_overview_tool` and the project tree walk; enumerates animals               |
| `/session-discovery`                          | Resolves session roots for session-snapshot paths                                              |
| `/session-data`                               | Owns `inspect_sessions_tool` that classifies read-asset files under a session                  |
| `/session-descriptors`                        | Sibling — descriptors capture per-session runtime state (separate from read assets)            |
| `forging:datasets`                    | Resolves dataset per-animal `surgery_metadata.yaml` paths (`DatasetData.surgery_paths`)        |
| `experiment:google-sheets-processing` | Owns the reader that captures a read asset from its external source into the on-disk dataclass |
