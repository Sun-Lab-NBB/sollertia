---
name: session-descriptors
description: >-
  Reads, writes, and validates per-session descriptor YAMLs (LickTraining, RunTraining,
  WindowChecking, MesoscopeExperiment descriptors) via the sollertia-shared-assets MCP server.
  Owns write_session_descriptor_tool and describe_session_descriptor_schema_tool. Use when
  repairing, amending, or inspecting a descriptor for any of the four session types.
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

Every session's descriptor is stored at a single canonical path — `{raw_data}/session_descriptor.yaml`
— regardless of session type. The YAML parses into a session-type-specific descriptor dataclass:

| `SessionTypes` value   | Descriptor dataclass            |
|------------------------|---------------------------------|
| `lick training`        | `LickTrainingDescriptor`        |
| `run training`         | `RunTrainingDescriptor`         |
| `window checking`      | `WindowCheckingDescriptor`      |
| `mesoscope experiment` | `MesoscopeExperimentDescriptor` |

The library's `DESCRIPTOR_REGISTRY` maps `SessionTypes` → descriptor class; the descriptor path
inside any session's `raw_data/` is always `session_descriptor.yaml` and only the parsing class
varies by session type.

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
- **Runtime-recorded water totals** — four float fields that distinguish water delivered
  during active runtime, water dispensed while paused, any post-session top-up the
  experimenter administered manually, and the target volume the animal should have received.
  Inspect the schema for exact field names.

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
| `read_session_descriptor_tool`            | Reads a descriptor file at an explicit path, parsing it with the class for the given `session_type`       |
| `write_session_descriptor_tool`           | Writes (or repairs) a descriptor file (exclusive to this skill). Defaults to `overwrite=True` — see below |
| `describe_session_descriptor_schema_tool` | Returns the field schema for the descriptor dataclass of a given session type                             |

Read, write, and describe tools all take an explicit `session_type` — path resolution and class
selection are the caller's responsibility. The canonical on-disk path is always
`<session>/raw_data/session_descriptor.yaml`; `session_type` selects the parsing dataclass via
`DESCRIPTOR_REGISTRY`. To enumerate valid values call `list_supported_session_types_tool`. To
discover session roots, hand off to `/project-hierarchy` for `discover_sessions_tool`; to confirm a
descriptor file is actually present, hand off to `/session-data` for
`discover_session_descriptors_tool`. If you don't know the session type for a given session, hand
off to `/session-data` to call `read_session_data_tool` first — its returned payload includes
`session_type`.

`describe_session_descriptor_schema_tool` returns two keys: `session_type` (the validated
enum value) and `schema` (the field schema of the session-type's descriptor dataclass).

---

## Workflows

### Repairing a corrupted or stale descriptor

1. **Verify prerequisites:** MCP server connected (else `/assets-mcp-environment-setup`).
2. **Resolve the descriptor path.** Construct it as `<session>/raw_data/session_descriptor.yaml`
   from the session root (e.g. from `discover_sessions_tool` output). You also need the session's
   `session_type` — read it via `/session-data`'s `read_session_data_tool` if you don't already
   know it.
3. **Read the existing descriptor:**
   ```text
   read_session_descriptor_tool(
       file_path="<session>/raw_data/session_descriptor.yaml",
       session_type="<type>",
   )
   ```
4. **Inspect the schema for the session type:**
   ```text
   describe_session_descriptor_schema_tool(session_type="<type>")
   ```
5. **Build the corrected descriptor dictionary** based on the schema and the values you can recover.
6. **Confirm the planned write with the user.** `write_session_descriptor_tool` defaults to
   `overwrite=True`, so it silently clobbers the existing descriptor file with no backup. If
   the user wants the call to refuse-on-existing instead, pass `overwrite=False` explicitly in
   Step 7.
7. **Write the corrected descriptor:**
   ```text
   write_session_descriptor_tool(
       file_path="<session>/raw_data/session_descriptor.yaml",
       session_type="<type>",
       descriptor_payload={ ... },
       overwrite=True,   # default; set to False to refuse-on-existing
   )
   ```
   The kwargs are `file_path`, `session_type`, `descriptor_payload`, and the keyword-only
   `overwrite` (default `True`). The tool does not look at `session_data.yaml`; you choose both
   the destination path and the parsing class via `session_type`.
8. **Re-read to verify:**
   ```text
   read_session_descriptor_tool(
       file_path="<session>/raw_data/session_descriptor.yaml",
       session_type="<type>",
   )
   ```

### Amending a descriptor with post-hoc data

Use the same workflow above but mutate only the fields the user wants to amend (e.g., adding the actual
water volume delivered after a manual recount). Always confirm with the user before writing.

### Inspecting a descriptor without modification

```text
read_session_descriptor_tool(
    file_path="<session>/raw_data/session_descriptor.yaml",
    session_type="<type>",
)
```

If you don't know the session type, hand off to `/session-data` to call `read_session_data_tool`
first — its returned payload includes `session_type`.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] File path was constructed as <session>/raw_data/session_descriptor.yaml
- [ ] session_type was passed to both the read and the write tools
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
