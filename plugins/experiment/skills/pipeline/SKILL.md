---
name: pipeline
description: >-
  End-to-end orchestration guide for the Sollertia experiment lifecycle: phase ordering and
  handoff conditions from system bringup through experiment design, runtime acquisition, and
  post-acquisition handoff. Use when planning a full data collection workflow or deciding
  which experiment skill to invoke next.
user-invocable: false
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
- MCP server connectivity (see each plugin's `*-mcp-environment-setup` skill, e.g.
  `/experiment-mcp-environment-setup`, `/assets-mcp-environment-setup`)
- Authoring system / experiment / session YAML files (see assets plugin skills)
- Post-acquisition data analysis (see forging plugin)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the
relevant skill for detailed tool usage, parameter reference, and troubleshooting.

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: AI assistance operates at configuration time;
runtime acquisition is fully deterministic and AI-independent.

| Phase                                 | AI-assisted? | Where it lives                                |
|---------------------------------------|--------------|-----------------------------------------------|
| Working directory + credentials       | yes          | assets plugin (`slsa mcp`)                    |
| System configuration authoring        | yes          | experiment plugin (`sle mcp`)                 |
| Hardware bringup and verification     | yes          | experiment plugin (`sle mcp` + ataraxis)      |
| Experiment design (templates, states) | yes          | assets plugin (`slsa mcp`)                    |
| Pre-session health check              | yes          | experiment plugin (`sle mcp` + ataraxis)      |
| **Runtime data acquisition**          | **no**       | sollertia-experiment Python entry points only |
| Post-acquisition preprocessing        | yes          | experiment plugin (`sle mcp`)                 |
| Data management (migrate / delete)    | yes          | experiment plugin (`sle mcp`)                 |
| Post-acquisition data analysis        | yes          | forging plugin                                |

The MCP tool surface intentionally has no "start a recording session" tool. Runtime is launched only
through the active system's run CLI (for the `mesoscope` system, `sle mesoscope run <mode>`), which
reads validated configuration files written during the AI-assisted phases.

---

## Pipeline phases

```text
1. Working directory      assets plugin /working-directory
2. System configuration   active system's skill (mesoscope → /mesoscope-vr)
3. Hardware bringup        /acquisition-system-setup
4. Experiment design       assets plugin /project-hierarchy → /task-templates → /experiment-configuration
5. Pre-session check       /system-health-check
6. Runtime acquisition     active system run CLI, e.g. sle mesoscope run <mode>  (no MCP, no AI)
7. Post-process & manage   /data-management
8. Handoff to forging      forging plugin
```

### Phase 1: Working directory and credentials

- **Plugin / Skill:** assets plugin → `/working-directory`
- **Actions:** Set the local Sollertia working directory; configure Google Sheets credentials and task
  templates directory.
- **Handoff condition:** `get_platform_environment_status_tool` reports the data root (and
  credentials/templates) healthy.
- **Skip condition:** The platform data root is already initialized for this host.

### Phase 2: System configuration

- **Plugin / Skill:** the active acquisition system's skill (for the `mesoscope` system,
  experiment plugin → `/mesoscope-vr`). Resolve the system type to its owning skill via
  `/acquisition-system-setup`'s **Supported acquisition systems** registry.
- **Actions:** Generate or edit the active system's configuration YAML via the `sle mcp` write
  tool (for the `mesoscope` system, `MesoscopeSystemConfiguration`: cameras, microcontrollers,
  filesystem paths, Google Sheets, and VR task assets — Zaber motor ports and the nested
  `vr_task` Unity MQTT configuration).
- **Handoff condition:** `read_system_configuration_tool` returns a valid configuration; the config
  passes schema validation.

### Phase 3: Hardware bringup

- **Plugin / Skill:** `/acquisition-system-setup` (this plugin)
- **Actions:** Discover the active system's hardware — cameras and microcontrollers (the platform-universal
  stack), plus any system-specific devices it composes (for the `mesoscope` system, Zaber motors and the
  mesoscope). Validate the system configuration against discovered hardware. Update fields (camera indices,
  port assignments) using the configuration plugin's MCP write tools as needed.
- **Handoff condition:** All required hardware enumerated; system configuration matches reality.
- **Cross-plugin handoffs:**
  - `ataraxis@video:camera-setup` for camera discovery
  - `ataraxis@communication:microcontroller-setup` for microcontroller enumeration
  - for any third-party-SDK subsystems the active system composes, that subsystem's skill (for the
    `mesoscope` system: Zaber motors via `/zaber-interface`)

### Phase 4: Experiment authoring

This phase is split across three assets plugin skills, invoked in dependency order. Each skill
owns exactly one slsa asset and the others must hand off to it.

- **Step 4a — `/project-hierarchy` (assets plugin):** Confirm the project under which the
  experiment will live exists on disk (read-only via `get_data_root_overview_tool`). Projects must
  exist before a session can be recorded — they are created by the `slsa configure project` CLI
  command (`SessionData.create` raises `FileNotFoundError` if the project is missing). There is no
  dedicated project-creation MCP tool.
- **Step 4b — `/task-templates` (assets plugin):** Author or load the task template that
  defines the VR environment, cue catalog, and trial structures (each trial owns its own segment
  geometry — there is no separate segment catalog at the template level). Owns `write_template_tool`.
  Hand off to the unity plugin's `/task-prefabs` if the template targets a Unity scene (prefab
  generation and zone validation).
- **Step 4c — `/experiment-configuration` (assets plugin):** Instantiate the template into a
  per-project experiment configuration, populate the state machine, and customize state durations,
  trial weights, and reward volumes. Owns `write_experiment_configuration_tool` and
  `create_experiment_configuration_tool`.
- **Handoff condition:** `read_experiment_configuration_tool` returns a validated experiment for the
  target project.

