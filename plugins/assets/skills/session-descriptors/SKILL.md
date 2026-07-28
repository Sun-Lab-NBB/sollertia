---
name: session-descriptors
description: >-
  Reads, writes, and validates per-system session descriptor YAMLs via the sollertia-shared-assets
  MCP server. Descriptors are keyed on SessionTypes via DESCRIPTOR_REGISTRY; the parsing class
  varies by session type while the file-path contract and tools stay generic. Owns
  write_session_descriptor_tool and describe_session_descriptor_schema_tool; tools are file-path
  based, accepting raw session snapshots or forged dataset copies. Use when repairing, amending, or
  inspecting a descriptor for any session type. For Mesoscope-VR's concrete descriptor schema, see
  mesoscope:mesoscope-vr-session-schema.
user-invocable: false
---

# Sollertia session descriptors

Reads, writes, and validates per-system session descriptor YAML files via the `slsa mcp` MCP
server. This skill is the **exclusive** owner of `write_session_descriptor_tool` and
`describe_session_descriptor_schema_tool` — no other skill in the marketplace may call these.

Descriptors are treated as **standalone per-session records** keyed on their schema (the
descriptor dataclass that matches a given `SessionTypes` value via `DESCRIPTOR_REGISTRY`). The
tools operate on whatever absolute `file_path` the caller supplies; they do not care whether that
path points at a raw session snapshot, a forged dataset copy, or an ad-hoc location. The caller is
responsible for resolving the path and supplying `session_type` so the right dataclass is used to
parse or validate the file. This skill owns the **generic per-system descriptor contract**; the
concrete field-level schema for any one system lives in that system's schema skill (for
Mesoscope-VR, `mesoscope:mesoscope-vr-session-schema`).

---

## Scope

**Covers:**
- Reading any `session_descriptor.yaml` (any supported session type) via
  `read_session_descriptor_tool`
- Writing or repairing any `session_descriptor.yaml` via `write_session_descriptor_tool`
  (validated full-record replacement; does not propagate to any sibling copy of the same
  descriptor)
- Schema introspection via `describe_session_descriptor_schema_tool`
- The relationship between `SessionTypes` enum values and descriptor classes
- Guidance on where `session_descriptor.yaml` is expected to live (raw session snapshot and
  forged dataset per-session copy) and how callers resolve those paths

**Does not cover:**
- Propagating an amendment from one copy of a descriptor to another. The raw session snapshot
  and the forged dataset copy are independent files on disk — writing to one does not update
  any other.
- Partial/per-field updates — the write tool validates and replaces the **full** descriptor
  payload. To change one field, read the current file, mutate the returned dict, then write
  it back whole.
- Resolving the canonical path for you. The caller (or a collaborating skill) supplies the
  absolute `file_path`; this skill only reads and writes.
- Documenting any one system's concrete descriptor field-level schema (descriptor classes, field
  names, types, defaults). For Mesoscope-VR, see `mesoscope:mesoscope-vr-session-schema`; for other
  systems, call `describe_session_descriptor_schema_tool` against the resolved system.
- Reading the `SessionData` marker file (see `/session-data`)
- Reading or writing the per-session hardware-state snapshot (`hardware_state.yaml`) (see
  `/session-hardware-state`)
- Reading the frozen `ZaberPositions` and `MesoscopePositions` snapshots (see the experiment
  plugin's `mesoscope:mesoscope-vr-snapshots`)
- Reading subject metadata (see `/data-assets`)
- Discovering sessions (see `/project-hierarchy` for `get_data_root_overview_tool`)
- Dataset assembly and dataset-level descriptor path resolution (see the forging plugin's
  `forging:datasets` skill)

---

## Session types and descriptor classes

The raw session copy and the forged dataset copy both use the filename `session_descriptor.yaml`
regardless of session type. The YAML parses into a session-type-specific descriptor dataclass, and
the library's `DESCRIPTOR_REGISTRY` maps each `SessionTypes` value → its descriptor class. **Only
the parsing class varies by session type.** The file-path contract, the full-record-replacement
semantics, and the MCP tool surface are all generic across systems and session types.

Each descriptor captures the **per-session** metadata that varies between sessions of the same type
(reward volume actually delivered, water restriction status, observed behavior summary,
experimenter notes). It is **distinct from** `SessionData`, which is the canonical session marker
file.

This skill is system-agnostic and does not enumerate the concrete descriptor classes or their
fields. To learn which descriptor class a given session type resolves to and what fields it
carries, call `describe_session_descriptor_schema_tool` for the session's resolved system (see
**What a descriptor stores and how it is used** below). For Mesoscope-VR's concrete descriptor
schema — the per-session-type descriptor classes and their exact field names, types, and defaults —
see `mesoscope:mesoscope-vr-session-schema`.

