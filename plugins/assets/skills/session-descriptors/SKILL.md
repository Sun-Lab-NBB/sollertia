---
name: session-descriptors
description: >-
  Reads, writes, and validates per-session-type session descriptor YAMLs via the sollertia-shared-assets MCP server.
  Descriptors dispatch on SessionTypes alone through DESCRIPTOR_REGISTRY, so the parsing class varies by session type
  while the file-path contract stays generic. Owns write_session_descriptor_tool and
  describe_session_descriptor_schema_tool. Use when repairing, amending, or inspecting a descriptor for any session
  type. For Mesoscope-VR's concrete descriptor schema, see mesoscope:mesoscope-vr-session-schema.
user-invocable: false
---

# Sollertia session descriptors

Reads, writes, and validates session descriptor YAML files via the `slsa mcp` MCP server. This skill is the
**exclusive** owner of `write_session_descriptor_tool` and `describe_session_descriptor_schema_tool`. No other skill in
the marketplace may call these.

Descriptors are treated as **standalone per-session records** keyed on their schema, meaning the descriptor dataclass
that matches a given `SessionTypes` value via `DESCRIPTOR_REGISTRY`. The tools operate on whatever absolute `file_path`
the caller supplies, and they do not care whether that path points at a raw session snapshot, a forged dataset copy, or
an ad-hoc location. The caller is responsible for resolving the path and supplying `session_type` so the right dataclass
is used to parse or validate the file. This skill owns the **generic per-session-type descriptor contract**. The
concrete field-level schema for any one system lives in that system's schema skill (for Mesoscope-VR,
`mesoscope:mesoscope-vr-session-schema`).

---

## Scope

**Covers:**
- Reading any `session_descriptor.yaml` (any supported session type) via `read_session_descriptor_tool`
- Writing or repairing any `session_descriptor.yaml` via `write_session_descriptor_tool` (full-record replacement behind
  a shape-only check, which does not propagate to any sibling copy of the same descriptor)
- Reconstructing a missing or unparseable `session_descriptor.yaml` from the schema, the session's own identity, and
  values the user confirms
- Schema introspection via `describe_session_descriptor_schema_tool`
- The relationship between `SessionTypes` enum values and descriptor classes
- Guidance on where `session_descriptor.yaml` is expected to live (raw session snapshot and forged dataset per-session
  copy) and how callers resolve those paths

**Does not cover:**
- Propagating an amendment from one copy of a descriptor to another. The raw session snapshot and the forged dataset
  copy are independent files on disk, so writing to one does not update any other.
- Partial or per-field updates. The write tool validates and replaces the **full** descriptor payload, so a
  single-field correction goes through the amend workflow below.
- Resolving the canonical path for you. The caller, or a collaborating skill, supplies the absolute `file_path`, and
  this skill only reads and writes.
- Documenting any one system's concrete descriptor field-level schema (descriptor classes, field names, types,
  defaults). For Mesoscope-VR, see `mesoscope:mesoscope-vr-session-schema`. For any session type, call
  `describe_session_descriptor_schema_tool` for that session type.
- Reading the `SessionData` marker file (see `/session-data`)
- Reading or writing the per-session hardware-state snapshot (`hardware_state.yaml`) (see `/session-hardware-state`)
- Reading the frozen `ZaberPositions` and `MesoscopePositions` snapshots (see `mesoscope:mesoscope-vr-snapshots`)
- Reading read assets (surgery metadata today) (see `/data-assets`)
- Discovering sessions (see `/project-hierarchy` for `get_data_root_overview_tool`)
- Resolving the forged per-session descriptor path and auditing dataset structure (see `/datasets`). For which sessions
  a dataset claims in the first place, see `forging:dataset-definition`.

---

## Session types and descriptor classes

The raw session copy and the forged dataset copy both use the filename `session_descriptor.yaml` regardless of session
type. The YAML parses into a session-type-specific descriptor dataclass, and the library's `DESCRIPTOR_REGISTRY` maps
each `SessionTypes` value to its descriptor class. **Only the parsing class varies by session type.** The file-path
contract, the full-record-replacement semantics, and the MCP tool surface are all generic across session types.

