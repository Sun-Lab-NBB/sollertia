---
name: experiment-pipeline
description: >-
  End-to-end orchestration guide for the Sollertia experiment lifecycle: phase ordering and
  handoff conditions from system bringup through experiment design, runtime acquisition, and
  post-acquisition handoff. Use when planning a full data collection workflow or deciding
  which experiment skill to invoke next.
user-invocable: true
---

# Sollertia experiment pipeline

End-to-end orchestration reference for the Sollertia experiment lifecycle. Covers canonical phase
ordering, handoff conditions to phase-specific skills, and the boundary between configuration-time
(AI-assisted) and runtime (deterministic) work.

---

## Scope

**Covers:**
- Canonical pipeline phase ordering for a Sollertia experiment
- Handoff conditions to phase-specific experiment skills
- Where the assets plugin and ataraxis plugins fit into the pipeline
- The configuration-time vs runtime boundary

**Does not cover:**
- Detailed tool usage for any individual phase (see phase-specific skills)
- MCP server connectivity (see `/experiment-mcp-environment-setup`)
- Authoring system / experiment / session YAML files (see assets plugin skills)
- Post-acquisition data analysis (see processing plugin)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the
relevant skill for detailed tool usage, parameter reference, and troubleshooting.

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: AI assistance operates at configuration time;
runtime acquisition is fully deterministic and AI-independent.

| Phase                                | AI-assisted? | Where it lives                                |
|--------------------------------------|--------------|-----------------------------------------------|
| Working directory + credentials      | yes          | assets plugin (`slsa mcp`)     |
| System configuration authoring       | yes          | assets plugin (`slsa mcp`)     |
| Hardware bringup and verification    | yes          | experiment plugin (`sle get mcp` + ataraxis)   |
| Experiment design (templates, states)| yes          | assets plugin (`slsa mcp`)     |
| Pre-session health check             | yes          | experiment plugin (`sle get mcp` + ataraxis)   |
| **Runtime data acquisition**         | **no**       | sollertia-experiment Python entry points only |
| Post-acquisition preprocessing       | yes          | experiment plugin (`sle manage mcp`)           |
| Data management (migrate / delete)   | yes          | experiment plugin (`sle manage mcp`)           |
| Post-acquisition data analysis       | yes          | processing plugin                             |

The MCP tool surface intentionally has no "start a recording session" tool. Runtime is launched only
through the `sle run` CLI, which reads validated configuration files written during the AI-assisted
phases.

---

## Pipeline phases

```text
Working      System         Hardware       Experiment     Pre-session    Runtime        Post-process    Handoff to
Directory →  Configuration →  Bringup    →  Design      →  Health Check →  Acquisition →  & Manage    →  Processing
    |              |              |              |              |              |              |              |
configuration  configuration  /acquisition-  configuration  /system-       (sle run CLI,   /data-         processing
 /working-     /system-       system-setup    /experiment-   health-       no MCP)         management    plugin
  directory     configuration                  configuration  check
```

### Phase 1: Working directory and credentials

- **Plugin / Skill:** assets plugin → `/working-directory`
- **Actions:** Set the local Sollertia working directory; configure Google Sheets credentials and task
  templates directory.
- **Handoff condition:** `read_working_directory_tool` returns the expected path.
- **Skip condition:** Working directory already initialized for this host.

### Phase 2: System configuration

- **Plugin / Skill:** experiment plugin → `/mesoscope-vr`
- **Actions:** Generate or edit `MesoscopeSystemConfiguration` YAML via the `sle get mcp` write
  tool. Author cameras, microcontrollers, file system paths, Google Sheets, and external assets
  (Zaber motor ports, Unity MQTT broker).
- **Handoff condition:** `read_system_configuration_tool` returns a valid configuration; the config
  passes schema validation.

### Phase 3: Hardware bringup

- **Skill:** `/acquisition-system-setup` (this plugin)
- **Actions:** Discover cameras, microcontrollers, motors, mesoscope. Validate the system configuration
  against discovered hardware. Update fields (camera indices, port assignments) using the configuration
  plugin's MCP write tools as needed.
- **Handoff condition:** All required hardware enumerated; system configuration matches reality.
- **Cross-plugin handoffs:**
  - `ataraxis@video:camera-setup` for camera discovery
  - `ataraxis@communication:microcontroller-setup` for microcontroller enumeration
  - `/zaber-interface` (this plugin) for Zaber motor discovery

### Phase 4: Experiment authoring

This phase is split across three assets plugin skills, invoked in dependency order. Each skill
owns exactly one slsa asset and the others must hand off to it.

- **Step 4a — `/project-hierarchy` (assets plugin):** Confirm the project under which the
  experiment will live exists on disk (read-only via `get_data_root_overview_tool`). Project
  directories are created implicitly when the first session lands there via the experiment
  plugin's `/managing-session-data` — there is no dedicated project-creation MCP tool.
- **Step 4b — `/task-templates` (assets plugin):** Author or load the task template that
  defines the VR environment, cue catalog, and trial structures (each trial owns its own segment
  geometry — there is no separate segment catalog at the template level). Owns `write_template_tool`.
  Hand off to the unity plugin's `/task-prefabs` if the template targets a Unity scene (prefab
  generation and zone validation).