To enumerate the canonical `SessionTypes` strings, hand off to `/session-data` for the
`list_supported_session_types_tool` call.

---

## What a descriptor stores and how it is used

This section orients an agent before it reads or amends a descriptor, without committing to any
one system's field set. The concrete fields a descriptor carries depend on the session's resolved
system and session type, so **the schema tool is the canonical field reference, not this skill**:
call `describe_session_descriptor_schema_tool` for the session's resolved system to obtain the
exact field names, types, and defaults for a given session type. For Mesoscope-VR's concrete
descriptor schema, see `mesoscope:mesoscope-vr-session-schema`.

Regardless of system, every descriptor is a **per-session provenance and bookkeeping record** —
who ran the session, how the animal behaved, how much water it received, and whether the runtime
completed cleanly. A descriptor typically surfaces a durable **data-quality** signal (commonly an
`incomplete` field): it means the session ran past initialization but hit a runtime issue and may
have data gaps, yet still holds real data and should not be purged. This is distinct from the
`nk.bin` **uninitialized** marker described in `/session-data`: `nk.bin` presence means the runtime
never finished initializing the session at all, so there is no data of value and the session is a
purge target. The two signals are orthogonal and are surfaced as independent keys (`uninitialized`
and `incomplete`) by every status-reporting MCP tool. Confirm the exact field set against the
schema tool before relying on any specific field.

### Lifecycle (high level)

The acquisition runtime is the only authorized **primary** writer of the raw-session copy:

1. At session start, it builds a precursor descriptor seeded with the known starting metadata for
   that session and system. Some systems seed session-type-specific values (such as training
   thresholds) from the previous session's descriptor cached at the per-animal `persistent_data/`
   slot (see `/project-hierarchy`), so those values carry across days without manual re-entry.
2. At session end, the runtime records the runtime-collected fields and flips the data-quality
   signal (`incomplete`) to its clean-end value.
3. The runtime then blocks shutdown until the experimenter replaces any default placeholder text
   (such as `experimenter_notes`) with real notes.
4. After verification, the descriptor is copied to the per-animal persistent cache, which
   becomes the seed for the next session of the same type.

For the concrete per-system field set that participates in this lifecycle (which fields are
seeded, which are runtime-recorded), call `describe_session_descriptor_schema_tool`; for
Mesoscope-VR specifically, see `mesoscope:mesoscope-vr-session-schema`.

The forging pipeline later writes a **secondary, independent** copy into the forged dataset
hierarchy next to each session's `data.feather` (see **Known file locations** below). That
copy is frozen at the moment the dataset was assembled and is read by downstream analysis code
that consumes forged datasets without reaching back into the raw session.

`write_session_descriptor_tool` (this skill) is for **post-acquisition repair or amendment** of
whichever copy the caller supplies a path to. The acquisition runtime never goes through slsa
MCP, and neither does forging's initial copy step.

### Post-creation consumers

