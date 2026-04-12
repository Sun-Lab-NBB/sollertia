---
name: behavior-session-setup
description: >-
  Discovers and filters sessions eligible for behavior processing via the sollertia-forgery MCP server.
  Owns `discover_behavior_sessions_tool` usage and the behavior-processing eligibility rules; delegates
  session marker format, SessionTypes surface, project hierarchy, and session directory layout to the
  configuration plugin's dedicated skills. Use when locating processable sessions ahead of a
  behavior-processing batch.
user-invocable: true
---

# Behavior session setup

Discovers sessions eligible for behavior processing and produces the confirmed `session_paths` list
that `/behavior-processing` consumes. This skill is deliberately narrow — it only owns the
behavior-processing eligibility layer and the `discover_behavior_sessions_tool` MCP surface. Session
metadata, directory layout, and project hierarchy semantics are documented in the configuration
plugin. Output locations are resolved statically by the pipeline (always under the session's
`processed_data_path/behavior_data/`) and are not configurable from any layer.

---

## Scope

**Covers:**
- `discover_behavior_sessions_tool` usage, parameters, and return structure
- Behavior-processing eligibility rules (`PROCESSABLE_SESSION_TYPES`)

**Does not cover:**
- Session marker file, `SessionData` loading, and the `SessionTypes` enum (see `/configuration:session-data`)
- Project / animal / session hierarchy conventions (see `/configuration:project-hierarchy`)
- Session-level frozen runtime snapshots (MesoscopeHardwareState, ZaberPositions, MesoscopePositions;
  see `/configuration:session-snapshots`)
- Per-session descriptor YAML files (see `/configuration:session-descriptors`)
- Experiment configuration YAML files (see `/configuration:experiment-configuration`)
- Raw input artifacts: runtime NPZ, camera feathers, microcontroller feathers (see `/behavior-input-format`)
- Batch preparation, execution, and monitoring (see `/behavior-processing`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)

---

## Handoff rules

If MCP tools are unavailable, invoke `/forging-mcp-environment-setup`. Once discovery completes and
the user confirms the session subset, hand off to `/behavior-processing`. If the user asks about
session metadata shape or layout beyond what the tool returns, invoke `/configuration:session-data` or
`/configuration:project-hierarchy`. If the user asks where behavior outputs will live, point at
`{session_root}/processed_data/behavior_data/` — the location is static and cannot be overridden.

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for session discovery. Do not import
`sollertia_forgery.processing.pipeline` helpers directly or scan the filesystem manually.

You MUST run `discover_behavior_sessions_tool` before calling any batch preparation tool. The caller
must provide confirmed session root paths — do not guess or derive them manually.

---

## Available tool

| Tool                               | Purpose                                                                    |
|------------------------------------|----------------------------------------------------------------------------|
| `discover_behavior_sessions_tool`  | Walks a root directory for session markers and filters by eligibility      |

**Parameters:**

| Parameter        | Type  | Default    | Description                                           |
|------------------|-------|------------|-------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to the root directory to search         |

**Return structure:**

```text
sessions[]:              Per-session entries:
  session_path:          Absolute path to the session root directory
  session_name:          Human-readable session name from SessionData
  animal_id:             Animal identifier from SessionData
  session_type:          String form of the SessionType enum
  raw_data_path:         Absolute path to the session's raw_data subdirectory
  processed_data_path:   Absolute path to the session's processed_data subdirectory
  eligible:              Boolean — True if the session is eligible for behavior processing
  error:                 (only if discovery failed) human-readable error message
session_paths:           Flat list of eligible session root paths (pass directly to /behavior-processing)
total_sessions:          Number of session entries returned
total_eligible:          Number of eligible session entries
error:                   (only on top-level failure) directory-level error message
```

`session_paths` is the pre-filtered list of eligible roots — pass it directly to
`prepare_behavior_processing_batch_tool` rather than re-filtering on `eligible` client-side.

For the authoritative meaning of `session_name`, `animal_id`, `session_type`, and the `raw_data_path`
/ `processed_data_path` layout, see `/configuration:session-data` and `/configuration:project-hierarchy`. This skill does not
redocument those fields.

---

## Eligibility rules

A session is eligible for behavior processing only if its `session_type` is in
`PROCESSABLE_SESSION_TYPES`:

| Session type           | Eligible | Notes                                                     |
|------------------------|----------|-----------------------------------------------------------|
| `LICK_TRAINING`        | yes      | Runtime, camera, and microcontroller jobs                 |
| `RUN_TRAINING`         | yes      | Runtime, camera, and microcontroller jobs                 |
| `MESOSCOPE_EXPERIMENT` | yes      | Same as training plus experiment-specific runtime outputs |
| (anything else)        | no       | Returned with `eligible=False` and no error               |