### Phase 5: Pre-session health check

- **Plugin / Skill:** `/system-health-check` (this plugin)
- **Actions:** Verify network mounts, hardware connectivity, animal metadata in Google Sheets, project
  readiness. For an experiment session, also confirm the Unity Editor MCP Bridge is reachable
  (`check_unity_bridge_tool` / `sle get unity`) so the run CLI can open the scene and arm the VR task.
  Light-touch sanity check before launching a runtime session.
- **Handoff condition:** All checklist items pass.

### Phase 6: Runtime acquisition (no AI)

- **Plugin / Skill:** none — invoked directly via the active system's run CLI by the experimenter
  (for the `mesoscope` system, `sle mesoscope run <mode>`, where `<mode>` is one of `window-checking`,
  `lick-training`, `run-training`, or `experiment`).
- **Actions:** The run CLI reads the validated system + experiment configuration files, dispatches a
  hardware-deterministic acquisition session, writes raw data + descriptors into the session directory.
  For an experiment session, the Unity Editor must be open before launch — the run CLI drives scene
  activation and Play Mode through the editor MCP Bridge and blocks until the bridge is reachable.
- **Handoff condition:** Session terminates cleanly; `session_data.yaml` and the appropriate session
  descriptor for the runtime mode exist on disk (for the `mesoscope` system: lick training /
  run training / window checking / mesoscope experiment).
- **Skill restriction:** You MUST NOT attempt to run sessions through MCP tools. There is no
  MCP tool that starts a runtime session, and there will not be one.

### Phase 7: Post-process and manage

- **Plugin / Skill:** `/data-management` (this plugin)
- **Actions:** Run `preprocess_session_tool` to aggregate raw data, validate session contents, optionally
  migrate the animal between projects or delete the session.
- **Handoff condition:** Preprocessed session lives at the canonical storage tier; `processed_data` is
  populated.

### Phase 8: Handoff to forging

- **Plugin / Skill:** forging plugin
- **Actions:** Once a session is preprocessed and transferred to long-term storage, hand off to the
  forging plugin's behavior processing subsystem — session discovery / transfer
  (`/session-transfer`), batch behavior processing (`/behavior-processing`), output verification
  (`/behavior-results`), and dataset curation (`/datasets`).
- **Handoff condition:** The preprocessed session is present on the storage destination the forging
  plugin reads from.

---

## Decision tree: which skill to start from

```text
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
                        ├─ no  → user runs the active system's run CLI, e.g. `sle mesoscope run <mode>` (no AI involvement)
                        └─ yes
                            └─ Is preprocessing complete?
                                ├─ no  → /data-management
                                └─ yes → handoff to forging plugin
```

---

## Cross-plugin handoffs at a glance

Rows naming a Mesoscope-VR skill illustrate the active system's hand-offs; another acquisition system
substitutes its own system skill (resolved via `/acquisition-system-setup`'s **Supported acquisition
systems** registry).

| You need to…                                      | Use…                                                                               |
|---------------------------------------------------|------------------------------------------------------------------------------------|
| Set the working directory or credentials          | assets plugin `/working-directory`                                                 |
| Author the active system's configuration YAML     | that system's skill (for `mesoscope`, `/mesoscope-vr`)                             |
| Author the server (remote transfer) configuration | forging plugin `/server-configuration`                                             |
| Create a project                                  | `slsa configure project` CLI (see assets plugin `/project-hierarchy`)              |
| Author a task template                            | assets plugin `/task-templates`                                                    |
| Author a per-project experiment configuration     | assets plugin `/experiment-configuration`                                          |
| Read a session marker / inspect session metadata  | assets plugin `/session-data`                                                      |
| Read or repair a session descriptor               | assets plugin `/session-descriptors`                                               |
| Read or patch a frozen runtime snapshot           | experiment plugin `/mesoscope-vr-snapshots`                                        |
| Look up animal surgery / implants / drugs         | assets plugin `/subject-metadata`                                                  |
| Curate or read a dataset                          | forging plugin `/datasets`                                                         |
| Discover GenICam cameras                          | `ataraxis@video:camera-setup`                                                      |
| Test camera acquisition interactively             | `ataraxis@video:camera-setup`                                                      |
| Verify a camera against its stored GenICam config | `/system-health-check` (verify) / `/acquisition-system-setup` (at bringup)         |
| Dump or restore a camera's GenICam config         | `ataraxis@video:camera-setup` (path sourced from the system configuration)         |
| Discover microcontrollers / verify MQTT           | `ataraxis@communication:microcontroller-setup`                                     |
| Write a new VideoSystem binding                   | `ataraxis@video:camera-interface` (general) / `/mesoscope-vr` (Mesoscope-specific) |
| Write a new ModuleInterface                       | `/microcontroller-interface` → `ataraxis@communication:microcontroller-interface`  |
| Write firmware for a new module                   | `ataraxis@microcontroller:firmware-module`                                         |
| Discover or configure Zaber motors                | `/zaber-interface`                                                                 |
| Modify Mesoscope-VR hardware composition          | `/mesoscope-vr`                                                                    |
| Modify Mesoscope-VR runtime behavior              | `/mesoscope-vr-runtime`                                                            |
| Drive the Unity VR task / MQTT coupling           | `/vr-driver-interface`                                                             |
| Design a new acquisition system (static)          | `/acquisition-system-design`                                                       |
| Implement an acquisition-system runtime loop      | `/acquisition-system-runtime`                                                      |
| Generate / verify Unity task prefab from template | unity plugin `/task-prefabs`                                                       |
| Open / create a Unity scene                       | unity plugin `/task-scenes`                                                        |
| Enter / exit Unity Play Mode                      | unity plugin `/play-mode`                                                          |

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
