---
name: session-descriptors
description: >-
  Reads, writes, and validates session descriptor YAMLs (LickTraining, RunTraining,
  WindowChecking, MesoscopeExperiment descriptors) via the sollertia-shared-assets MCP server.
  Owns write_session_descriptor_tool and describe_session_descriptor_schema_tool; tools are
  file-path based, accepting raw session snapshots or forged dataset copies. Use when
  repairing, amending, or inspecting a descriptor for any of the four session types.
user-invocable: false
---

# Sollertia session descriptors

Reads, writes, and validates session descriptor YAML files via the `slsa mcp` MCP server.
This skill is the **exclusive** owner of `write_session_descriptor_tool` and
`describe_session_descriptor_schema_tool` — no other skill in the marketplace may call these.

Descriptors are treated as **standalone per-session records** keyed on their schema (the
descriptor dataclass that matches a given `SessionTypes` value). The tools operate on whatever
absolute `file_path` the caller supplies; they do not care whether that path points at a raw
session snapshot, a forged dataset copy, or an ad-hoc location. The caller is responsible for
resolving the path and supplying `session_type` so the right dataclass is used to parse or
validate the file.

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
- Reading the `SessionData` marker file (see `/session-data`)
- Reading or writing the per-session `MesoscopeHardwareState` snapshot (see
  `/session-hardware-state`)