- **Step 4c — `/experiment-configuration` (assets plugin):** Instantiate the template into a
  per-project experiment configuration, populate the state machine, and customize state durations,
  trial weights, and reward volumes. Owns `write_experiment_configuration_tool` and
  `create_experiment_config_tool`.
- **Handoff condition:** `read_experiment_configuration_tool` returns a validated experiment for the
  target project.

### Phase 5: Pre-session health check

- **Skill:** `/system-health-check` (this plugin)
- **Actions:** Verify network mounts, hardware connectivity, animal metadata in Google Sheets, project
  readiness. Light-touch sanity check before launching a runtime session.
- **Handoff condition:** All checklist items pass.

### Phase 6: Runtime acquisition (no AI)

- **Plugin / Skill:** none — invoked directly via the `sle run` CLI by the experimenter.
- **Actions:** `sle run` reads the validated system + experiment configuration files, dispatches a
  hardware-deterministic acquisition session, writes raw data + descriptors into the session directory.
- **Handoff condition:** Session terminates cleanly; `session_data.yaml` and the appropriate descriptor
  file (lick training / run training / window checking / mesoscope experiment) exist on disk.
- **Skill restriction:** This skill MUST NOT attempt to run sessions through MCP tools. There is no
  MCP tool that starts a runtime session, and there will not be one.

### Phase 7: Post-process and manage

- **Skill:** `/data-management` (this plugin)
- **Actions:** Run `preprocess_session_tool` to aggregate raw data, validate session contents, optionally
  migrate the animal between projects or delete the session.
- **Handoff condition:** Preprocessed session lives at the canonical storage tier; `processed_data` is
  populated.

### Phase 8: Handoff to processing

- **Plugin:** processing plugin
- **Status:** The processing plugin is currently a placeholder while sl-behavior is being absorbed into
  other Sollertia libraries. When the absorption completes, this phase will dispatch to processing-plugin
  skills for log archive extraction, behavior data processing, and downstream analysis.
- **For now:** Hand the user the path to the preprocessed session and tell them to use the relevant
  per-library tools directly.

---

## Decision tree: which skill to start from

```
Is the system already configured?
├─ no  → start at Phase 1 (assets plugin /working-directory)
└─ yes
    └─ Is the hardware verified for this session?
        ├─ no  → /system-health-check
        └─ yes
            └─ Does an experiment configuration exist for this project?
                ├─ no  → /project-hierarchy → /task-templates → /experiment-configuration
                └─ yes
                    └─ Is a session already recorded?
                        ├─ no  → user runs `sle run` (no AI involvement)
                        └─ yes
                            └─ Is preprocessing complete?
                                ├─ no  → /data-management
                                └─ yes → handoff to processing plugin
```

---

## Cross-plugin handoffs at a glance

| You need to…                                       | Use…                                                                           |
|----------------------------------------------------|--------------------------------------------------------------------------------|
| Set the working directory or credentials          | assets plugin `/working-directory`                                       |
| Author the system configuration YAML              | experiment plugin `/mesoscope-vr`                                            |
| Author the server (remote transfer) configuration | forging plugin `/server-configuration`                                          |
| Create a project                                  | assets plugin `/project-hierarchy`                                       |
| Author a task template                            | assets plugin `/task-templates`                                          |
| Author a per-project experiment configuration     | assets plugin `/experiment-configuration`                                |
| Read a session marker / inspect session metadata  | assets plugin `/session-data`                                            |
| Read or repair a session descriptor               | assets plugin `/session-descriptors`                                     |
| Read or patch a frozen runtime snapshot           | experiment plugin `/session-snapshots`                                       |
| Look up animal surgery / implants / drugs         | assets plugin `/subject-metadata`                                        |
| Curate or read a dataset                          | forging plugin `/datasets`                                                      |
| Discover GenICam cameras                          | `ataraxis@video:camera-setup`                                                   |
| Test camera acquisition interactively             | `ataraxis@video:camera-setup`                                                   |
| Discover microcontrollers / verify MQTT           | `ataraxis@communication:microcontroller-setup`                                  |
| Write a new VideoSystem binding                   | `ataraxis@video:camera-interface` (general) / `/mesoscope-vr` (Mesoscope-specific) |
| Write a new ModuleInterface                       | `/microcontroller-interface` → `ataraxis@communication:microcontroller-interface` |
| Write firmware for a new module                   | `ataraxis@microcontroller:firmware-module`                                      |
| Discover or configure Zaber motors                | `/zaber-interface`                                                              |
| Modify Mesoscope-VR hardware composition          | `/mesoscope-vr`                                                                 |
| Modify Mesoscope-VR runtime behavior              | `/mesoscope-vr-runtime`                                                         |
| Design a new acquisition system                   | `/acquisition-system-design`                                                    |
| Generate / verify Unity task prefab from template | unity plugin `/task-prefabs`                                              |
| Open / create a Unity scene                       | unity plugin `/task-scenes`                                                    |
| Enter / exit Unity Play Mode                      | unity plugin `/play-mode`                                                 |

---

## Verification checklist

```text
Pipeline orchestration:
- [ ] Identified the current phase and the next handoff condition
- [ ] Confirmed the relevant phase-specific skill was invoked (not duplicated inline)
- [ ] Did not attempt to run a runtime acquisition session through MCP tools
- [ ] Cross-plugin handoffs to ataraxis or assets plugin are explicit, not implicit
- [ ] Handoff condition for the current phase is satisfied before advancing to the next
```
