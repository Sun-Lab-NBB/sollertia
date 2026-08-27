---
name: data-assets
description: >-
  Reads, writes, and describes on-disk read-asset dataclasses via the sollertia-shared-assets MCP server's generic
  data-asset tools, dispatched by a `data_asset` identifier resolved from the platform's read-asset registry. Surgery
  data (`surgery_data` → `SurgeryData`) is the current worked example. Use when looking up or amending an animal's
  surgical history, or any other registered read asset.
user-invocable: false
---

# Sollertia data assets

Reads, writes, and describes the platform's **read assets** via the `slsa mcp` MCP server. A read asset is metadata the
platform reads from an external source (e.g., a Google Sheet) and caches on disk as a typed dataclass. The canonical set
is the `READ_ASSET_REGISTRY` in `sollertia-shared-assets` (see `/library-extension`). This skill is the **exclusive**
owner of `read_data_asset_tool`, `write_data_asset_tool`, and `describe_data_asset_schema_tool`.

Every tool is keyed by a **`data_asset`** identifier (a `ReadAssets` value such as `"surgery_data"`) that selects the
on-disk dataclass. The interface is generic, with one tool set serving every registered read asset, so new assets become
available here automatically once registered, with no new tools. **Surgery data is the current worked example** and the
only registered read asset today. The rest of this skill uses it to illustrate the generic flow.

---

## Scope

**Covers:**
- Reading any read-asset YAML (full payload) via `read_data_asset_tool(file_path, data_asset)`
- Amending any read-asset YAML in place, or authoring a new copy at a path that does not yet exist, via
  `write_data_asset_tool(file_path, data_asset, data_asset_payload)` (full-record replacement that does not propagate to
  the upstream source or to any other copy)
- Schema introspection via `describe_data_asset_schema_tool(data_asset)`
- Enumerating the registered read assets via `list_supported_data_assets_tool`
- Resolving the `data_asset` identifier, preferring automatic resolution as described below
- The surgery-data worked example: its `SurgeryData` model, provenance, and file locations

**Does not cover:**
- Adding a **new** read asset, meaning the dataclass, the `ReadAssets` member, and the registry entry, which
  `/library-extension` owns under "Adding a new read asset"
- Durable corrections to an asset's upstream source. For read assets captured from a Google Sheet, the authoritative
  source is the sheet. The MCP layer **does not query it at runtime**, and writes here amend only the one on-disk file
  whose path the caller passes. Edit the upstream source for a fix that should apply to every future capture.
- Propagating an amendment from one copy of the file to another, because copies are independent on disk
- Partial or per-section updates, because there is no partial-update tool. To change one field, read the current file,
  mutate the returned dict, then write it back whole.
- Resolving the canonical `file_path` for you. The caller, or a collaborating skill, supplies the absolute path, and
  this skill only reads and writes.
- Discovering animals (see `/project-hierarchy`)
- Reading session-level data (see `/session-data`, `/session-descriptors`, `/session-hardware-state`)

---

## MCP tool surface

| Tool                              | Purpose                                                                                             |
|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| `read_data_asset_tool`            | Loads a read-asset YAML, parsing it with the dataclass for the given `data_asset` (exclusive)       |
| `write_data_asset_tool`           | Writes a validated full payload to a path, validated against the `data_asset` dataclass (exclusive) |
| `describe_data_asset_schema_tool` | Returns the schema (nested dataclasses recursed inline) for the given `data_asset` (exclusive)      |
| `list_supported_data_assets_tool` | Enumerates the registered read assets, giving `value`, `name`, and `data_asset_class` for each      |

Signatures: `read_data_asset_tool(file_path, data_asset)`,
`write_data_asset_tool(file_path, data_asset, data_asset_payload, *, overwrite=True)`, and
`describe_data_asset_schema_tool(data_asset)`. The `file_path` is always the caller's responsibility, and the
`data_asset` selects the schema. There is no partial-update tool, so to change a single field, read the current file,
mutate the returned dict, and write it back whole.

This skill is where the read pattern for `list_supported_data_assets_tool` is documented and where every other skill
hands off for it. The tool is read-only, takes no parameters, and lives on the same `slsa mcp` server as the other
three. It returns its entries under `data_assets`, and each entry's `value` is the `data_asset` argument the other three
tools take.