**Descriptors dispatch on session type alone.** All three tools take `session_type: str` and nothing system-related.
There is no acquisition-system parameter anywhere on this surface, and no per-system variant of a descriptor class.
`DESCRIPTOR_REGISTRY` is deliberately flat. A session type maps to exactly one descriptor class platform-wide, so an
acquisition system that needs a different descriptor MUST mint a new `SessionTypes` member rather than register a second
class against an existing one (see `/library-extension`). The hardware-state trio works the other way and **is**
system-keyed. It dispatches on `acquisition_system` through `HARDWARE_STATE_REGISTRY`, so two systems there do carry
different classes (see `/session-hardware-state`). Do not carry that mental model across.

Each descriptor captures the **per-session** metadata that varies between sessions of the same type (reward volume
actually delivered, water restriction status, observed behavior summary, experimenter notes). It is **distinct from**
`SessionData`, which is the canonical session marker file.

This skill does not enumerate the concrete descriptor classes or their fields. To learn which descriptor class serves a
given session type and what fields it carries, call `describe_session_descriptor_schema_tool` for that session type (see
**What a descriptor stores and how it is used** below). For Mesoscope-VR's concrete descriptor schema, meaning the
per-session-type descriptor classes and their exact field names, types, and defaults, see
`mesoscope:mesoscope-vr-session-schema`.

To enumerate the canonical `SessionTypes` strings, hand off to `/session-data` for the
`list_supported_session_types_tool` call.

---

## What a descriptor stores and how it is used

The concrete fields a descriptor carries depend on the session type, so **the schema tool is the canonical field
reference rather than this skill**. Call `describe_session_descriptor_schema_tool` for the session type to obtain the
exact field names, types, and defaults. For Mesoscope-VR's concrete descriptor schema, see
`mesoscope:mesoscope-vr-session-schema`.

Regardless of system, every descriptor is a **per-session provenance and bookkeeping record**: who ran the session, how
the animal behaved, how much water it received, and whether the runtime completed cleanly.

`incomplete` is the one descriptor field this skill names unconditionally, because it is contractual rather than
conventional. Every descriptor dataclass registered in `DESCRIPTOR_REGISTRY` MUST declare `incomplete: bool = True`.
`_assert_descriptor_contract()` enforces that at import time and raises `RuntimeError` naming the offending session
types when a registered descriptor omits the field, so a descriptor without it cannot reach the tool surface at all. The
library helper `read_descriptor_incomplete` reads the field to decide whether a session's data is complete and eligible
for unsupervised processing. `incomplete=True` means the session ran past initialization but hit a runtime issue and may
have data gaps, yet still holds real data and should not be purged. This is distinct from the `nk.bin` **uninitialized**
marker described in `/session-data`. A present `nk.bin` means the runtime never finished initializing the session at
all, so there is no data of value and the session is a purge target. The two signals are orthogonal and are surfaced as
independent keys (`uninitialized` and `incomplete`) by `get_data_root_overview_tool` and `inspect_sessions_tool`.

### Lifecycle (high level)

The acquisition runtime is the only authorized **primary** writer of the raw-session copy:

1. At session start, it builds a precursor descriptor seeded with the known starting metadata for that session and
   system. Some systems seed session-type-specific values (such as training thresholds) from the previous session's
   descriptor cached at the per-animal `persistent_data/` slot (see `/project-hierarchy`), so those values carry across
   days without manual re-entry.
2. At session end, the runtime records the runtime-collected fields and flips the data-quality signal (`incomplete`) to
   its clean-end value.
3. The runtime then blocks shutdown until the experimenter replaces the default placeholder text with real content.
   Descriptors commonly carry a placeholder notes field. See the system's schema skill for the exact field name.
4. After verification, the descriptor is copied to the per-animal persistent cache, which becomes the seed for the next
   session of the same type.

For the concrete per-system field set that participates in this lifecycle, meaning which fields are seeded and which are
runtime-recorded, call `describe_session_descriptor_schema_tool`. For Mesoscope-VR specifically, see
`mesoscope:mesoscope-vr-session-schema`.

The forging pipeline later writes a **secondary, independent** copy into the forged dataset hierarchy next to each
session's `data.feather` (see **Known file locations** below). That copy is frozen at the moment the dataset was
assembled and is read by downstream analysis code that consumes forged datasets without reaching back into the raw
session.