After a descriptor exists on disk, downstream code reads it for several generic purposes; the
specific fields each consumer routes are system-dependent, so defer to the consumer's owning skill
(and the system's schema skill) for exact field names:

- **Cross-session continuity** — the next session's precursor reads the per-animal
  persistent-cache copy to inherit carried-over values from the previous session of the same type.
- **Per-animal record logging** — the acquisition system's preprocessing step reads
  session-summary fields from the raw session copy and writes them to the relevant per-animal log.
- **Project manifest assembly** — the forging library reads the descriptor's status fields from
  the raw session copy for the project-level manifest.
- **Dataset eligibility and inclusion** — the forging library's dataset assembly verifies the
  descriptor exists in the raw session and copies it next to `data.feather` in the forged
  dataset so downstream analysis has experimenter context without touching raw data.

For exact paths, fields, and call sites, defer to the owning skills (experiment plugin's data
management, forging plugin's processing and forging skills) and the system's schema skill
(`mesoscope:mesoscope-vr-session-schema` for Mesoscope-VR). Descriptors themselves are primarily a
**provenance and bookkeeping record** — what happened, who ran it, how much water the animal got —
not an input to numerical processing.

### Known file locations

The descriptor is written by different pipelines into several canonical locations. The raw session
snapshot and the forged dataset copy both carry the filename `session_descriptor.yaml`
(`RawDataFiles.SESSION_DESCRIPTOR`), while the per-animal persistent cache names its file after the
session type. All of them hold the same schema and are read and written by the same tools, and this
skill does not distinguish between them beyond helping the caller resolve the right path.

| Location                                                    | Populated by                                                                   | Discovery path                                                                                                                        |
|-------------------------------------------------------------|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `<session>/raw_data/session_descriptor.yaml`                | Acquisition runtime at session end (primary on-disk copy)                      | Session root from `/session-discovery`. `inspect_sessions_tool` (`/session-data`) confirms presence in its `raw_data_files` inventory |
| `<dataset_root>/<animal>/<session>/session_descriptor.yaml` | Forging pipeline (copy alongside `data.feather` at dataset assembly)           | Forging plugin's `forging:datasets` resolves the forged-session layout under the dataset root                                         |
| `<animal>/persistent_data/<session-type>_descriptor.yaml`   | Acquisition runtime (per-animal cache used to seed the next same-type session) | `/project-hierarchy`                                                                                                                  |

Other locations are possible, because the tools take any absolute path, but the three above are the
ones populated automatically. The dataset root is the directory that holds the dataset's
`dataset.yaml` marker, which `DatasetData` re-derives as `dataset_data_path.parent` on every load.
The forged copy lives **directly under the session directory inside the dataset root**, not under a
`raw_data/` subdirectory, so the path shape does **not** match `<session>/raw_data/...` for forged
copies. The persistent cache sits at `<root>/<project>/<animal>/persistent_data/`
(`AnimalData.persistent_data_path`) and holds one descriptor per session type, so its filename
encodes the session type. In `<session-type>_descriptor.yaml` the placeholder stands for a
snake_case name the acquisition system hardcodes. Substitute the exact filename that system uses,
because `SessionTypes` values carry spaces. For Mesoscope-VR those filenames are
`lick_training_descriptor.yaml`, `run_training_descriptor.yaml`,
`mesoscope_experiment_descriptor.yaml`, and `window_checking_descriptor.yaml`.

All copies are **snapshots** frozen at the moment their pipeline wrote them; none is a live
view. Copies are not strictly immutable: `write_session_descriptor_tool` can amend any one file
in place. That amendment is local to the one file whose path was passed — it does not flow
back to any sibling copy (raw session ↔ forged dataset ↔ persistent cache). For a correction
that should apply everywhere, amend each file explicitly.

---

## MCP tool surface

| Tool                                      | Purpose                                                                                                    |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `read_session_descriptor_tool`            | Loads a descriptor file at an explicit `file_path`, parsing it with the class for the given `session_type` |
| `write_session_descriptor_tool`           | Writes a validated full descriptor payload to a `file_path` (exclusive). Defaults to `overwrite=True`      |
| `describe_session_descriptor_schema_tool` | Returns the field schema for the descriptor dataclass of a given session type (exclusive)                  |

All three tools take an explicit `session_type` — class selection is the caller's
responsibility. `read_session_descriptor_tool` and `write_session_descriptor_tool` additionally
take `file_path`; path resolution is also the caller's responsibility. See **Known file
locations** above for the canonical paths and which neighboring skill owns each.

Path-resolution hand-offs:
- Raw session snapshot → `/project-hierarchy` + `/session-discovery` for session roots;
  `/session-data` (`inspect_sessions_tool`) to confirm the file is present.
- Forged dataset per-session copy → `forging:datasets` skill, which resolves
  the dataset-hierarchy layout.
- Per-animal persistent cache → `/project-hierarchy`.
- Ad-hoc location → the user supplies the path directly.

If you don't know the session type for a given file and it's a raw session snapshot, hand off
to `/session-data`. Call `inspect_sessions_tool` on the session root and read
`identity.session_type` from the report, or read the marker directly with
`read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For forged dataset
copies (or ad-hoc paths) where no sibling `session_data.yaml` is reachable, the caller must
supply `session_type` directly; `forging:datasets` skill can surface the
session-type mapping stored in the dataset marker.

`write_session_descriptor_tool` also accepts `descriptor_payload: dict[str, Any]` (the full
record) and a keyword-only `overwrite: bool = True`. The payload is validated against the
descriptor dataclass before anything is written, so a bad payload fails without touching the
file. There is no partial-update tool; to change a single field, read the current file, mutate
the returned dict, and write it back whole.

`describe_session_descriptor_schema_tool` returns two keys: `session_type` (the validated enum
value) and `schema` (the field schema of the session-type's descriptor dataclass).

---

## Workflows

All workflows reduce to: **resolve the path → pick the session type → read / write**. The
resolution step changes depending on where the file lives; the read/write step is the same
everywhere.

### Reading a descriptor (generic)

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`), and the
   target descriptor file exists at the path you are about to pass.
2. **Resolve the `file_path`** using the hand-off that matches the container:
   - **Raw session snapshot** → session root from `/project-hierarchy` or
     `/session-discovery`; optionally confirm the file is present via
     `inspect_sessions_tool` (`/session-data`). Path is
     `<session>/raw_data/session_descriptor.yaml`.
   - **Forged dataset copy** → hand off to `forging:datasets` skill for the
     per-session path inside the dataset root. Path shape is
     `<dataset_root>/<animal>/<session>/session_descriptor.yaml`.
   - **Per-animal persistent cache** → hand off to `/project-hierarchy`. Path shape is
     `<animal>/persistent_data/<session-type>_descriptor.yaml`, with the system's hardcoded
     snake_case filename in place of the placeholder (see **Known file locations**).
   - **Ad-hoc location** → the user supplies the path directly.
3. **Determine `session_type`.**
   - If the file sits in a raw session snapshot, hand off to `/session-data` — either call
     `inspect_sessions_tool(session_paths=["<session root>"])` and read
     `identity.session_type` from the report, or read the marker directly with
     `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`.
   - Otherwise, the caller supplies `session_type` directly (e.g., from a dataset marker or
     from the user).
4. **Read the descriptor:**
   ```text
   read_session_descriptor_tool(
       file_path="<absolute path to the descriptor file>",
       session_type="<type>",
   )
   ```
5. **Project or report the fields the user asked about** from `response["data"]`. If the read
   came from a snapshot, remind the user the values reflect the state at the moment that copy
   was written; they do not live-track any sibling copy.

### Amending a descriptor in place

Use this when a single copy has an error and a durable correction to every other copy is
either not warranted or will be handled separately. The amendment affects only the one file
whose path is passed.

1. **Resolve the `file_path` and `session_type`** as in the read workflow.
2. **Read the current record** so you mutate a validated baseline rather than constructing one
   from scratch:
   ```text
   read_session_descriptor_tool(file_path="<absolute path>", session_type="<type>")
   ```
3. **Inspect the schema** if you need to add fields or double-check types:
   ```text
   describe_session_descriptor_schema_tool(session_type="<type>")
   ```
4. **Mutate the returned `response["data"]` dict** in memory. Change only the fields that need
   correcting; keep every other field intact — the write tool validates and replaces the full
   record.
5. **Confirm the planned write with the user.** `write_session_descriptor_tool` defaults to
   `overwrite=True`, so it silently clobbers the existing descriptor file with no backup. If
   the user wants the call to refuse-on-existing instead, pass `overwrite=False` explicitly.
6. **Write the corrected payload back to the same path:**
   ```text
   write_session_descriptor_tool(
       file_path="<absolute path>",
       session_type="<type>",
       descriptor_payload=<mutated dict>,
       overwrite=True,  # default; set to False to refuse-on-existing
   )
   ```
   The tool validates against the descriptor dataclass before overwriting, so a malformed edit
   fails without damaging the file.
7. **Re-read to verify:**
   ```text
   read_session_descriptor_tool(file_path="<absolute path>", session_type="<type>")
   ```
8. **Tell the user which copy was amended** and that the change does not propagate to any
   sibling copy (raw session ↔ forged dataset ↔ persistent cache). If other copies should
   match, amend each explicitly.

### Inspecting a descriptor without modification

Same as **Reading a descriptor** above, stopping at step 4.

---

## Amendment propagation

The three on-disk copies are independent — writing to one does not touch any other. Pick the
target(s) that match the durability the user actually wants:

| Scenario                                                                              | Action                                                                                                                                                 |
|---------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| One raw session snapshot has a data-entry error                                       | `write_session_descriptor_tool` on `<session>/raw_data/session_descriptor.yaml`                                                                        |
| A forged dataset copy is wrong (e.g., it was copied from a bad raw snapshot)          | `write_session_descriptor_tool` on `<dataset_root>/<animal>/<session>/session_descriptor.yaml`                                                         |
| Both the raw snapshot and the forged copy are wrong for the same session              | Call `write_session_descriptor_tool` against each file separately. There is no propagation                                                             |
| The per-animal persistent cache is seeding the wrong values into future training runs | Amend `<animal>/persistent_data/<session-type>_descriptor.yaml` directly (exact filename per **Known file locations**, path from `/project-hierarchy`) |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] file_path was resolved via the right owner: /session-discovery or /session-data for raw
      session snapshots, forging:datasets for forged dataset copies,
      /project-hierarchy for per-animal persistent cache, or the user directly for ad-hoc paths
