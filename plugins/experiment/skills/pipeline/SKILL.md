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

End-to-end orchestration reference for the Sollertia experiment lifecycle. Covers canonical phase ordering, handoff
conditions to phase-specific skills, and the boundary between configuration-time (AI-assisted) and runtime
(deterministic) work.

---

## Scope

**Covers:**
- Canonical pipeline phase ordering for a Sollertia experiment
- Handoff conditions to phase-specific experiment skills
- Where the assets plugin and the ataraxis plugins fit into the pipeline
- The configuration-time versus runtime boundary

**Does not cover:**
- Designing or building a new acquisition system type, owned by `/system-design-pipeline`
- The library seams a new acquisition system touches, owned by `/library-extension`
- Detailed tool usage for any individual phase, owned by each phase-specific skill
- MCP server connectivity, owned by each plugin's `*-mcp-environment-setup` skill, such as
  `/experiment-mcp-environment-setup` and `assets:assets-mcp-environment-setup`
- Authoring system, experiment, or session YAML files. The active system's skill authors the system configuration
  through the `sle mcp` write tool, and the assets plugin skills author experiment and session files
- The current worked example's run CLI, runtime modes, and session-lifecycle specifics, owned by
  `mesoscope:mesoscope-vr-runtime`
- Post-acquisition data processing, owned by the forging plugin

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the relevant skill for
detailed tool usage, parameter reference, and troubleshooting.

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: AI assistance operates at configuration time, and runtime
acquisition is fully deterministic and AI-independent.

| Phase                                 | AI-assisted? | Where it lives                             |
|---------------------------------------|--------------|--------------------------------------------|
| Working directory + credentials       | yes          | assets plugin (`slsa mcp`)                 |
| System configuration authoring        | yes          | experiment plugin (`sle mcp`)              |
| Hardware bringup and verification     | yes          | experiment plugin (`sle mcp` + ataraxis)   |
| Experiment design (templates, states) | yes          | assets plugin (`slsa mcp`)                 |
| Pre-session health check              | yes          | experiment plugin (`sle mcp` + ataraxis)   |
| **Runtime data acquisition**          | **no**       | sollertia-experiment CLI entry points only |
| Post-acquisition preprocessing        | yes          | experiment plugin (`sle mcp`)              |
| Data management (migrate / delete)    | yes          | experiment plugin (`sle mcp`)              |
| Post-acquisition data processing      | yes          | forging plugin (optional, separate infra)  |

The `sle mcp` tool surface holds no tool that starts a recording session. Runtime is launched only through the active
system's run CLI, `sle <system> run <mode>`, which reads the validated configuration files the AI-assisted phases
wrote. `AcquisitionSystems` currently holds one member, so Mesoscope-VR is the only registered system, and
`mesoscope:mesoscope-vr-runtime` owns its run CLI.

---

## Pipeline phases

```text
1. Working directory      assets:working-directory
2. System configuration   the active system's skill (mesoscope → mesoscope:mesoscope-vr)
3. Hardware bringup       /acquisition-system-setup
4. Experiment design      assets:project-hierarchy → assets:task-templates → assets:experiment-configuration
5. Pre-session check      /system-health-check
6. Runtime acquisition    the active system's run CLI, sle <system> run <mode>  (no MCP, no AI)
7. Post-process & manage  /data-management
8. Handoff to forging     forging plugin  (optional, often separate infrastructure)
```

### Phase 1: Working directory and credentials

- **Plugin / Skill:** assets plugin → `assets:working-directory`
- **Actions:** Set the local Sollertia working directory, which is always required. Set the task templates directory for
  systems that run the corridor task, because every session type in `SESSION_TYPES_USING_VR_TASK` seeds its
  configuration from a task template and caches a `vr_configuration.yaml` snapshot
  (`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`). Configure platform credentials by category only
  when the system integrates with the matching external service. `CredentialsTypes` currently declares one category,
  `google`, used for every Google Sheets interaction (`enums.py`).
- **Handoff condition:** `get_platform_environment_status_tool` reports `overall_ok`, which is computed from the
  required components alone, and the working directory is the only required component
  (`sollertia-shared-assets/src/sollertia_shared_assets/interfaces/configuration_tools.py`). The data root, the task
  templates directory, and each credentials category report as separate optional components. Configure the data root
  here too, so the host can record and run sessions.
- **Skip condition:** The platform data root is already initialized for this host.

### Phase 2: System configuration

- **Plugin / Skill:** the active acquisition system's skill. Resolve the system type to its owning skill through
  `/acquisition-system-setup`'s **Supported acquisition systems** registry. Mesoscope-VR is the current worked
  example. See `mesoscope:mesoscope-vr`.
