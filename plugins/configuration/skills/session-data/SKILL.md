---
name: configure-session-data
description: >-
  Discovers, reads, and writes session-level YAML files (SessionData, session descriptors, hardware
  state, Zaber positions, mesoscope positions, frozen system configuration, frozen experiment
  configuration) for sollertia-shared-assets via the sl-configure MCP server. Covers session directory
  discovery, descriptor types per session type, and the frozen configuration files captured at session
  start. Use when inspecting an existing session, recovering metadata, fixing a corrupted descriptor,
  or building tooling that needs session-level introspection.
user-invocable: true
---

# Sollertia session data

Discovers, reads, and writes the session-level YAML files that live inside every Sollertia session
directory. Uses the `sl-configure mcp` MCP server.

---

## Scope

**Covers:**
- Discovering sessions across projects and animals
- Reading and writing `SessionData` (the canonical session marker file)
- Reading and writing per-session-type descriptors (lick training, run training, window checking,
  mesoscope experiment)
- Reading and writing the hardware state, Zaber positions, and mesoscope positions snapshots captured
  at session start
- Reading the frozen system / experiment configuration files captured at session start
- Reading subject metadata (subject, surgery, implants, injections, drugs)

**Does not cover:**
- Authoring system or experiment configuration (see `/system-configuration`,
  `/experiment-configuration`)
