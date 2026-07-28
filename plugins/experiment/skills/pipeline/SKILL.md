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
- Designing or building a new acquisition system type — see `/system-design-pipeline`
- Detailed tool usage for any individual phase (see phase-specific skills)
- MCP server connectivity (see each plugin's `*-mcp-environment-setup` skill, e.g.
  `/experiment-mcp-environment-setup`, `assets:assets-mcp-environment-setup`)
- Authoring system, experiment, or session YAML files. The active system's skill authors the system configuration
  through the `sle mcp` write tool, and the assets plugin skills author experiment and session files
- Post-acquisition data processing (see forging plugin)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the
relevant skill for detailed tool usage, parameter reference, and troubleshooting.

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: AI assistance operates at configuration time;
runtime acquisition is fully deterministic and AI-independent.

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
| Post-acquisition data processing      | yes          | forging plugin (optional; separate infra)  |

The MCP tool surface intentionally has no "start a recording session" tool. Runtime is launched only
through the active system's run CLI (for the `mesoscope` system, `sle mesoscope run <mode>`), which
reads validated configuration files written during the AI-assisted phases.

---

## Pipeline phases

```text
1. Working directory      assets:working-directory
2. System configuration   active system's skill (mesoscope → mesoscope:mesoscope-vr)
3. Hardware bringup        /acquisition-system-setup
4. Experiment design       assets:project-hierarchy → assets:task-templates → assets:experiment-configuration
5. Pre-session check       /system-health-check
6. Runtime acquisition     active system run CLI, e.g. sle mesoscope run <mode>  (no MCP, no AI)
7. Post-process & manage   /data-management
8. Handoff to forging      forging plugin  (optional; often separate infrastructure)
```

### Phase 1: Working directory and credentials

- **Plugin / Skill:** assets plugin → `assets:working-directory`
- **Actions:** Set the local Sollertia working directory (always required). Set the task templates directory
  for systems that run experiment sessions (every experiment seeds its configuration from a corridor task
  template; training and window-checking sessions need no template). Optionally configure platform credentials
  by category — needed only for systems that integrate with the corresponding external service (e.g. the
  `mesoscope` system uses `google` credentials for Google Sheets animal metadata).
- **Handoff condition:** `get_platform_environment_status_tool` reports `overall_ok` — the working directory
  is the only required component that gates it. The data root, the task templates directory, and credentials
  are reported as separate optional components; configure the data root (and, for experiment systems, the
  templates directory) here too so the host can record and run sessions.
- **Skip condition:** The platform data root is already initialized for this host.

### Phase 2: System configuration

- **Plugin / Skill:** the active acquisition system's skill (for the `mesoscope` system,
  mesoscope plugin → `mesoscope:mesoscope-vr`). Resolve the system type to its owning skill via
  `/acquisition-system-setup`'s **Supported acquisition systems** registry.
- **Actions:** Generate or edit the active system's configuration YAML via the `sle mcp` write
  tool (e.g. for the `mesoscope` system, `MesoscopeSystemConfiguration`).
- **Handoff condition:** `read_system_configuration_tool` returns a valid configuration; the config
  passes schema validation.

### Phase 3: Hardware bringup

- **Plugin / Skill:** `/acquisition-system-setup` (this plugin)
- **Actions:** Discover the active system's hardware — cameras, microcontrollers, and Zaber motors (the
  platform-universal / domain-general stack), plus any system-specific instrument it composes (for the
  `mesoscope` system, the mesoscope itself, controlled via the ScanImage bridge). Validate the system
  configuration against discovered hardware. Update fields (camera indices, port assignments) with the
  `sle mcp` write tool (`write_system_configuration_tool`) as needed. Where a device has cached configuration
  in the system configuration directory, apply it as part of bringup, e.g. restore each camera's GenICam
  configuration from its cached file via `video:camera-setup` (ataraxis marketplace), with the path sourced
  from the system configuration.
- **Handoff condition:** All required hardware enumerated; system configuration matches reality.
- **Cross-plugin handoffs:**
  - `video:camera-setup` for camera discovery
  - `communication:microcontroller-setup` for microcontroller enumeration
  - `/zaber-interface` for Zaber motor discovery and validation (domain-general, not system-specific)
  - for any system-specific instrument the active system composes, that system's skill (for the
    `mesoscope` system: the mesoscope via the ScanImage bridge; see `mesoscope:mesoscope-vr`)

### Phase 4: Experiment authoring

This phase spans three assets plugin skills, each owning exactly one slsa asset. Step 4a (project),
Step 4b (task template), and Step 4c (experiment configuration) are all required: every experiment
seeds its configuration from a corridor task template.

- **Step 4a — `assets:project-hierarchy`:** Confirm the project under which the
  experiment will live exists on disk (`get_data_root_overview_tool`), or create it with
  `create_project_tool` (equivalently the `slsa configure project` CLI). Projects must exist before a
  session can be recorded — `SessionData.create` raises `FileNotFoundError` if the project is missing.
  Both the discovery tool and `create_project_tool` are owned by `assets:project-hierarchy`.