- **Actions:** Generate or edit the active system's configuration YAML through that system's `sle mcp` write tool. The
  canonical filename is derived from the system's `AcquisitionSystems` value as `<system>_system_configuration.yaml`,
  which `_system_configuration_filename()` composes (`cross_system/system_configuration.py`).
- **Handoff condition:** The system's configuration read tool returns a loadable configuration, and its validation
  tool reports the configuration as valid with an empty issue list.

### Phase 3: Hardware bringup

- **Plugin / Skill:** `/acquisition-system-setup` (this plugin)
- **Actions:** Discover the active system's hardware, which spans the platform-universal cameras, microcontrollers,
  and Zaber motors, plus any system-specific scientific instrument the system composes. Validate the system
  configuration against the discovered hardware. Update the drifted fields, such as camera indices and port
  assignments, through the system's `sle mcp` configuration write tool. Where a device carries cached configuration
  in the system configuration directory, apply it as part of bringup. Restore each camera's GenICam configuration
  from its cached file through `video:camera-setup` (ataraxis marketplace), with the path sourced from the system
  configuration.
- **Handoff condition:** All required hardware is enumerated and the system configuration matches reality.
- **Cross-plugin handoffs:**
  - `video:camera-setup` for camera discovery
  - `communication:microcontroller-setup` for microcontroller enumeration
  - `/zaber-interface` for Zaber motor discovery and validation, which is domain-general rather than system-specific
  - that system's skill for any system-specific instrument the active system composes. For the current worked
    example, see `mesoscope:mesoscope-vr`

### Phase 4: Experiment authoring

This phase spans three assets plugin skills, each owning exactly one slsa asset. Step 4a (project), Step 4b (task
template), and Step 4c (experiment configuration) are all required for a session type that runs the corridor task.
Every path cited in this phase is relative to `sollertia-shared-assets/src/sollertia_shared_assets/`.

- **Step 4a, `assets:project-hierarchy`:** Confirm the project under which the experiment will live exists on disk
  (`get_data_root_overview_tool`), or create it with `create_project_tool`, equivalently the `slsa configure project`
  CLI. Projects must exist before a session can be recorded, because `SessionData.create` raises `FileNotFoundError`
  when the project is missing (`data_hierarchy/session_data.py`). Both the discovery tool and `create_project_tool` are
  owned by `assets:project-hierarchy`.
- **Step 4b, `assets:task-templates`:** A `TaskTemplate` is the corridor task asset, holding the cue catalog, the VR
  environment, and the per-trial corridor geometry. Each trial owns its own geometry, so no separate segment catalog
  exists on the `TaskTemplate` dataclass in `configuration/vr_configuration.py`. Author or load the template, which owns
  `write_template_tool`, then hand off to `unity:task-prefabs` for prefab generation and zone validation and to
  `unity:scene-setup` for the editor-side scene and display rig.
- **Step 4c, `assets:experiment-configuration`:** Author the per-project experiment configuration, which holds the trial
  structures, the experiment state machine, and the per-state runtime parameters the active system declares.
  `create_experiment_from_vr_template_tool` builds the configuration through the system's experiment-configuration
  class, infers `unity_scene_name` from the template filename stem, seeds `state_count` default-valued states, and
  returns `file_path`, `acquisition_system`, `template_path`, and `data` (`interfaces/configuration_tools.py`).
  `write_experiment_configuration_tool` authors or repairs the full payload directly.
- **Handoff condition:** `read_experiment_configuration_tool` returns a validated experiment for the target project.

### Phase 5: Pre-session health check

- **Plugin / Skill:** `/system-health-check` (this plugin)
- **Actions:** Verify platform configuration prerequisites, network mounts, hardware connectivity, configuration
  validity, and project readiness. Google credentials report as a configured-or-not platform component only, and the
  sheets themselves are read at preprocessing time rather than here. For a session type in
  `SESSION_TYPES_USING_VR_TASK`, also confirm the Unity Editor MCP Bridge is reachable through
  `check_unity_bridge_tool` (`interfaces/get_tools.py`) or `sle get unity`, whose `get_unity_bridge()` command lives in
  `interfaces/get.py`, so the run CLI can open the scene and arm the VR task. Session types outside that set run no
  task and skip this check. Keep the phase a light-touch sanity check before launching a runtime session. A system
  whose configuration declares out-of-process tools verifies their host-specific settings through that system's skill.
- **Handoff condition:** All checklist items pass.

### Phase 6: Runtime acquisition (no AI)