- Authoring dataset metadata (see `/dataset-data`)
- Preprocessing, deleting, or migrating sessions (see the experiment plugin's `/data-management`)
- Initial working directory setup (see `/working-directory`)

---

## Anatomy of a session directory

Every Sollertia session is a directory containing:

```
<session-id>/
├── session_data.yaml                    # SessionData — the canonical marker
├── <session-type>_descriptor.yaml       # one of the descriptor types listed below
├── system_configuration.yaml            # frozen at session start
├── experiment_configuration.yaml        # frozen at session start (experiment sessions only)
├── mesoscope_hardware_state.yaml        # frozen at session start
├── zaber_positions.yaml                 # frozen at session start
├── mesoscope_positions.yaml             # frozen at session start
├── raw_data/                            # acquired data
└── processed_data/                      # populated by /data-management
```

The `SessionData` file is the discovery marker — `discover_sessions_tool` walks the working directory
tree and recognizes any directory containing a `session_data.yaml`.

### Session types and descriptors

| `SessionTypes` value     | Descriptor file                       | Descriptor dataclass                  |
|--------------------------|---------------------------------------|---------------------------------------|
| `lick training`          | `lick_training_descriptor.yaml`       | `LickTrainingDescriptor`              |
| `run training`           | `run_training_descriptor.yaml`        | `RunTrainingDescriptor`               |
| `window checking`        | `window_checking_descriptor.yaml`     | `WindowCheckingDescriptor`            |
| `mesoscope experiment`   | `mesoscope_experiment_descriptor.yaml`| `MesoscopeExperimentDescriptor`       |

Use `list_supported_session_types_tool` to query the canonical strings.

---

## MCP tool surface

### Discovery

| Tool                                  | Purpose                                                          |
|---------------------------------------|------------------------------------------------------------------|
| `discover_projects_tool`              | Lists all projects under the working directory                   |
| `discover_animals_tool`               | Lists animals within a project                                   |
| `discover_sessions_tool`              | Walks the working directory and lists sessions (filterable)      |
| `discover_session_descriptors_tool`   | Lists the descriptor file(s) in a specific session directory     |

### Read

| Tool                                            | Returns                                                  |
|-------------------------------------------------|----------------------------------------------------------|
| `read_session_data_tool`                        | `SessionData` (the canonical marker)                     |
| `read_session_descriptor_tool`                  | The descriptor for the session's type                    |
| `read_session_hardware_state_tool`              | `MesoscopeHardwareState` snapshot                        |
| `read_session_zaber_positions_tool`             | `ZaberPositions` snapshot                                |
| `read_session_mesoscope_positions_tool`         | `MesoscopePositions` snapshot                            |
| `read_session_system_configuration_tool`        | Frozen system configuration at session start             |
| `read_session_experiment_configuration_tool`    | Frozen experiment configuration at session start         |

### Write

| Tool                                          | Use case                                                    |
|-----------------------------------------------|-------------------------------------------------------------|
| `write_session_descriptor_tool`               | Repair or amend a descriptor (e.g., post-hoc water volume)  |
| `write_session_hardware_state_tool`           | Repair a corrupted hardware state file                      |
| `write_session_zaber_positions_tool`          | Patch Zaber positions after manual stage adjustment         |
| `write_session_mesoscope_positions_tool`      | Patch mesoscope positions after manual relocation           |

The schemas:

| Tool                                      | Returns                                                          |
|-------------------------------------------|------------------------------------------------------------------|
| `describe_session_descriptor_schema_tool` | Returns the descriptor schema for a given session type           |

### Subject metadata

| Tool                                | Purpose                                                       |
|-------------------------------------|---------------------------------------------------------------|
| `discover_subjects_tool`            | Lists all subjects (optionally filtered by project)           |
| `read_subject_tool`                 | Reads core subject metadata                                   |
| `read_subject_surgery_tool`         | Reads the subject's surgery records                           |
| `read_subject_implants_tool`        | Reads implant records                                         |
| `read_subject_injections_tool`      | Reads injection records                                       |
| `read_subject_drugs_tool`           | Reads drug administration records                             |
| `describe_surgery_schema_tool`      | Returns the surgery schema                                    |

---

## Common workflows

### Inspecting a specific session

1. `discover_sessions_tool(...)` — narrow by project, animal, or session type as needed.
2. `read_session_data_tool(session_path="<absolute>")` — confirm the marker file is intact.
3. `discover_session_descriptors_tool(session_path="<absolute>")` — confirm the descriptor file exists.
4. `read_session_descriptor_tool(session_path="<absolute>")` — read the descriptor.
5. (Optional) Read the frozen `_zaber_positions`, `_mesoscope_positions`, `_hardware_state`,
   `_system_configuration`, `_experiment_configuration` snapshots to reconstruct the runtime context.

### Repairing a descriptor

1. Read the existing descriptor with `read_session_descriptor_tool`.
2. Inspect the schema with `describe_session_descriptor_schema_tool(session_type="<type>")`.
3. Build the corrected descriptor dictionary.
4. **Confirm with the user before writing** — descriptor edits are irreversible without a backup.
5. Call `write_session_descriptor_tool(session_path="<absolute>", descriptor={ ... })`.
6. Re-read to verify.

### Recovering metadata for a session with a missing or corrupted descriptor

1. Read the frozen `system_configuration.yaml` and `experiment_configuration.yaml` from the session
   directory — these were captured at session start and should still be intact.
2. Read the frozen `hardware_state`, `zaber_positions`, and `mesoscope_positions` snapshots.
3. Reconstruct what you can from the raw_data directory contents and timestamps.
4. Hand the recovery proposal to the user before writing a replacement descriptor.

### Auditing subjects across sessions

Use the subject tools to enumerate animals and pull surgery / implant / injection / drug records when
you need a longitudinal view that crosses sessions.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] discover_sessions_tool was used to locate sessions, not raw filesystem walks
- [ ] describe_session_descriptor_schema_tool was called before any descriptor write
- [ ] User confirmed any write operation before it was executed
- [ ] read_*_tool was called after every write to verify the result
- [ ] Frozen system / experiment configuration was preserved (not overwritten with current values)
```

---

## Related skills

| Skill                                          | Relationship                                                            |
|------------------------------------------------|-------------------------------------------------------------------------|
| `/working-directory`                           | Required prerequisite — must be run first                               |
| `/configuration-mcp-environment-setup`         | Run first if the MCP server is not connected                            |
| `/dataset-data`                                | Sibling — datasets are higher-level groupings of sessions               |
| `/system-configuration`                        | The active system configuration becomes the frozen file at session start |
| `/experiment-configuration`                    | The active experiment configuration becomes the frozen file at session start |
| experiment plugin `/data-management`           | Preprocesses, migrates, and deletes sessions (mutating MCP tools)       |