`write_session_descriptor_tool` (this skill) is for **post-acquisition repair or amendment** of the copy at whatever
path the caller supplies. The acquisition runtime never goes through slsa MCP, and neither does forging's initial copy
step.

### Post-creation consumers

After a descriptor exists on disk, downstream code reads it for several generic purposes. The specific fields each
consumer routes are system-dependent, so defer to the consumer's owning skill, and to the system's schema skill, for
exact field names:

- **Cross-session continuity.** The next session's precursor reads the per-animal persistent-cache copy to inherit
  carried-over values from the previous session of the same type.
- **Per-animal record logging.** The acquisition system's preprocessing step reads session-summary fields from the raw
  session copy and writes them to the relevant per-animal log.
- **Project manifest assembly.** The forging library reads the descriptor's status fields from the raw session copy for
  the project-level manifest.
- **Dataset eligibility and inclusion.** The forging library's dataset assembly verifies the descriptor exists in the
  raw session and copies it next to `data.feather` in the forged dataset, so downstream analysis has experimenter
  context without touching raw data.

For exact paths, fields, and call sites, defer to the owning skills, which are `experiment:data-management` for
per-animal record logging, `forging:project-state` for the project-level manifest, `forging:dataset-forging` for the
assembly-time copy, and the system's schema skill (`mesoscope:mesoscope-vr-session-schema` for Mesoscope-VR).
Descriptors themselves are primarily a **provenance and bookkeeping record**, capturing what happened, who ran it, and
how much water the animal got, rather than an input to numerical processing.

### Known file locations

The descriptor is written by different pipelines into several canonical locations. The raw session snapshot and the
forged dataset copy both carry the filename `session_descriptor.yaml` (`RawDataFiles.SESSION_DESCRIPTOR`), while the
per-animal persistent cache names its file after the session type. All of them hold the same schema and are read and
written by the same tools, and this skill does not distinguish between them beyond helping the caller resolve the right
path.

| Location                                                    | Populated by                                                                   | Discovery path                                                                                                            |
|-------------------------------------------------------------|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| `<session>/raw_data/session_descriptor.yaml`                | Acquisition runtime at session end (primary on-disk copy)                      | Session root from `/session-discovery`. `inspect_sessions_tool` (`/session-data`) confirms presence via `required_assets` |
| `<dataset_root>/<animal>/<session>/session_descriptor.yaml` | Forging pipeline (copy alongside `data.feather` at dataset assembly)           | `/datasets`, whose `inspect_datasets_tool` returns the absolute path directly                                             |
| `<animal>/persistent_data/<session-type>_descriptor.yaml`   | Acquisition runtime (per-animal cache used to seed the next same-type session) | `/project-hierarchy`                                                                                                      |

Other locations are possible, because the tools take any absolute path, but the three above are the ones populated
automatically. The dataset root is the directory that holds the dataset's `dataset.yaml` marker, which `DatasetData`
re-derives as `dataset_data_path.parent` on every load. The forged copy lives **directly under the session directory
inside the dataset root**, not under a `raw_data/` subdirectory, so the path shape does **not** match
`<session>/raw_data/...` for forged copies. `inspect_datasets_tool` (`/datasets`) returns that absolute path for every
forged session as an artifact entry named `session_descriptor.yaml` and marked `required: true`, so resolving a forged
copy needs no cross-server hop into the forging plugin.

The persistent cache sits at `<root>/<project>/<animal>/persistent_data/` (`AnimalData.persistent_data_path`, owned by
`/project-hierarchy`) and holds one descriptor per session type, so its filename encodes the session type. In
`<session-type>_descriptor.yaml` the placeholder stands for a snake_case name the acquisition system hardcodes, because
`SessionTypes` values carry spaces. The library defines no constant for these names. Substitute the exact filename the
system uses. For the acquisition system's concrete roster, see that system's schema skill
(`mesoscope:mesoscope-vr-session-schema` for Mesoscope-VR).

All copies are **snapshots** frozen at the moment their pipeline wrote them, and none is a live view. Copies are not
strictly immutable, because `write_session_descriptor_tool` can amend any one file in place. That amendment is local to
the one file whose path was passed and does not flow back to any sibling copy (raw session, forged dataset, persistent
cache). For a correction that should apply everywhere, amend each file explicitly.