- **Plugin / Skill:** none. The experimenter invokes the active system's run CLI, `sle <system> run <mode>`, directly.
  The mode roster belongs to the system. For the current worked example, see `mesoscope:mesoscope-vr-runtime`.
- **Actions:** The run CLI reads the validated system and experiment configuration files, dispatches a
  hardware-deterministic acquisition session, and writes raw data and descriptors into the session directory. For a
  session type in `SESSION_TYPES_USING_VR_TASK`, the Unity Editor must be open before launch. The driver blocks and
  prompts the operator until the editor bridge answers, then opens the scene and enters Play Mode itself
  (`VRTaskDriver._require_bridge`, `VRTaskDriver._activate_scene`, and `VRTaskDriver._arm_unity` in
  `vr_task/driver.py`).
- **Handoff condition:** The session terminates cleanly, and `session_data.yaml` plus the session descriptor that
  `DESCRIPTOR_REGISTRY` maps to the runtime mode's session type exist on disk (`registries.py`).
- **Skill restriction:** You MUST NOT attempt to run sessions through MCP tools. No MCP tool starts a runtime
  session, and there will not be one.

### Phase 7: Post-process and manage

- **Plugin / Skill:** `/data-management` (this plugin)
- **Usually already done at runtime:** A runtime normally preprocesses the session before it exits, so this phase is
  reached only when preprocessing was skipped, when the runtime ended early, or when an animal must be migrated or a
  session deleted. What the runtime offers the operator at shutdown is the active system's decision. For the current
  worked example, see `mesoscope:mesoscope-vr-runtime`.
- **Actions:** Run the system's preprocessing tool to aggregate the raw data and validate the session contents.
  Optionally, run its animal-migration or session-deletion tool. Those are separate tools rather than preprocessing
  steps, and the deletion tool is irreversible. A system's preprocessing may also launch out-of-process tools, whose
  behavior belongs to that system. See `mesoscope:mesoscope-vr-runtime`.
- **Bulk operations:** Build the `session_paths` list a batch preprocessing or deletion pass consumes through
  `assets:session-discovery`.
- **Handoff condition:** The session's `raw_data` directory reached every configured long-term storage destination, and
  the local session directory is removed. A host that configures no storage destinations completes preprocessing with
  the local copy retained, which is how `push_session_data()` in `cross_system/data_preprocessing.py` handles that case.
  `processed_data` is created later, on the processing host, when the session is loaded for processing.

### Phase 8: Handoff to forging (optional)

- **Plugin / Skill:** forging plugin
- **Optional phase:** Forging sits outside the acquisition pipeline proper, because acquisition is complete after
  Phase 7. It frequently runs on entirely separate machine infrastructure, a dedicated processing server or cluster
  operated independently of the acquisition host. Sessions are usually recorded many in a row, so advance to forging
  only once there are no more sessions to record. Otherwise loop back to Phase 6 for the next session.
- **Actions:** Once a session is preprocessed and transferred to long-term storage, generate the project manifest that
  records each session's processing state with `forging:project-manifest`. Then hand off to data-integrity verification
  (`forging:checksum-verification`), batch behavior processing (`forging:behavior-processing`), output verification
  (`forging:behavior-results`), per-session dataset assembly (`forging:dataset-forging`), and dataset composition
  (`forging:dataset-definition`).
- **Handoff condition:** The preprocessed session is present on the storage destination from which the forging plugin
  reads.

---

## Decision tree: which skill to start from

```text
Is the system already configured?
├─ no  → start at Phase 1 (assets:working-directory)
└─ yes
    └─ Is the hardware verified for this session?
        ├─ no  → /system-health-check
        └─ yes
            └─ Does an experiment configuration exist for this project?
                ├─ no  → assets:project-hierarchy → assets:task-templates → assets:experiment-configuration
                └─ yes
                    └─ Is a session already recorded?
                        ├─ no  → the experimenter runs `sle <system> run <mode>` (no AI involvement)
                        └─ yes
                            └─ Is the session preprocessed?  (the runtime normally preprocesses before it exits)
                                ├─ no  → /data-management
                                └─ yes
                                    └─ More sessions to record?
                                        ├─ yes → loop back: the experimenter runs the next session via the run CLI
                                        └─ no  → handoff to forging plugin (optional)
```

---

## Cross-plugin handoffs at a glance

Rows naming a Mesoscope-VR skill illustrate the active system's hand-offs. Another acquisition system substitutes its
own system skill, resolved through `/acquisition-system-setup`'s **Supported acquisition systems** registry. The
`video:`, `communication:`, and `microcontroller:` entries resolve through the ataraxis marketplace.