- Reading the frozen `ZaberPositions` and `MesoscopePositions` snapshots (see the experiment
  plugin's `/mesoscope-vr-snapshots`)
- Reading subject metadata (see `/data-assets`)
- Discovering sessions (see `/project-hierarchy` for `get_data_root_overview_tool`)
- Dataset assembly and dataset-level descriptor path resolution (see the forging plugin's
  `/datasets` skill)

---

## Session types and descriptor classes

Every descriptor file uses the same filename (`session_descriptor.yaml`) regardless of session
type. The YAML parses into a session-type-specific descriptor dataclass:

| `SessionTypes` value   | Descriptor dataclass            |
|------------------------|---------------------------------|
| `lick training`        | `LickTrainingDescriptor`        |
| `run training`         | `RunTrainingDescriptor`         |
| `window checking`      | `WindowCheckingDescriptor`      |
| `mesoscope experiment` | `MesoscopeExperimentDescriptor` |

The library's `DESCRIPTOR_REGISTRY` maps `SessionTypes` → descriptor class; only the parsing
class varies by session type. Each descriptor captures the **per-session** metadata that varies
between sessions of the same type (reward volume actually delivered, water restriction status,
observed behavior summary, experimenter notes). It is **distinct from** `SessionData`, which is
the canonical session marker file.

To enumerate the canonical `SessionTypes` strings, hand off to `/session-data` for the
`list_supported_session_types_tool` call.

---

## What a descriptor stores and how it is used

The summary below is loose, intended to orient an agent before it reads or amends a
descriptor. For exact field names, types, and defaults call
`describe_session_descriptor_schema_tool` or read the dataclass source — the schema is the
canonical reference, this section is not.

### Field categories

All four descriptors share only **three** common fields:

- **`experimenter`** — the supervising experimenter's ID.
- **`incomplete`** — defaults to `True` and flips to `False` on a clean session end. This is
  the durable **data-quality** signal: `incomplete=True` means the session ran past
  initialization but hit a runtime issue and may have data gaps — the session still holds
  real data and should not be purged. Distinct from the `nk.bin` **uninitialized** marker
  described in `/session-data`: `nk.bin` presence means the runtime never finished
  initializing the session at all, so there is no data of value and the session is a purge
  target. The two signals are orthogonal and are surfaced as independent keys (`uninitialized`
  and `incomplete`) by every status-reporting MCP tool.
- **`experimenter_notes`** — runtime notes. The acquisition runtime requires the default
  placeholder text be replaced before a session is signed off, so a non-default value is the
  expected steady state.

The three **non-window-checking** descriptors (`LickTrainingDescriptor`,
`RunTrainingDescriptor`, `MesoscopeExperimentDescriptor`) additionally share:

- **`animal_weight_g`** — the animal's weight at the start of the session.
- **`maximum_unconsumed_rewards`** — cap on consecutive unclaimed water rewards before
  delivery is paused.
- **Runtime-recorded water totals** — three float fields that distinguish water delivered
  during active runtime, water dispensed while paused, and any post-session top-up the
  experimenter administered manually. Inspect the schema for exact field names.

`WindowCheckingDescriptor` does **not** carry `animal_weight_g`, `maximum_unconsumed_rewards`,
or water totals — its only session-type-specific field is `surgery_quality`.

Beyond the shared fields above, each descriptor adds session-type-specific data:

| Descriptor                      | Additional data                                                                                                                   |
|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| `LickTrainingDescriptor`        | Lick-training reward schedule (reward size, tone duration, min/max reward delay) and training-time / water caps                   |
| `RunTrainingDescriptor`         | Run-training thresholds (initial and final speed + duration, per-step increments, idle tolerance), reward schedule and water caps |
| `MesoscopeExperimentDescriptor` | No additional fields beyond the non-window-checking shared set (reward schedule and zone params live on the experiment config)    |
| `WindowCheckingDescriptor`      | `surgery_quality` (integer 0–3 grading the cranial-window surgery)                                                                |

### Lifecycle (high level)

The acquisition runtime is the only authorized **primary** writer of the raw-session copy:

1. At session start, it builds a precursor descriptor with the experimenter ID and starting
   weight. For training sessions, training thresholds are seeded from the previous session's
   descriptor cached at the per-animal `persistent_data/` slot (see `/project-hierarchy`), so
   trained values carry across days without manual re-entry.
2. At session end, the runtime records water totals and flips `incomplete` to `False`.
3. The runtime then blocks shutdown until the experimenter replaces the default
   `experimenter_notes` placeholder with real notes.
4. After verification, the descriptor is copied to the per-animal persistent cache, which
   becomes the seed for the next session of the same type.

The forging pipeline later writes a **secondary, independent** copy into the forged dataset
hierarchy next to each session's `data.feather` (see **Known file locations** below). That
copy is frozen at the moment the dataset was assembled and is read by downstream analysis code
that consumes forged datasets without reaching back into the raw session.

`write_session_descriptor_tool` (this skill) is for **post-acquisition repair or amendment** of
whichever copy the caller supplies a path to. The acquisition runtime never goes through slsa
MCP, and neither does forging's initial copy step.

### Post-creation consumers

After a descriptor exists on disk, downstream code reads it for:

- **Cross-session continuity** — the next session's precursor reads the per-animal
  persistent-cache copy to inherit training thresholds from the previous session of the same
  type.
- **Water-restriction logging** — the experiment library's preprocessing step reads water
  totals, weight, and experimenter ID from the raw session copy and writes to the per-animal
  water-restriction log; for window-checking sessions, reads `surgery_quality` and writes to
  the surgery log instead.
- **Project manifest assembly** — the forging library reads `incomplete` and
  `experimenter_notes` from the raw session copy for the project-level manifest.
- **Dataset eligibility and inclusion** — the forging library's dataset assembly verifies the
  descriptor exists in the raw session and copies it next to `data.feather` in the forged
  dataset so downstream analysis has experimenter context without touching raw data.

For exact paths and call sites, defer to the owning skills (experiment plugin's data
management, forging plugin's processing and forging skills). Descriptors themselves are
primarily a **provenance and bookkeeping record** — what happened, who ran it, how much water
the animal got — not an input to numerical processing.

### Known file locations

`session_descriptor.yaml` (`RawDataFiles.SESSION_DESCRIPTOR`) is written by different pipelines
into several canonical locations. All of them hold the same schema and are read/written by the
same tools; this skill does not distinguish between them beyond helping the caller resolve the
right path.

| Location                                                    | Populated by                                                                   | Discovery path                                                                                                                        |
|-------------------------------------------------------------|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `<session>/raw_data/session_descriptor.yaml`                | Acquisition runtime at session end (primary on-disk copy)                      | Session root from `/session-discovery`; `inspect_sessions_tool` (`/session-data`) confirms presence in its `raw_data_files` inventory |
| `<project_root>/<animal>/<session>/session_descriptor.yaml` | Forging pipeline (copy alongside `data.feather` at dataset assembly)           | Forging plugin's `/datasets` resolves the forged-session layout under a dataset's `project_root`                                      |
| `<persistent_data>/<animal>/session_descriptor.yaml`        | Acquisition runtime (per-animal cache used to seed the next same-type session) | `/project-hierarchy`                                                                                                                  |

Other locations are possible — the tools take any absolute path — but the three above are the
ones populated automatically. The forged copy lives **directly under the session directory
inside the project root**, not under a `raw_data/` subdirectory; the path shape therefore does
**not** match `<session>/raw_data/...` for forged copies.

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
- Forged dataset per-session copy → the forging plugin's `/datasets` skill, which resolves
  the dataset-hierarchy layout.
- Per-animal persistent cache → `/project-hierarchy`.
- Ad-hoc location → the user supplies the path directly.

If you don't know the session type for a given file and it's a raw session snapshot, hand off
to `/session-data`. Call `inspect_sessions_tool` on the session root and read
`identity.session_type` from the report, or read the marker directly with
`read_session_data_tool(file_path="<session>/raw_data/session_data.yaml")`. For forged dataset
copies (or ad-hoc paths) where no sibling `session_data.yaml` is reachable, the caller must
supply `session_type` directly; the forging plugin's `/datasets` skill can surface the
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

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`); the
   target `session_descriptor.yaml` exists at the path you are about to pass.
2. **Resolve the `file_path`** using the hand-off that matches the container:
   - **Raw session snapshot** → session root from `/project-hierarchy` or
     `/session-discovery`; optionally confirm the file is present via
     `inspect_sessions_tool` (`/session-data`). Path is
     `<session>/raw_data/session_descriptor.yaml`.
   - **Forged dataset copy** → hand off to the forging plugin's `/datasets` skill for the
     per-session path inside the dataset's `project_root`. Path shape is
     `<project_root>/<animal>/<session>/session_descriptor.yaml`.
   - **Per-animal persistent cache** → hand off to `/project-hierarchy`.
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
       file_path="<absolute path to session_descriptor.yaml>",
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

| Scenario                                                                               | Action                                                                                                |
|----------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| One raw session snapshot has a data-entry error                                        | `write_session_descriptor_tool` on `<session>/raw_data/session_descriptor.yaml`                       |
| A forged dataset copy is wrong (e.g., it was copied from a bad raw snapshot)           | `write_session_descriptor_tool` on `<project_root>/<animal>/<session>/session_descriptor.yaml`        |
| Both the raw snapshot and the forged copy are wrong for the same session               | Call `write_session_descriptor_tool` against each file separately — there is no propagation           |
| The per-animal persistent cache is seeding the wrong values into future training runs  | Amend the cache copy directly (path from `/project-hierarchy`)                                        |

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] file_path was resolved via the right owner: /session-discovery or /session-data for raw
      session snapshots, forging plugin's /datasets for forged dataset copies,
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

| Skill                                       | Relationship                                                                                                             |
|---------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`             | Run first if the MCP server is not connected                                                                             |
| `/working-directory`                        | Required prerequisite — bootstraps the local working directory the agent uses to resolve project roots                   |
| `/session-data`                             | Sibling — owns `SessionData`, `list_supported_session_types_tool`, and `inspect_sessions_tool` for per-session inventory |
| `/session-discovery`                        | Resolves raw session roots                                                                                               |
| `/session-hardware-state`                   | Sibling — owns the per-session hardware-state snapshot                                                                   |
| `/data-assets`                              | Sibling — owns read assets (e.g., subject/surgery records)                                                               |
| `/project-hierarchy`                        | Provides `get_data_root_overview_tool` and per-animal persistent-cache paths                                             |
| experiment plugin `/mesoscope-vr-snapshots` | Owns the frozen Zaber and mesoscope-objective position snapshots                                                         |
| `/library-extension`                        | Cross-cutting recipe to add a new `SessionTypes` member; lists the descriptor mapping table here that needs updating     |
| forging plugin `/datasets`                  | Resolves forged dataset per-session `session_descriptor.yaml` paths                                                      |