`overwrite` is keyword-only and defaults to `True`, so a write replaces an existing file silently. Passing
`overwrite=False` refuses the write instead and returns
`Unable to write <Class> to <path>: a file already exists at this path. Pass overwrite=True to replace it.`

What a `write_*` tool actually validates, and why every amendment MUST be a read-mutate-write of the complete record,
is documented in the `## Response contract` section of `/assets-mcp-environment-setup`. `SurgeryData` defines no
`__post_init__`, so no value checking runs for surgery data and the write is a shape check only.

For downstream Python callers working in code rather than through MCP, `resolve_read_asset` is the code-path counterpart
of `read_data_asset_tool`'s dispatch and is documented in `/library-extension`.

### Response payloads

`read_data_asset_tool` and `write_data_asset_tool` both return `success`, `file_path`, `data`, `data_asset_class` (the
resolved dataclass name), and `data_asset` (the caller's input echoed back). The last two are attached only on success,
so a failure envelope carries neither and cannot be used to confirm which class the dispatch picked.
`describe_data_asset_schema_tool` returns `data_asset`, `data_asset_class`, and `schema`. It is the one describe tool in
the module that names the resolved class, so a caller can confirm the dispatch without a second call. The shape of the
`schema` payload is documented in the `## Response contract` section of `/assets-mcp-environment-setup`.

### Failure modes

The `## Response contract` section of `/assets-mcp-environment-setup` documents the envelope that carries every
response. Four failure messages are shared with every other read and write tool on the server, and a caller routes on
the message text:

- **The file does not exist.** `Unable to read <Class> from <path>: the file does not exist.`
- **The file does not parse as the resolved class.** `Unable to load <path> as <Class>: <exception>`
- **The payload failed validation.** `Unable to validate the payload as <Class>: <exception>`
- **The validated record could not be persisted.** `Unable to persist <Class> to <path>: <exception>`

The fifth failure, an invalid `data_asset`, is covered under [Resolving the `data_asset`](#resolving-the-data_asset).

---

## Resolving the `data_asset`

`data_asset` is **required** and carries no default. Resolve it **automatically** wherever possible and only fall back
to asking the user:

1. **Infer from context first.** A `surgery_metadata.yaml` file, or a request about an animal's surgical history /
   implants / drugs, resolves to `data_asset="surgery_data"`. The thing being read or the user's intent almost always
   determines the asset.
2. **Enumerate when unsure.** Call `list_supported_data_assets_tool` to see the registered assets (`value` is the
   `data_asset` argument, and `data_asset_class` is the dataclass resolved from it). With one registered asset today,
   the choice is unambiguous.
3. **Prompt the user only as a fallback**, when the asset genuinely cannot be inferred and the list does not
   disambiguate, or when the user is explicitly choosing. Prefer automatic resolution to prompting.

An invalid `data_asset` returns an error naming the valid `ReadAssets` values, so a wrong guess fails loudly rather than
silently mis-parsing.

---

## Worked example: surgery data (`data_asset="surgery_data"`)

Surgery data is the platform's current read asset. One `SurgeryData` YAML (`surgery_metadata.yaml`) is the complete
record of a **surgical intervention performed on a research animal prior to data acquisition**, covering what was done,
by whom, when, with which drugs, and what was implanted or injected where. There is one file per animal-surgery,
aggregating every surgery-time fact into five sections.

### The `SurgeryData` model

`SurgeryData` is a **monolithic single-file record** with five nested sections, none of which is an independent on-disk
artifact. Callers project the sections they need from the returned `data` dict.

| Section      | Nested dataclass  | Captures                                                                                        |
|--------------|-------------------|-------------------------------------------------------------------------------------------------|
| `subject`    | `SubjectData`     | id, ear-punch, sex, genotype, date of birth, pre-surgery weight, cage, housing location, status |
| `procedure`  | `ProcedureData`   | Surgery start/end timestamps, surgeon, protocol, surgery + post-op notes, quality               |
| `drugs`      | `DrugData[]`      | Per-drug: descriptive name, volume (mL), manufacturer or reference code                         |
| `implants`   | `ImplantData[]`   | Per-implant: name, target region, manufacturer code, AP/ML/DV stereotactic coordinates          |
| `injections` | `InjectionData[]` | Per-injection: name, target, volume (nL), manufacturer code, AP/ML/DV stereotactic coordinates  |

`drugs`, `implants`, and `injections` are **lists** holding zero or more entries, while `subject` and `procedure` are
singletons. Section projection is **caller-side**. The returned `data` dict has top-level keys `subject`, `procedure`,
`drugs`, `implants`, and `injections`, so pick the one you need. There are no per-section MCP tools.

**Units and encodings.** Call `describe_data_asset_schema_tool(data_asset="surgery_data")` for the authoritative schema.
Timestamps (`*_us`) are `int` microseconds since the UTC epoch, `weight_g` is `float` grams, drug volumes are `float`
mL, and `injection_volume_nl` is `float` nL, so note the mL against nL mismatch. Stereotactic coordinates
(`*_ap/ml/dv_coordinate_mm`) are `float` mm relative to bregma. `surgery_quality` is an `int` from 0 to 3, where 0 is
unusable and 3 is high-tier publication-grade, defaulting to 0. `cage` and subject `id` are `int`, and all other
identifiers, codes, and notes are strings.

### Provenance and file locations

Surgery data is **animal-scoped**, so an animal accumulates records within whichever project owns it, and each animal
belongs to exactly one project (see `/project-hierarchy`). The authoritative source is the upstream Google Sheet. The
MCP layer **does not query it at runtime** and only reads the YAML file whose path the caller passes.
`surgery_metadata.yaml` (`RawDataFiles.SURGERY_METADATA`) is materialized into:

| Location                                        | Populated by                                                                       | Discovery path                                                                  |
|-------------------------------------------------|------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| `<session>/raw_data/surgery_metadata.yaml`      | Acquisition-system preprocessing (`snapshot_surgery_data`), after the session ends | `SessionData.raw_data.surgery_metadata_path`, reported under `raw_data_files`   |
| `<dataset_root>/<animal>/surgery_metadata.yaml` | Forging pipeline (from the animal's latest session)                                | `DatasetData.animals`, then `DatasetAnimal.surgery_path` (owned by `/datasets`) |

Prefer the MCP route to path arithmetic on either side. `inspect_sessions_tool` (`/session-data`) reports the session
copy under `raw_data_files`, and `inspect_datasets_tool` (`/datasets`) reports each animal's `animal_path` together with
its `surgery_metadata` artifact entry.

All copies are **snapshots** of the Google Sheet state when their pipeline ran, and none is a live view. A write to one
does not update any sibling copy or flow back to the sheet.

`surgery_metadata.yaml` is never a required raw asset, and neither the session inventory nor the dataset inventory
flags its absence. The session copy does not exist until preprocessing runs, so its absence on a session that was
acquired but not yet preprocessed is normal rather than a sign of corruption. Route that case to
`experiment:data-management`, which runs preprocessing. A file still missing after preprocessing means the capture
skipped the animal, so route that case to `experiment:google-sheets-processing`, which owns the capture, rather than to
a repair write here.

Each preprocessing run rewrites the session copy from the Google Sheet, so a later run replaces any amendment made here
with `write_data_asset_tool`. Amend the session copy only after preprocessing has produced it, and re-apply the
amendment if the session is preprocessed again.

---

## Workflows

Every workflow reduces to: **resolve the `data_asset` and the `file_path`, then read or write**. The `data_asset` is
resolved per [Resolving the data_asset](#resolving-the-data_asset), and the `file_path` resolution depends on where the
file lives.

### Reading a read asset

1. **Verify prerequisites:** the MCP server is connected, else `/assets-mcp-environment-setup`, and the target file
   exists at the path you will pass.
2. **Resolve the `data_asset`**, inferring it from context, or calling `list_supported_data_assets_tool` if unsure.
3. **Resolve the `file_path`** via the owning hand-off. For surgery data: the session snapshot via `/project-hierarchy`
   and `/session-discovery` (confirm with `inspect_sessions_tool` in `/session-data`), the dataset per-animal copy via
   `/datasets` (`inspect_datasets_tool` reports each animal's `animal_path` and its `surgery_metadata` entry), or an
   ad-hoc path the user supplies.
4. **Read:** `read_data_asset_tool(file_path="<absolute path>", data_asset="<asset>")`.
5. **Project the section(s)** the user's question names from `response["data"]`, then report. If the read came from a
   snapshot, remind the user the values reflect the upstream state when the copy was written.

### Amending a read asset in place

Use when a single copy has an error and a durable upstream correction is not warranted or not yet available. The
amendment affects only the one file whose path is passed.

1. **Resolve `data_asset` and `file_path`** as above.
2. **Read the current record** so you mutate a validated baseline:
   `read_data_asset_tool(file_path="<absolute path>", data_asset="<asset>")`.
3. **Mutate `response["data"]`** in memory. Change only the fields that need correcting, and keep every other section
   intact, as the write contract in `/assets-mcp-environment-setup` requires.
4. **Write back to the same path:**
   `write_data_asset_tool(file_path="<absolute path>", data_asset="<asset>", data_asset_payload=<mutated dict>)`.
   `SurgeryData` defines no `__post_init__`, so nothing checks the values themselves.
5. **Tell the user which copy was amended** and that the change does not propagate to sibling copies or to the upstream
   source.

### Amendment vs. upstream correction (surgery example)

| Scenario                                                                            | Use                                                                                              |
|-------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| One session's snapshot has a data-entry error, and re-preprocessing is overkill     | `write_data_asset_tool` on the session file                                                      |
| A dataset's per-animal copy is wrong                                                | `write_data_asset_tool` on the dataset file                                                      |
| The same field is wrong in both the session snapshot and the dataset copy           | `write_data_asset_tool` against each file separately, because there is no propagation            |
| A field is wrong for the animal itself and should be right for every future capture | Edit the upstream Google Sheet, and the next preprocessing run, plus the forge, captures the fix |
| Both a past file and future captures need fixing                                    | Do both, using the write tool for the existing file(s) and the source for future captures        |

The MCP layer never pushes an amendment back upstream, and copies stay separate until the next capture.

---

## Related skills

| Skill                                 | Relationship                                                                                                                                                      |
|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/cli-reference`                      | Reference: the `slsa` commands available while the MCP server is down                                                                                             |
| `/assets-mcp-environment-setup`       | Run first if the MCP server is not connected                                                                                                                      |
| `/working-directory`                  | Bootstraps the working directory and the Google credentials required by the preprocessing-side capture. The data-asset tools take absolute paths and need neither |
| `/library-extension`                  | Adds a **new** read asset (dataclass + `ReadAssets` member + `READ_ASSET_REGISTRY` entry) and owns `resolve_read_asset`                                           |
| `/project-hierarchy`                  | Owns `get_data_root_overview_tool` and the project tree walk, and enumerates animals                                                                              |
| `/session-discovery`                  | Resolves session roots for session-snapshot paths                                                                                                                 |
| `/session-data`                       | Owns `inspect_sessions_tool` that classifies read-asset files under a session                                                                                     |
| `/session-descriptors`                | Sibling whose descriptors capture per-session runtime state, held separately from read assets                                                                     |
| `/datasets`                           | Owns `inspect_datasets_tool` and resolves the dataset per-animal `surgery_metadata.yaml` path                                                                     |
| `experiment:data-management`          | Runs the preprocessing that writes the session copy of `surgery_metadata.yaml` after the session ends                                                             |
| `experiment:google-sheets-processing` | Owns the reader that captures a read asset from its external source into the on-disk dataclass                                                                    |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] data_asset resolved automatically where possible (inferred from context or list_supported_data_assets_tool);
      the user was prompted only when it could not be inferred
- [ ] file_path resolved via the right owner (/session-data or /session-discovery for session snapshots,
      /datasets for dataset copies, or the user directly for ad-hoc paths) and is absolute
- [ ] The target file exists at the resolved path
- [ ] Section extraction was done caller-side from response["data"]["<section>"], since no per-section MCP tool exists
- [ ] If writing: the full payload was supplied to write_data_asset_tool, since no partial-update path exists
- [ ] If writing: the user was told which single copy was amended and that the change does not propagate
      to sibling copies or to the upstream source
```