Only `MESOSCOPE_EXPERIMENT` sessions trigger the experiment-specific runtime outputs
(`reinforcing_guidance_state_data.feather`, `aversive_guidance_state_data.feather`,
`vr_cue_data.feather`, `vr_trigger_zone_data.feather`, `trial_data.feather`). Training sessions
produce only the state feathers (`system_state_data.feather`, `runtime_state_data.feather`) plus any
eligible microcontroller / camera outputs.

The authoritative `SessionTypes` enum surface lives in `/configuration:session-data`. This skill only filters the
enum — it does not define it.

Eligibility is determined at discovery time. A session listed as `eligible: true` still goes
through a second check in `prepare_behavior_processing_batch_tool`, which re-loads `SessionData`,
re-validates its type against `PROCESSABLE_SESSION_TYPES`, requires a `*hardware_state*.yaml` file
to exist under `raw_data/`, and rejects the session unless at least one runtime / camera /
microcontroller job is found on disk.

---

## Discovery workflow

### Step 1: Ask for a root directory

The user must provide an absolute path to a directory that contains one or more sessions. The
directory is searched recursively. A project-level root (e.g., `/data/projects/my_project`) is the
typical input — do not assume a default. For project hierarchy conventions, see `/configuration:project-hierarchy`.

### Step 2: Run discovery

Call `discover_behavior_sessions_tool` with the confirmed `root_directory`. Present the result as a
readable summary so the user can see what was found:

```text
**Session Discovery** — /data/projects/my_project
Found 12 session(s): 9 eligible, 3 ineligible

| Session                         | Animal | Type                   | Eligible |
|---------------------------------|--------|------------------------|----------|
| mouse_001_2026-03-04_lick_01    | 001    | LICK_TRAINING          | yes      |
| mouse_001_2026-03-05_run_01     | 001    | RUN_TRAINING           | yes      |
| mouse_002_2026-03-06_exp_01     | 002    | MESOSCOPE_EXPERIMENT   | yes      |
| mouse_003_2026-03-07_habituation| 003    | WINDOW_CHECKING        | no       |
```

Surface any sessions returned with an `error` field — broken session metadata often points at
upstream acquisition bugs that should be repaired via `/configuration:session-data` or `/configuration:session-descriptors`.

### Step 3: Confirm the subset to process

Ask the user whether to process all eligible sessions or a subset. They may want to exclude sessions
by animal, date, or session name even when they are eligible. Produce the final list of session
paths for the batch.

### Step 4: Hand off

Pass the confirmed `session_paths` list to `/behavior-processing` for
`prepare_behavior_processing_batch_tool`. Do not call the preparation tool from this skill directly.
Behavior outputs will be written under `{session_root}/processed_data/behavior_data/` for every
session in the batch — no output directory list is required or accepted.

---

## Error routing

| Error                               | Resolution                                                                                                        |
|-------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| `Directory does not exist`          | Verify the root directory path with the user                                                                      |
| `Path is not a directory`           | The path points to a file; ask the user for the correct directory                                                 |
| `Permission denied during search`   | Filesystem ACLs block recursion; resolve or try a subdirectory                                                    |
| `Unable to load session: <reason>`  | Session metadata is corrupt or missing; see `/configuration:session-data` or `/configuration:session-descriptors` |
| `sessions=[]` or `total_eligible=0` | No markers found, or no eligible session types; verify the search root                                            |
| MCP tool unavailable                | Invoke `/forging-mcp-environment-setup`                                                                           |

---

## Related skills

| Skill                                     | Relationship                                                                       |
|-------------------------------------------|------------------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`          | Prerequisite: MCP server connectivity                                              |
| `/configuration:session-data`             | Reference: SessionData marker and SessionTypes enum                                |
| `/configuration:project-hierarchy`        | Reference: project / animal / session layout                                       |
| `/configuration:session-snapshots`        | Reference: MesoscopeHardwareState snapshot YAML                                    |
| `/configuration:session-descriptors`      | Reference: per-session descriptor YAML                                             |
| `/configuration:experiment-configuration` | Reference: MesoscopeExperimentConfiguration YAML                                   |
| `/behavior-input-format`                  | Reference: behavior-specific input artifacts and cross-library handoff             |
| `/behavior-processing`                    | Downstream: consumes the confirmed session_paths list                              |
| `/behavior-results`                       | Downstream: reads outputs written under `{session}/processed_data/behavior_data/`  |

---

## Verification checklist

```text
Behavior Session Setup:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Confirmed root directory with user
- [ ] Ran discover_behavior_sessions_tool and recorded at least one eligible session
- [ ] Surfaced ineligible and errored sessions to the user
- [ ] Confirmed the session_paths subset with user
- [ ] Handed off to /behavior-processing
```