- [ ] session_type was resolved from /session-data (raw snapshot) or supplied directly
      (dataset copy, ad-hoc), and passed to every tool call
- [ ] file_path passed to the tool is absolute
- [ ] describe_session_descriptor_schema_tool was called before any write when the payload
      structure was not already known
- [ ] User confirmed the planned write — including awareness that overwrite defaults to True
- [ ] Payload was passed as descriptor_payload (the correct kwarg name)
- [ ] If refuse-on-existing semantics were required, overwrite=False was passed explicitly
- [ ] If writing: the user was told which single copy was amended and that the change does not
      propagate to sibling copies (raw session ↔ forged dataset ↔ persistent cache)
- [ ] Did not touch SessionData, hardware state, Zaber positions, or mesoscope positions from
      this skill
```

---

## Related skills

| Skill                                   | Relationship                                                                                                             |
|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`         | Run first if the MCP server is not connected                                                                             |
| `/working-directory`                    | Required prerequisite — bootstraps the local working directory the agent uses to resolve project roots                   |
| `/session-data`                         | Sibling — owns `SessionData`, `list_supported_session_types_tool`, and `inspect_sessions_tool` for per-session inventory |
| `/session-discovery`                    | Resolves raw session roots                                                                                               |
| `/session-hardware-state`               | Sibling — owns the per-session hardware-state snapshot                                                                   |
| `/data-assets`                          | Sibling — owns read assets (e.g., subject/surgery records)                                                               |
| `/project-hierarchy`                    | Provides `get_data_root_overview_tool` and per-animal persistent-cache paths                                             |
| `mesoscope:mesoscope-vr-session-schema` | Owns Mesoscope-VR's concrete descriptor field-level schema (classes, field names, types, defaults)                       |
| `mesoscope:mesoscope-vr-snapshots`      | Owns the frozen Zaber and mesoscope-objective position snapshots                                                         |
| `/library-extension`                    | Cross-cutting recipe to add a new `SessionTypes` member and its `DESCRIPTOR_REGISTRY` mapping                            |
| `forging:datasets`                      | Resolves forged dataset per-session `session_descriptor.yaml` paths                                                      |
