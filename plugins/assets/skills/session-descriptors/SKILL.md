---
name: session-descriptors
description: >-
  Reads, writes, and validates per-session descriptor YAML files (LickTrainingDescriptor,
  RunTrainingDescriptor, WindowCheckingDescriptor, MesoscopeExperimentDescriptor) via the slsa
  MCP server. Owns the session descriptor write tools and schema introspection. Use when repairing,
  amending, or inspecting a session descriptor for any of the four supported session types.
user-invocable: true
---

# Sollertia session descriptors

Reads, writes, and validates the per-session descriptor YAML files that live inside every Sollertia
session directory. This skill is the **exclusive** owner of `write_session_descriptor_tool` and
`describe_session_descriptor_schema_tool` — no other skill in the marketplace may call these.

---

## Scope

**Covers:**
- Reading session descriptors of all four supported session types
- Writing (repairing or amending) session descriptors
- Schema introspection via `describe_session_descriptor_schema_tool`
- The relationship between `SessionTypes` enum values and descriptor classes

**Does not cover:**
- Reading the `SessionData` marker file (see `/session-data`)
- Reading or writing the per-session `MesoscopeHardwareState` snapshot (see
  `/session-hardware-state`)
- Reading the frozen `ZaberPositions` and `MesoscopePositions` snapshots (see the experiment
  plugin's `/session-snapshots`)
- Reading subject metadata (see `/subject-metadata`)
- Discovering sessions (see `/project-hierarchy` for `discover_sessions_tool`)

---

## Session types and descriptor classes

| `SessionTypes` value   | Descriptor file                        | Descriptor dataclass            |
|------------------------|----------------------------------------|---------------------------------|
| `lick training`        | `lick_training_descriptor.yaml`        | `LickTrainingDescriptor`        |
| `run training`         | `run_training_descriptor.yaml`         | `RunTrainingDescriptor`         |
| `window checking`      | `window_checking_descriptor.yaml`      | `WindowCheckingDescriptor`      |
| `mesoscope experiment` | `experiment_descriptor.yaml`           | `MesoscopeExperimentDescriptor` |

Each descriptor captures the **per-session** metadata that varies between sessions of the same type
(reward volume actually delivered, water restriction status, observed behavior summary, experimenter
notes). It is **distinct from** `SessionData`, which is the canonical session marker file.

To enumerate the canonical `SessionTypes` strings, hand off to `/session-data` for the
`list_supported_session_types_tool` call.

---

## What a descriptor stores and how it is used

The summary below is loose, intended to orient an agent before it reads or amends a
descriptor. For exact field names, types, and defaults call
`describe_session_descriptor_schema_tool` or read the dataclass source — the schema is the
canonical reference, this section is not.

### Field categories

All four descriptors carry the same four **common fields**:

- **`experimenter`** — the supervising experimenter's ID.
- **`animal_weight_g`** — the animal's weight at the start of the session.
- **`incomplete`** — flips to `False` on a successful acquisition. Distinct from the `nk.bin`
  initialization marker described in `/session-data`; this flag persists as the durable
  completeness record.
- **`experimenter_notes`** — runtime notes. The acquisition runtime requires the default
  placeholder text be replaced before a session is signed off, so a non-default value is the
  expected steady state.

Beyond the common four, each descriptor adds session-type-specific data:

| Descriptor                      | Categories of additional data                                                                                                                 |
|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| `LickTrainingDescriptor`        | Lick-training reward schedule, training-time and water caps, runtime-recorded water totals, unconsumed-reward limit                           |
| `RunTrainingDescriptor`         | Run-training thresholds (speed and duration with per-step increments and idle tolerance), reward schedule and water caps, water totals, limit |
| `MesoscopeExperimentDescriptor` | Runtime-recorded water totals and unconsumed-reward limit (reward schedule and zone params live on the experiment configuration)              |
| `WindowCheckingDescriptor`      | Just `surgery_quality` (integer 0–3 grading the cranial-window surgery)                                                                       |

The three non-window-checking descriptors all carry the same set of runtime-recorded water
totals, distinguishing water delivered during active runtime, water dispensed while paused,
any post-session top-up the experimenter administered manually, and the target volume the
animal should have received. Inspect the schema for exact field names.

### Lifecycle (high level)

The acquisition runtime is the only authorized writer:

1. At session start, it builds a precursor descriptor with the experimenter ID and starting
   weight. For training sessions, training thresholds are seeded from the previous session's
   descriptor cached at the per-animal `persistent_data/` slot (see `/project-hierarchy`), so
   trained values carry across days without manual re-entry.
2. At session end, the runtime records water totals and flips `incomplete` to `False`.
3. The runtime then blocks shutdown until the experimenter replaces the default
   `experimenter_notes` placeholder with real notes.
4. After verification, the descriptor is copied to the per-animal persistent cache, which
   becomes the seed for the next session of the same type.

`write_session_descriptor_tool` (this skill) is for **post-acquisition repair or amendment** by
agents, not for primary creation. The acquisition runtime never goes through slsa MCP.

### Post-creation consumers

After the descriptor exists on disk, downstream code reads it for:

- **Cross-session continuity** — the next session's precursor reads the persistent-cache copy
  to inherit training thresholds from the previous session of the same type.
- **Water-restriction logging** — the experiment library's preprocessing step reads water
  totals, weight, and experimenter ID and writes to the per-animal water-restriction log; for
  window-checking sessions, reads `surgery_quality` and writes to the surgery log instead.
- **Project manifest assembly** — the forging library reads `incomplete` and
  `experimenter_notes` for the project-level manifest.
- **Dataset eligibility** — the forging library's dataset assembly verifies the experiment
  descriptor exists and parses cleanly before admitting a mesoscope-experiment session into a
  forged dataset.

For exact paths and call sites, defer to the owning skills (experiment plugin's data
management, forging plugin's processing and forging skills). Descriptors themselves are
primarily a **provenance and bookkeeping record** — what happened, who ran it, how much water
the animal got — not an input to numerical processing.

---

## MCP tool surface

| Tool                                      | Purpose                                                                                                   |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| `read_session_descriptor_tool`            | Reads the descriptor file for a session                                                                   |
| `write_session_descriptor_tool`           | Writes (or repairs) a descriptor file (exclusive to this skill). Defaults to `overwrite=True` — see below |
| `describe_session_descriptor_schema_tool` | Returns the canonical descriptor filename and field schema for a given session type                       |

All `session_path` arguments accept **either the session root directory or its `raw_data/`
subdirectory** — the resolver normalizes both forms to the canonical session root before any
tool runs.

The discovery side of "which descriptors exist in a session directory" is owned by `/session-data`
(`discover_session_descriptors_tool`). Call it as a natural share when you need to confirm a descriptor
file exists before reading or writing it.

`describe_session_descriptor_schema_tool` returns `session_type`, `descriptor_filename` (the
canonical YAML filename for that session type — same value as the table above), and `schema`
(the dataclass field schema). The `descriptor_filename` is useful when the caller wants to
construct a path under `raw_data/` without re-consulting this skill's mapping table.

---

## Workflows

### Repairing a corrupted or stale descriptor

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`).
2. **Read the existing descriptor:**
   ```text
   read_session_descriptor_tool(session_path="<absolute>")
   ```
3. **Inspect the schema for the session type:**
   ```text
   describe_session_descriptor_schema_tool(session_type="<type>")
   ```
4. **Build the corrected descriptor dictionary** based on the schema and the values you can recover.
5. **Confirm the planned write with the user.** `write_session_descriptor_tool` defaults to
   `overwrite=True`, so it silently clobbers the existing descriptor file with no backup. If
   the user wants the call to refuse-on-existing instead, pass `overwrite=False` explicitly in
   Step 6.
6. **Write the corrected descriptor:**
   ```text
   write_session_descriptor_tool(
       session_path="<absolute>",
       descriptor_payload={ ... },
       overwrite=True,   # default; set to False to refuse-on-existing
   )
   ```
   The kwargs are `session_path`, `descriptor_payload`, and the keyword-only `overwrite` (default
   `True`). The destination filename is determined automatically from the session's
   `session_type` (loaded from `session_data.yaml`) and the file is written into
   `<session>/raw_data/`.
7. **Re-read to verify:**
   ```text
   read_session_descriptor_tool(session_path="<absolute>")
   ```

### Amending a descriptor with post-hoc data

Use the same workflow above but mutate only the fields the user wants to amend (e.g., adding the actual
water volume delivered after a manual recount). Always confirm with the user before writing.

### Inspecting a descriptor without modification

```text
read_session_descriptor_tool(session_path="<absolute>")
```

If you need to know which descriptor file is present (when the session type is unknown), hand off to
`/session-data` to call `discover_session_descriptors_tool` first.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] describe_session_descriptor_schema_tool was called before any write
- [ ] User confirmed the planned write — including awareness that overwrite defaults to True
- [ ] Payload was passed as descriptor_payload (the correct kwarg name)
- [ ] If refuse-on-existing semantics were required, overwrite=False was passed explicitly
- [ ] write_session_descriptor_tool succeeded without schema errors
- [ ] read_session_descriptor_tool returned the expected content after the write
- [ ] Did not touch SessionData, hardware state, Zaber positions, or mesoscope positions from this skill
```

---

## Related skills

| Skill                                  | Relationship                                                         |
|----------------------------------------|----------------------------------------------------------------------|
| `/assets-mcp-environment-setup`        | Run first if the MCP server is not connected                         |
| `/session-data`                        | Sibling — owns `SessionData` and `list_supported_session_types_tool` |
| `/session-hardware-state`              | Sibling — owns the per-session `MesoscopeHardwareState` snapshot     |
| experiment plugin `/session-snapshots` | Owns the frozen Zaber and mesoscope-objective position snapshots     |
| `/subject-metadata`                    | Sibling — owns subject records                                       |
| `/project-hierarchy`                   | Provides `discover_sessions_tool` to locate sessions                 |