- **Step 4b — `assets:task-templates`:** A task template (`TaskTemplate`) is the corridor
  task asset: cue catalog, VR environment, and per-trial corridor geometry (each trial owns its own
  geometry — there is no separate segment catalog at the template level). Author or load the template
  (owns `write_template_tool`), then hand off to `unity:task-prefabs` if it targets a
  Unity scene (prefab generation and zone validation).
- **Step 4c — `assets:experiment-configuration`:** Author the per-project experiment
  configuration — trial structures, the experiment state machine, and runtime parameters (state
  durations, reward volumes). `create_experiment_from_vr_template_tool` seeds the configuration from a
  Unity VR task template (the template is read only at creation time and not stored in the result);
  `write_experiment_configuration_tool` authors or repairs the full payload directly.
- **Handoff condition:** `read_experiment_configuration_tool` returns a validated experiment for the
  target project.

### Phase 5: Pre-session health check

- **Plugin / Skill:** `/system-health-check` (this plugin)
- **Actions:** Verify network mounts, hardware connectivity, animal metadata in Google Sheets (**optional**, only 
  for systems that use them), project readiness. For an experiment session that runs the corridor task (e.g.
  Mesoscope-VR's `experiment` mode), also confirm the Unity Editor MCP Bridge is reachable
  (`check_unity_bridge_tool` / `sle get unity`) so the run CLI can open the scene and arm the VR task; training
  and window-checking sessions skip this check. Light-touch sanity check before launching a runtime session.
  `validate_system_configuration_tool` and `check_system_mounts_tool` report on the filesystem mounts, so verify
  the `video_tracking` section by hand whenever face-camera inference is configured. A wrong `dlc_project_path`
  passes every pre-flight check and surfaces only when Phase 7 joins the inference subprocess.
- **Handoff condition:** All checklist items pass.

### Phase 6: Runtime acquisition (no AI)

- **Plugin / Skill:** none — invoked directly via the active system's run CLI by the experimenter
  (for the `mesoscope` system, `sle mesoscope run <mode>`, where `<mode>` is one of `window-checking`,
  `lick-training`, `run-training`, or `experiment`).
- **Actions:** The run CLI reads the validated system + experiment configuration files, dispatches a
  hardware-deterministic acquisition session, writes raw data + descriptors into the session directory.
  For an experiment session that runs the corridor task, the Unity Editor must be open before launch —
  the run CLI drives scene activation and Play Mode through the editor MCP Bridge and blocks until the
  bridge is reachable. Training and window-checking sessions run no task, drive no Unity, and need no
  editor bridge.
- **Handoff condition:** Session terminates cleanly; `session_data.yaml` and the appropriate session
  descriptor for the runtime mode exist on disk (for the `mesoscope` system: lick training /
  run training / window checking / mesoscope experiment).
- **Skill restriction:** You MUST NOT attempt to run sessions through MCP tools. There is no
  MCP tool that starts a runtime session, and there will not be one.

### Phase 7: Post-process and manage

- **Plugin / Skill:** `/data-management` (this plugin)
- **Usually already done at runtime:** sessions normally preprocess at the end of Phase 6. The data
  modes (lick-training, run-training, experiment) prompt the experimenter at shutdown to choose
  `preprocess` / `skip preprocessing` / `purge session`, with `preprocess` as the default, recommended
  action (driven by the Mesoscope-VR system class); window-checking preprocesses automatically in its
  runtime function. Reach for this phase only when the experimenter chose `skip preprocessing` (or the
  runtime ended before the prompt), or to migrate an animal between projects or delete a session.
- **Actions:** Run `preprocess_session_tool` to aggregate raw data and validate session contents.
  Optionally, run `migrate_animal_tool` to migrate the animal between projects or `delete_session_tool`
  to delete the session — these are separate tools, not part of preprocessing.
- **Face-camera inference sub-step:** For experiment sessions (`SessionTypes.MESOSCOPE_EXPERIMENT`),
  preprocessing launches DeepLabCut face-camera eye-tracking as a background
  `conda run -n <conda_environment> slvt infer` subprocess, right after the session videos are renamed. It
  therefore runs on the rig's otherwise-idle GPU while the CPU-bound and disk-bound stages proceed. The step
  is opt-in through the `video_tracking` section of `MesoscopeSystemConfiguration`, which gates it on
  `conda_environment` and `dlc_project_path`. Leaving either unset skips inference, as does a missing
  face-camera video, which skips with a warning. The subprocess boundary is load-bearing, because
  sollertia-video-tracking requires Python 3.12 and numpy 1.x for DeepLabCut 3.0.0, while the acquisition
  stack runs Python 3.14 and numpy 2. sollertia-video-tracking ships no MCP server and no plugin, so the
  `slvt` CLI is its whole agent-facing surface.
- **Face-camera inference outcome:** The subprocess exits zero and writes at least one DeepLabCut `.h5`
  prediction beside the face-camera video in `raw_data/camera_data/`, where the raw-data checksum and the
  storage transfer then capture it. A non-zero exit status or zero predictions raises `RuntimeError`, aborts
  the transfer to long-term storage, and retains the local session copy for a manual retry. The transient log
  at `<tmp>/slvt_infer_<session_name>.log` is retained on failure, with its last 2000 characters echoed into
  the error.
- **Handoff condition:** The session's `raw_data` directory reached every configured long-term storage
  destination and the local session copy is removed. A host that configures no storage destinations completes
  preprocessing with the local copy retained, which satisfies this phase as well. `processed_data` is created
  later, on the processing host, when the session is loaded for processing.

### Phase 8: Handoff to forging (optional)

- **Plugin / Skill:** forging plugin
- **Optional phase:** Forging sits outside the acquisition pipeline proper — acquisition is complete
  after Phase 7. It frequently runs on entirely separate machine infrastructure (a dedicated
  processing server or cluster) operated independently of the acquisition host. Sessions are usually
  recorded many in a row, so advance to forging only once there are no more sessions to record —
  otherwise loop back to Phase 6 for the next session.
- **Actions:** Once a session is preprocessed and transferred to long-term storage, hand off to the
  forging plugin's behavior processing subsystem — batch behavior processing
  (`forging:behavior-processing`), output verification (`forging:behavior-results`), and dataset
  curation (`forging:datasets`).
- **Handoff condition:** The preprocessed session is present on the storage destination the forging
  plugin reads from.

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
                        ├─ no  → user runs the active system's run CLI, e.g. `sle mesoscope run <mode>` (no AI involvement)
                        └─ yes
                            └─ Is the session preprocessed?  (runtime prompts at session end, default preprocess)
                                ├─ no  → /data-management
                                └─ yes
                                    └─ More sessions to record?
                                        ├─ yes → loop back: user runs the next session via the run CLI (preferred)
                                        └─ no  → handoff to forging plugin (optional)
```

---

## Cross-plugin handoffs at a glance

Rows naming a Mesoscope-VR skill illustrate the active system's hand-offs; another acquisition system
substitutes its own system skill (resolved via `/acquisition-system-setup`'s **Supported acquisition
systems** registry).

| You need to…                                      | Use…                                                                               |
|---------------------------------------------------|------------------------------------------------------------------------------------|
| Set the working directory or credentials          | `assets:working-directory`                                                         |
| Author the active system's configuration YAML     | that system's skill (for `mesoscope`, `mesoscope:mesoscope-vr`)                    |
| Author the server (remote transfer) configuration | `forging:server-configuration`                                                     |
| Create a project                                  | `create_project_tool` or `slsa configure project` CLI (`assets:project-hierarchy`) |
| Author a task template                            | `assets:task-templates`                                                            |
| Author a per-project experiment configuration     | `assets:experiment-configuration`                                                  |
| Read a session marker / inspect session metadata  | `assets:session-data`                                                              |
| Read or repair a session descriptor               | `assets:session-descriptors`                                                       |
| Read or patch a frozen runtime snapshot           | `mesoscope:mesoscope-vr-snapshots`                                                 |
| Look up animal surgery / implants / drugs         | `assets:data-assets`                                                               |
| Curate or read a dataset                          | `forging:datasets`                                                                 |
| Discover GenICam cameras                          | `video:camera-setup`                                                               |
| Test camera acquisition interactively             | `video:camera-setup`                                                               |
| Verify a camera against its stored GenICam config | `/system-health-check` (verify) / `/acquisition-system-setup` (at bringup)         |
| Dump or restore a camera's GenICam config         | `video:camera-setup` (path sourced from the system configuration)                  |
| Discover microcontrollers / verify MQTT           | `communication:microcontroller-setup`                                              |
| Write a new VideoSystem binding                   | `video:camera-interface` (general) / `mesoscope:mesoscope-vr` (Mesoscope-specific) |
| Write a new ModuleInterface                       | `/microcontroller-interface` → `communication:microcontroller-interface`           |
| Write firmware for a new module                   | `microcontroller:firmware-module`                                                  |
| Discover or configure Zaber motors                | `/zaber-interface`                                                                 |
| Modify Mesoscope-VR hardware composition          | `mesoscope:mesoscope-vr`                                                           |
| Modify Mesoscope-VR runtime behavior              | `mesoscope:mesoscope-vr-runtime`                                                   |
| Drive the Unity VR task / MQTT coupling           | `/vr-driver-interface`                                                             |
| Design a new acquisition system (static)          | `/acquisition-system-design`                                                       |
| Implement an acquisition-system runtime loop      | `/acquisition-system-runtime`                                                      |
| Generate / verify Unity task prefab from template | `unity:task-prefabs`                                                               |
| Open / create a Unity scene                       | `unity:task-scenes`                                                                |
| Enter / exit Unity Play Mode                      | `unity:play-mode`                                                                  |

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