| You need to…                                      | Use…                                                                               |
|---------------------------------------------------|------------------------------------------------------------------------------------|
| Set the working directory or credentials          | `assets:working-directory`                                                         |
| Author the active system's configuration YAML     | that system's skill (for `mesoscope`, `mesoscope:mesoscope-vr`)                    |
| Author the server (remote transfer) configuration | `forging:server-configuration`                                                     |
| Create a project                                  | `create_project_tool` or `slsa configure project` CLI (`assets:project-hierarchy`) |
| Author a task template                            | `assets:task-templates`                                                            |
| Author a per-project experiment configuration     | `assets:experiment-configuration`                                                  |
| Read a session marker / inspect session metadata  | `assets:session-data`                                                              |
| Build a `session_paths` list for batch work       | `assets:session-discovery`                                                         |
| Read or repair a session descriptor               | `assets:session-descriptors`                                                       |
| Read or repair a session hardware-state snapshot  | `assets:session-hardware-state`                                                    |
| Read or patch a frozen runtime snapshot           | `mesoscope:mesoscope-vr-snapshots`                                                 |
| Look up animal surgery / implants / drugs         | `assets:data-assets`                                                               |
| Inspect, read, or repair a forged dataset         | `assets:datasets`                                                                  |
| Snapshot a project's session processing state     | `forging:project-manifest`                                                         |
| Verify or regenerate data-integrity checksums     | `forging:checksum-verification`                                                    |
| Process behavior data for recorded sessions       | `forging:behavior-processing`                                                      |
| Verify behavior-processing outputs                | `forging:behavior-results`                                                         |
| Assemble a per-session `data.feather`             | `forging:dataset-forging`                                                          |
| Compose or grow a dataset                         | `forging:dataset-definition`                                                       |
| Discover GenICam cameras                          | `video:camera-setup`                                                               |
| Test camera acquisition interactively             | `video:camera-setup`                                                               |
| Verify a camera against its stored GenICam config | `/system-health-check` (verify) / `/acquisition-system-setup` (at bringup)         |
| Dump or restore a camera's GenICam config         | `video:camera-setup` (path sourced from the system configuration)                  |
| Discover microcontrollers / verify MQTT           | `communication:microcontroller-setup`                                              |
| Write a new VideoSystem binding                   | `video:camera-interface` (general) / `mesoscope:mesoscope-vr` (worked example)     |
| Write a new ModuleInterface                       | `/microcontroller-interface` → `communication:microcontroller-interface`           |
| Write firmware for a new module                   | `microcontroller:firmware-module`                                                  |
| Discover or configure Zaber motors                | `/zaber-interface`                                                                 |
| Modify the worked example's hardware composition  | `mesoscope:mesoscope-vr`                                                           |
| Modify the worked example's runtime behavior      | `mesoscope:mesoscope-vr-runtime`                                                   |
| Drive the Unity VR task / MQTT coupling           | `/vr-driver-interface`                                                             |
| Design a new acquisition system (static)          | `/acquisition-system-design`                                                       |
| Implement an acquisition-system runtime loop      | `/acquisition-system-runtime`                                                      |
| Catalogue the seams a new system must touch       | `/library-extension`                                                               |
| Generate / verify a Unity task (prefab + scene)   | `unity:task-prefabs`                                                               |
| Configure the editor-side scene and display rig   | `unity:scene-setup`                                                                |
| Open / inspect a Unity scene                      | `unity:task-scenes`                                                                |
| Enter / exit Unity Play Mode                      | `unity:play-mode`                                                                  |

---

## Related skills

| Skill                            | Relationship                                                                            |
|----------------------------------|-----------------------------------------------------------------------------------------|
| `/system-design-pipeline`        | The build-time counterpart, hands a finished acquisition system to this pipeline        |
| `/library-extension`             | Owns the sollertia-experiment and sollertia-micro-controllers seam catalog for a system |
| `/acquisition-system-setup`      | Resolves the active system to its owning skill and runs the hardware bringup phase      |
| `mesoscope:mesoscope-vr`         | The current worked example's system skill, receiving Phase 2 and Phase 3                |
| `mesoscope:mesoscope-vr-runtime` | The current worked example's run CLI, runtime modes, and session lifecycle              |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Pipeline orchestration:
- [ ] Identified the current phase and the next handoff condition
- [ ] Confirmed the relevant phase-specific skill was invoked, not duplicated inline
- [ ] Did not attempt to run a runtime acquisition session through MCP tools
- [ ] Cross-plugin handoffs to the ataraxis, assets, unity, and forging plugins are explicit
- [ ] Handoff condition for the current phase is satisfied before advancing to the next
```