### Presence, absence, and session status

`session_descriptor.yaml` is a **required** raw asset for **every** session type, with no exceptions. That has two
consequences for how you check whether one exists:

- `inspect_sessions_tool` (`/session-data`) reports the descriptor **twice** per session. It appears in `raw_data_files`
  under the dataclass field name `session_descriptor_path`, and again in `required_assets` under the filename
  `session_descriptor.yaml`. You MUST read the presence check off `required_assets` (its `present` flag) or off
  `issues`, never off `raw_data_files`, which lists the path whether or not anything is on disk. An absent descriptor
  adds the `issues` string `Missing required session_descriptor.yaml at <path>`.
- A missing or unparseable descriptor collapses the session's `status` to `"error"` in both status-reporting tools
  (`get_data_root_overview_tool` and `inspect_sessions_tool`), sets `incomplete` to `null`, and puts the reason in
  `error_detail`. That collapsed status is the usual reason this skill gets invoked, and the repair path for it is
  **Reconstructing an unreadable descriptor** under **Workflows**.

---

## MCP tool surface

| Tool                                      | Purpose                                                                                                    |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `read_session_descriptor_tool`            | Loads a descriptor file at an explicit `file_path`, parsing it with the class for the given `session_type` |
| `write_session_descriptor_tool`           | Writes a validated full descriptor payload to a `file_path` (exclusive). Defaults to `overwrite=True`      |
| `describe_session_descriptor_schema_tool` | Returns the field schema for the descriptor dataclass of a given session type (exclusive)                  |

All three tools take an explicit `session_type: str` and nothing system-related, so class selection is the caller's
responsibility. `read_session_descriptor_tool` and `write_session_descriptor_tool` additionally take `file_path`, and
path resolution is also the caller's responsibility. See **Known file locations** above for the canonical paths and
which neighboring skill owns each.

Path-resolution hand-offs:
- Raw session snapshot: `/project-hierarchy` plus `/session-discovery` for session roots, then `/session-data`
  (`inspect_sessions_tool`) to confirm the file is present.
- Forged dataset per-session copy: `/datasets`, whose `inspect_datasets_tool` returns the absolute
  `session_descriptor.yaml` path for each forged session.
- Per-animal persistent cache: `/project-hierarchy`.
- Ad-hoc location: the user supplies the path directly.

If you do not know the session type for a given file and it is a raw session snapshot, hand off to `/session-data`. Call
`inspect_sessions_tool` on the session root and read `identity.session_type` from the report, or read the marker
directly with `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For forged dataset copies or
ad-hoc paths where no sibling `session_data.yaml` is reachable, the caller MUST supply `session_type` directly.
`/datasets` can surface the session type stored in the dataset marker.

### What the write actually checks

`write_session_descriptor_tool` also accepts `descriptor_payload: dict[str, Any]` (the full record) and a keyword-only
`overwrite: bool = True`. What that write validates is the plugin-wide contract in the `## Response contract` section of
`/assets-mcp-environment-setup`. Read it there. Two consequences are specific to descriptors:

- **No value checking runs at all.** None of the registered descriptor classes defines `__post_init__`, so the write
  reduces to a shape check. An out-of-range number or a nonsensical string is persisted exactly as supplied.
- **Nearly every field carries a default.** Descriptors declare one for all but a handful of the fields they hold, so a
  partial payload passes the write and quietly resets everything it did not mention. Read the `required` marker off
  `describe_session_descriptor_schema_tool` (see **Schema payload shape** under the `## Response contract` section of
  `/assets-mcp-environment-setup`) to learn which fields have no default, and never treat the absence of a rejection as
  evidence the payload was complete.

There is no partial-update tool on this surface.

You MUST confirm the destination path exists before writing to a path from which you did not just read, either from a
prior successful read or from `inspect_sessions_tool` (`/session-data`). A read needs no such confirmation, because
`read_session_descriptor_tool` against a path that does not exist fails with
`Unable to read <Class> from <path>: the file does not exist.`

