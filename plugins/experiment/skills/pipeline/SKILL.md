---
name: experiment-pipeline
description: >-
  End-to-end orchestration guide for the Sollertia experiment lifecycle. Covers canonical phase ordering
  with handoff conditions from system bringup through experiment design, runtime acquisition, and
  post-acquisition handoff to processing. Use when planning a full Sollertia data collection workflow,
  setting up a new acquisition system, or deciding which experiment skill to invoke at each step.
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
- Where the configuration plugin and ataraxis plugins fit into the pipeline
- The configuration-time vs runtime boundary

**Does not cover:**
- Detailed tool usage for any individual phase (see phase-specific skills)
- MCP server connectivity (see `/experiment-mcp-environment-setup`)
- Authoring system / experiment / session YAML files (see configuration plugin skills)
- Post-acquisition data analysis (see processing plugin)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the
relevant skill for detailed tool usage, parameter reference, and troubleshooting.

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: AI assistance operates at configuration time;
runtime acquisition is fully deterministic and AI-independent.

| Phase                                | AI-assisted? | Where it lives                                |
|--------------------------------------|--------------|-----------------------------------------------|
| Working directory + credentials      | yes          | configuration plugin (`sl-configure mcp`)     |
| System configuration authoring       | yes          | configuration plugin (`sl-configure mcp`)     |
| Hardware bringup and verification    | yes          | experiment plugin (`sl-get mcp` + ataraxis)   |
| Experiment design (templates, states)| yes          | configuration plugin (`sl-configure mcp`)     |
| Pre-session health check             | yes          | experiment plugin (`sl-get mcp` + ataraxis)   |
| **Runtime data acquisition**         | **no**       | sollertia-experiment Python entry points only |
| Post-acquisition preprocessing       | yes          | experiment plugin (`sl-manage mcp`)           |
| Data management (migrate / delete)   | yes          | experiment plugin (`sl-manage mcp`)           |
| Post-acquisition data analysis       | yes          | processing plugin                             |

The MCP tool surface intentionally has no "start a recording session" tool. Runtime is launched only
through the `sl-run` CLI, which reads validated configuration files written during the AI-assisted
phases.

---

## Pipeline phases

```text
Working      System         Hardware       Experiment     Pre-session    Runtime        Post-process    Handoff to
Directory →  Configuration →  Bringup    →  Design      →  Health Check →  Acquisition →  & Manage    →  Processing
    |              |              |              |              |              |              |              |
configuration  configuration  /acquisition-  configuration  /system-       (sl-run CLI,   /data-         processing
 /working-     /system-       system-setup    /experiment-   health-       no MCP)         management    plugin
  directory     configuration                  configuration  check
```

### Phase 1: Working directory and credentials

- **Plugin / Skill:** configuration plugin → `/working-directory`
- **Actions:** Set the local Sollertia working directory; configure Google Sheets credentials and task
  templates directory.
- **Handoff condition:** `read_working_directory_tool` returns the expected path.
- **Skip condition:** Working directory already initialized for this host.

### Phase 2: System configuration

- **Plugin / Skill:** configuration plugin → `/system-configuration`
- **Actions:** Generate or edit `MesoscopeSystemConfiguration` YAML via the `sl-configure mcp` write
  tools. Author cameras, microcontrollers, file system paths, Google Sheets, external assets, server
  configuration.
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

### Phase 4: Experiment design

- **Plugin / Skill:** configuration plugin → `/experiment-configuration`
- **Actions:** Author or load a task template, customize trial parameters, populate experiment state
  machine, write the per-project experiment configuration.
- **Handoff condition:** `read_experiment_configuration_tool` returns a validated experiment for the
  target project.

### Phase 5: Pre-session health check

- **Skill:** `/system-health-check` (this plugin)
- **Actions:** Verify network mounts, hardware connectivity, animal metadata in Google Sheets, project
  readiness. Light-touch sanity check before launching a runtime session.
- **Handoff condition:** All checklist items pass.

### Phase 6: Runtime acquisition (no AI)

- **Plugin / Skill:** none — invoked directly via the `sl-run` CLI by the experimenter.
- **Actions:** `sl-run` reads the validated system + experiment configuration files, dispatches a
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
├─ no  → start at Phase 1 (configuration plugin /working-directory)
└─ yes
    └─ Is the hardware verified for this session?
        ├─ no  → /system-health-check
        └─ yes
            └─ Does an experiment configuration exist for this project?
                ├─ no  → configuration plugin /experiment-configuration
                └─ yes
                    └─ Is a session already recorded?
                        ├─ no  → user runs `sl-run` (no AI involvement)
                        └─ yes
                            └─ Is preprocessing complete?
                                ├─ no  → /data-management
                                └─ yes → handoff to processing plugin
```

---

## Cross-plugin handoffs at a glance

| You need to…                                | Use…                                              |
|---------------------------------------------|---------------------------------------------------|
| Author a system or experiment YAML          | configuration plugin (`/system-configuration`, `/experiment-configuration`) |
| Discover GenICam cameras                    | `ataraxis@video:camera-setup`                     |
| Test camera acquisition interactively       | `ataraxis@video:camera-setup`                     |
| Discover microcontrollers / verify MQTT     | `ataraxis@communication:microcontroller-setup`    |
| Write a new VideoSystem binding             | `/camera-interface` → `ataraxis@video:camera-interface` |
| Write a new ModuleInterface                 | `/microcontroller-interface` → `ataraxis@communication:microcontroller-interface` |
| Write firmware for a new module             | `ataraxis@microcontroller:firmware-module`        |
| Discover or configure Zaber motors          | `/zaber-interface`                                |
| Modify the Mesoscope-VR system itself       | `/modifying-mesoscope-vr-system`                  |
| Verify Unity task template values           | `/configuration-verification`                     |

---

## Verification checklist

```text
Pipeline orchestration:
- [ ] Identified the current phase and the next handoff condition
- [ ] Confirmed the relevant phase-specific skill was invoked (not duplicated inline)
- [ ] Did not attempt to run a runtime acquisition session through MCP tools
- [ ] Cross-plugin handoffs to ataraxis or configuration plugin are explicit, not implicit
- [ ] Handoff condition for the current phase is satisfied before advancing to the next
```