`describe_session_descriptor_schema_tool` returns two keys: `session_type` (the caller's input string, echoed back) and
`schema` (the field schema of the session type's descriptor dataclass).

### Response contract

Every response follows the plugin-wide envelope documented in the `## Response contract` section of
`/assets-mcp-environment-setup`. Read that section first, then use the per-tool keys below.

| Tool                                      | Success keys beyond `success`                           |
|-------------------------------------------|---------------------------------------------------------|
| `read_session_descriptor_tool`            | `file_path`, `data`, `descriptor_class`, `session_type` |
| `write_session_descriptor_tool`           | `file_path`, `data`, `descriptor_class`, `session_type` |
| `describe_session_descriptor_schema_tool` | `session_type`, `schema`                                |

On read and write, `descriptor_class` (the resolved class `__name__`) and `session_type` (the caller's input string,
echoed back) are attached **only when the call succeeded**. A failed call returns the bare error envelope, so you MUST
NOT read `descriptor_class` to learn which class was attempted after a failure. The describe tool returns no
`descriptor_class` key at all.

Three error messages cover nearly every failure. The first is emitted by all three tools:

```text
Unable to resolve the descriptor class. The session_type '<value>' is not a member of SessionTypes.
Valid values: lick training, run training, mesoscope experiment, window checking.
```

```text
Unable to write <Class> to <path>: a file already exists at this path. Pass overwrite=True to replace it.
```

```text
Unable to validate the payload as <Class>: <exception>
```

The first message is one line at runtime and is wrapped here for width. Note that the valid `SessionTypes` values are
**space-separated strings**, not snake_case, so `"lick_training"` fails with that message while `"lick training"`
resolves.

---

## Workflows

All workflows reduce to: **resolve the path, pick the session type, then read or write**. The resolution step changes
depending on where the file lives, and the read and write step is the same everywhere. One workflow starts without a
readable file: **Reconstructing an unreadable descriptor** rebuilds the record from the schema rather than from a read.

### Reading a descriptor (generic)

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`), and the target descriptor file
   exists at the path you are about to pass.
2. **Resolve the `file_path`** using the hand-off that matches the container:
   - **Raw session snapshot**: session root from `/project-hierarchy` or `/session-discovery`, optionally confirming the
     file is present via `inspect_sessions_tool` (`/session-data`). The path is
     `<session>/raw_data/session_descriptor.yaml`.
   - **Forged dataset copy**: hand off to `/datasets` and read the `session_descriptor.yaml` artifact entry that
     `inspect_datasets_tool` returns for the session. Path shape is
     `<dataset_root>/<animal>/<session>/session_descriptor.yaml`.
   - **Per-animal persistent cache**: hand off to `/project-hierarchy`. The path shape is
     `<animal>/persistent_data/<session-type>_descriptor.yaml`, with the system's hardcoded snake_case filename in place
     of the placeholder (see **Known file locations**).
   - **Ad-hoc location**: the user supplies the path directly.
3. **Determine `session_type`.**
   - If the file sits in a raw session snapshot, hand off to `/session-data`. Either call
     `inspect_sessions_tool(session_paths=["<session root>"])` and read `identity.session_type` from the report, or read
     the marker directly with `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`.
   - Otherwise, the caller supplies `session_type` directly (e.g., from a dataset marker or from the user).
4. **Read the descriptor:**
   ```text
   read_session_descriptor_tool(
       file_path="<absolute path to the descriptor file>",
       session_type="<type>",
   )
   ```
5. **Project or report the fields the user requested** from `response["data"]`. If the read came from a snapshot, remind
   the user the values reflect the state at the moment that copy was written, and do not live-track any sibling copy.

### Amending a descriptor in place

Use this when a single copy has an error and a durable correction to every other copy is either not warranted or will be
handled separately. The amendment affects only the one file whose path is passed.

1. **Resolve the `file_path` and `session_type`** as in the read workflow.
2. **Read the current record** so you mutate a validated baseline rather than constructing one from scratch:
   ```text
   read_session_descriptor_tool(file_path="<absolute path>", session_type="<type>")
   ```
3. **Inspect the schema** if you need to add fields or double-check types:
   ```text
   describe_session_descriptor_schema_tool(session_type="<type>")
   ```
4. **Mutate the returned `response["data"]` dict** in memory. Change only the fields that need correcting, and carry
   every other field through untouched. The read in step 2 supplies a complete baseline here, so within this workflow
   you MUST NOT hand-build a payload from a subset of the fields. When the file cannot be read at all, use
   **Reconstructing an unreadable descriptor** below instead.
5. **Confirm the planned write with the user.** `write_session_descriptor_tool` defaults to `overwrite=True`, so it
   silently clobbers the existing descriptor file with no backup. If the user wants the call to refuse-on-existing
   instead, pass `overwrite=False` explicitly.
6. **Write the corrected payload back to the same path the read returned.** Confirm the destination is the path the
   read in step 2 returned before you call:
   ```text
   write_session_descriptor_tool(
       file_path="<absolute path>",
       session_type="<type>",
       descriptor_payload=<mutated dict>,
       overwrite=True,  # the default, set to False to refuse-on-existing
   )
   ```
   Descriptors define no `__post_init__`, so this is a shape check only, and a wrong-typed or out-of-range value is
   written exactly as supplied.
7. **Re-read and diff to verify:**
   ```text
   read_session_descriptor_tool(file_path="<absolute path>", session_type="<type>")
   ```
   Compare every key in the returned `data` against the payload you intended. This is the only way to catch a field that
   was silently reset to its default, and you MUST do it before reporting success.
8. **Tell the user which copy was amended** and that the change does not propagate to any sibling copy (raw session,
   forged dataset, persistent cache). If other copies should match, amend each explicitly.

### Reconstructing an unreadable descriptor

Use this when the descriptor is missing or fails to parse, which is the condition that collapses the session's `status`
to `"error"` and sets `incomplete` to `null`. The amend workflow cannot start here, because there is no record to read
and mutate. Treat the result as a repair of a historical record rather than an authoring pass: every value you put in
the payload is a claim about a session that already ran.

1. **Resolve the `file_path`** as in the read workflow, and confirm from the container's owner that the file is the one
   that is missing or unparseable (`inspect_sessions_tool` via `/session-data` for a raw session snapshot,
   `inspect_datasets_tool` via `/datasets` for a forged copy).
2. **Determine `session_type` from outside the descriptor.** Read `identity.session_type` off `inspect_sessions_tool`,
   or read the marker with `read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")` (`/session-data`).
   For a forged copy or an ad-hoc path, the caller supplies it.
3. **Recover the field set** with `describe_session_descriptor_schema_tool(session_type="<type>")`, and read `required`
   off each field to learn which ones the schema leaves without a default:
   ```text
   describe_session_descriptor_schema_tool(session_type="<type>")
   ```
4. **Recover the session identity** from `inspect_sessions_tool` (`/session-data`), which reports the animal, the
   session name, and the session type for the session that owns the file.
5. **Recover whatever the raw file still yields.** An unparseable YAML is often readable as text, so salvage the
   key-value pairs that survived and treat the rest as unknown.
6. **Confirm every reconstructed value with the user before it is written**, field by field, including the values you
   salvaged from the file text. A value the user cannot confirm goes in at the schema default, and you MUST tell the
   user which fields were defaulted rather than recovered. A field the schema marks `required` carries no default to
   fall back on, so its value MUST come from the user.
7. **Write the complete record**, carrying every field the schema names:
   ```text
   write_session_descriptor_tool(
       file_path="<absolute path>",
       session_type="<type>",
       descriptor_payload=<reconstructed dict>,
       overwrite=True,  # the default, set to False to refuse-on-existing
   )
   ```
8. **Re-read and diff to verify:**
   ```text
   read_session_descriptor_tool(file_path="<absolute path>", session_type="<type>")
   ```
   Compare every key in the returned `data` against the payload you intended before reporting success.
9. **Re-check the session status** with `inspect_sessions_tool` (`/session-data`) to confirm the descriptor now parses
   and the session no longer reports `status: "error"`.
10. **Tell the user which copy was reconstructed**, which fields were defaulted, and that sibling copies (raw session,
    forged dataset, persistent cache) are untouched.

### Inspecting a descriptor without modification

Same as **Reading a descriptor** above, stopping at step 4.

---

## Amendment propagation

The three on-disk copies are independent, so writing to one does not touch any other. Pick the target or targets that
match the durability the user actually wants:

| Scenario                                                                              | Action                                                                                                                                                 |
|---------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| One raw session snapshot has a data-entry error                                       | `write_session_descriptor_tool` on `<session>/raw_data/session_descriptor.yaml`                                                                        |
| A forged dataset copy is wrong (e.g., it was copied from a bad raw snapshot)          | `write_session_descriptor_tool` on `<dataset_root>/<animal>/<session>/session_descriptor.yaml`                                                         |
| Both the raw snapshot and the forged copy are wrong for the same session              | Call `write_session_descriptor_tool` against each file separately. There is no propagation                                                             |
| The per-animal persistent cache is seeding the wrong values into future training runs | Amend `<animal>/persistent_data/<session-type>_descriptor.yaml` directly (exact filename per **Known file locations**, path from `/project-hierarchy`) |

---

## Related skills

| Skill                                   | Relationship                                                                      |
|-----------------------------------------|-----------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`         | Run first if the MCP server is not connected. Owns the response contract          |
| `/working-directory`                    | Required prerequisite that bootstraps the local working directory                 |
| `/session-data`                         | Owns `SessionData`, `inspect_sessions_tool`, and the session-type list            |
| `/session-discovery`                    | Resolves raw session roots                                                        |
| `/session-hardware-state`               | Sibling that owns the hardware-state snapshot, keyed on acquisition system        |
| `/data-assets`                          | Sibling that owns read assets (surgery metadata today)                            |
| `/project-hierarchy`                    | Owns the project hierarchy and the generic `AnimalData.persistent_data_path` slot |
| `/datasets`                             | Owns `inspect_datasets_tool`, which returns the forged descriptor path            |
| `/library-extension`                    | Cross-cutting recipe to add a `SessionTypes` member and its registry mapping      |
| `mesoscope:mesoscope-vr-session-schema` | Owns Mesoscope-VR's concrete descriptor schema and cache filenames                |
| `mesoscope:mesoscope-vr-snapshots`      | Owns the frozen Zaber and mesoscope-objective position snapshots                  |
| `experiment:acquisition-system-runtime` | Writes the descriptor at session end, whose repair and amendment this skill owns  |
| `experiment:data-management`            | Owns the acquisition-side per-animal record logging of descriptor fields          |
| `forging:project-state`                 | Owns the project-level manifest that reads descriptor status fields               |
| `forging:dataset-forging`               | Owns the assembly step that copies the descriptor into a forged dataset           |
| `forging:dataset-definition`            | Owns dataset composition semantics (which sessions a dataset claims)              |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] file_path was resolved via the right owner: /session-discovery or /session-data for raw
      session snapshots, /datasets for forged dataset copies, /project-hierarchy for the
      per-animal persistent cache, or the user directly for ad-hoc paths
- [ ] session_type was resolved from /session-data (raw snapshot) or supplied directly
      (dataset copy, ad-hoc), and passed to every tool call as a space-separated SessionTypes value
- [ ] file_path passed to the tool is absolute
- [ ] describe_session_descriptor_schema_tool was called before any write when the payload
      structure was not already known
- [ ] User confirmed the planned write, including awareness that overwrite defaults to True
- [ ] Payload was passed as descriptor_payload (the correct kwarg name)
- [ ] Every field the read or the schema returned was carried into the write payload
- [ ] The destination path was confirmed to exist before writing (prior successful read or
      inspect_sessions_tool)
- [ ] If the descriptor was missing or unparseable: session_type came from the session marker
      rather than from the descriptor, every reconstructed value was confirmed with the user
      before the write, and the defaulted fields were reported afterwards
- [ ] After a reconstruction, inspect_sessions_tool was re-run to confirm the session no longer
      reports status "error"
- [ ] The write was verified by re-reading and diffing the result against the intended payload
- [ ] If refuse-on-existing semantics were required, overwrite=False was passed explicitly
- [ ] If writing: the user was told which single copy was amended and that the change does not
      propagate to sibling copies (raw session, forged dataset, persistent cache)
- [ ] Did not touch SessionData, hardware state, Zaber positions, or mesoscope positions from
      this skill
```
