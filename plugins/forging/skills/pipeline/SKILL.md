---
name: pipeline
description: >-
  Orchestrates the sollertia-forgery processing lifecycle end to end. Covers phase ordering, handoff conditions, and the
  local versus remote split, from working-directory setup through planning, batch execution, dataset definition,
  forging, and project state. Use when planning a full processing run, when deciding which forging skill to invoke next,
  or when a batch is already mid-flight and its next step is unclear.
user-invocable: false
---

# Sollertia forging pipeline

Entry point and router for the forging plugin. This skill owns none of the 26 `slf mcp` tools, and every tool named
below is invoked through the skill that owns it.

---

## Scope

**Covers:**
- Canonical phase ordering for a sollertia-forgery processing run, with the handoff condition that ends each phase
- The decision tree that routes a request to its starting skill
- The local execution path and the remote scheduler path, and the phases remote work inserts
- Which phases the agent drives and which the scheduler drives
- The cross-plugin handoffs into the assets, experiment, mesoscope, ataraxis, and cindra skill systems

**Does not cover:**
- Any MCP tool signature, parameter, return key, or failure mode. Owned by the skill that owns that tool.
- Preparing, executing, monitoring, cancelling, resetting, or cleaning a batch. Owned by `/batch-processing`.
- Sizing a unit's jobs and reading the plan projection. Owned by `/job-planning`.
- Remote unit discovery, the submission ledger, and the scheduler's own job views. Owned by `/remote-execution`.
- The compute server's five-field configuration. Owned by `/server-configuration`.
- Dataset hierarchy definition, admission, and dataset state. Owned by `/dataset-definition`.
- The forging pipeline's stages, job names, and ordering. Owned by `/dataset-forging`.
- The project manifest and the project job artifact. Owned by `/project-state`.
- What must exist on disk before a job can run. Owned by `/processing-input-format`.
- What a finished pipeline wrote and how to read it. Owned by `/processing-results`.
- MCP server connectivity and the plugin-wide response contract. Owned by `/forging-mcp-environment-setup`.
- The `slf` command surface and its divergence from the MCP path. Owned by `/cli-reference`.
- Where a concern belongs in the library and how a registry seam is filled. Owned by `/data-processing-design`.
- Acquisition-system-specific file names, columns, and session types. Owned by the `mesoscope:mesoscope-vr-*` skills.

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the relevant skill for
detailed tool usage, parameter reference, and troubleshooting.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## The local and remote split

Choose the host before planning, because every unit path, every artifact read, and every monitoring surface changes
with it. A batch runs where it was prepared, so the choice is not revisable after Phase 4.

| Concern                        | Local path                                                             | Remote path                                       |
|--------------------------------|------------------------------------------------------------------------|---------------------------------------------------|
| Who runs the jobs              | one process pool the MCP server owns, agent-driven                     | the SLURM scheduler, scheduler-driven             |
| Lifetime of a running batch    | dies with the MCP server process                                       | outlives it, held by the submission ledger        |
| Concurrent batches             | one run at a time, over any number of batches, a second run is refused | unbounded, the ledger tracks many at once         |
| Unit paths named to a tool     | paths on this machine                                                  | paths on the server, under its configured root    |
| Where a read tool reads        | the project directory itself                                           | a mirror of it under the working directory        |
| Refreshing what a read returns | the generate tools rewrite it in place                                 | generate on the server first, a read never does   |
| Closing a finished batch       | the manager thread, once the pool drains                               | a remote status read, cancellation, or submission |

Never round-trip a path out of a read response back into a write tool on the remote path. A remote read reports the
local mirror, so take server paths from `discover_remote_project_tool` or from a generate or plan tool's own response.
The mechanics of each path are owned by `/batch-processing` and `/remote-execution`.

---

## Pipeline phases

```text
 1.  Working directory + credentials   assets:working-directory
 2.  Session discovery                 assets:session-discovery
 R1. Server configuration              /server-configuration      (remote only, precedes phase 3)
 R2. Remote unit discovery             /remote-execution          (remote only, replaces phase 2)
 3.  Plan                              /job-planning
 4.  Prepare                           /batch-processing
 5.  Execute                           /batch-processing          (+ /remote-execution on the remote path)
 6.  Monitor                           /batch-processing          (+ /remote-execution on the remote path)
 7.  Close                             none. closure carries no tool of its own
 8.  Verify                            /processing-results
 9.  Define a dataset                  /dataset-definition
10.  Forge                             /dataset-forging           (mechanics: /batch-processing)
11.  Read project state                /project-state
```

Phases 3 through 8 repeat once per pipeline. Six pipelines are dispatchable as batches, so a project is normally
carried through them in several passes before a dataset is defined.

### Phase 1: Working directory and credentials

- **Plugin / Skill:** assets plugin, `assets:working-directory`
- **Actions:** Set the local Sollertia working directory. Every durable forging record resolves under it, including the
  prepared-batch registry at `remote_state/prepared_batches`, the submission ledger at
  `remote_state/submission_ledger.yaml`, each project's remote mirror at `remote_state/<project>`, and the server
  configuration at `configuration/server_configuration.yaml`. On a macOS host, also confirm that the OpenMP runtime
  loaded by the Numba threading layer has been linked, since a host that has not linked it fails every parallel job
  while the MCP server still reports healthy. That diagnostic is owned by `/forging-mcp-environment-setup`.
- **Handoff condition:** `list_prepared_batches_tool` answers with a `batch_directory` path, which proves the working
  directory resolves for this process.
- **Skip condition:** The host already carries a configured working directory for this account.
- **Cross-plugin handoffs:**
  - `assets:working-directory` for the working directory and the data root
  - `assets:project-hierarchy` when the project directory the units live under does not exist yet

### Phase 2: Session discovery

- **Plugin / Skill:** assets plugin, `assets:session-discovery`
- **Actions:** Build the list of processing-unit roots the batch tools consume. `assets:session-discovery` is the
  exclusive producer of that list. Every unit of one batch must belong to one project, because the plan and state
  artifacts that resolve a batch are written per project.
- **Handoff condition:** A non-empty list of unit roots under a single project root, each holding the data the target
  pipeline consumes.
- **Skip condition:** The units are already named by an earlier read, such as `read_project_jobs_tool`'s listing.
- **Cross-plugin handoffs:**
  - `assets:session-discovery` to build the list
  - `experiment:data-management` when a session has not been preprocessed and therefore carries no `raw_data`
  - `/processing-input-format` when a discovered unit is rejected for missing inputs

### Phase R1: Server configuration (remote only)

- **Plugin / Skill:** `/server-configuration`
- **Actions:** Author the flat five-field configuration, `username`, `password`, `host`, `root`, and `environment`.
  Every call that names `host="remote"` opens its connection from it, and every server-side invocation activates the
  named environment before running `slf`.
- **Handoff condition:** `read_server_configuration_tool` returns a payload whose five fields are all populated. A
  file with any field left empty is reported as an error rather than as an empty payload.
- **Skip condition:** Every phase of this run stays on the local host.

### Phase R2: Remote unit discovery (remote only)

- **Plugin / Skill:** `/remote-execution`
- **Actions:** Enumerate the project's sessions and datasets as they exist on the server with
  `discover_remote_project_tool`, and take the unit paths from its listing. This replaces Phase 2, because
  `assets:session-discovery` reads this machine rather than the server.
- **Handoff condition:** A non-empty list of server unit paths under one project, ready to be named to a planning or
  preparation call that carries `host="remote"`.
- **Skip condition:** The run stays local, or the server paths are already known from a generate or plan response.

### Phase 3: Plan

- **Plugin / Skill:** `/job-planning`
- **Actions:** Estimate each unit's jobs with `plan_session_jobs_tool` or `plan_dataset_jobs_tool`, then read the
  project projection with `read_project_plan_tool`. Planning is the expensive half of the lifecycle, because sizing
  opens each unit's acquired data, which is why it is a call of its own. `read_resource_model_tool` reports the
  declared width and the concurrency terms of every job type without touching disk, and `inspect_job_resources_tool`
  reports what one specific batch would occupy.
- **Handoff condition:** The projection carries a row for every unit and every job the batch will dispatch, and each
  sizing refusal reported against a unit has been read and accepted.
- **Skip condition:** Preparation performs the same planning pass itself, so a first run may enter at Phase 4
  directly. Plan separately when the cost has to be known before the work is committed.

### Phase 4: Prepare

- **Plugin / Skill:** `/batch-processing`
- **Actions:** Call `prepare_batch_tool` with one pipeline and the unit roots from Phase 2 or Phase R2. The six batch
  pipelines are `checksum`, `runtime`, `microcontroller`, `video`, `two_photon`, and `forging`. The first five take
  session roots, and `forging` takes dataset roots. Preparation plans each unit, creates and aligns its processing
  trackers, rewrites the project's plan and state artifacts, and records a durable batch document that returns a
  `batch_id`. Jobs already recorded as succeeded are omitted, and a job whose prerequisite can neither be queued in
  this batch nor confirmed as already succeeded is reported blocked rather than dispatched.
- **Handoff condition:** The response carries a `batch_id` and a `total_jobs` count above zero. A successful response
  reporting `total_jobs: 0` means every job already succeeded or every job is blocked, and executing it does nothing.
- **Skip condition:** None. Execution reads its descriptors from a recorded batch, so it cannot run without one.
- **Recovery:** A `batch_id` is a durable record under the working directory and outlives the MCP server process.
  `list_prepared_batches_tool` is the only path back to a lost identifier.

### Phase 5: Execute

- **Plugin / Skill:** `/batch-processing`, and `/remote-execution` for the scheduler path
- **Actions:** Call `execute_jobs_tool` with the recorded identifiers. It takes no host argument, because a batch runs
  where it was prepared. A local run dispatches onto one process pool the MCP server process owns and honors a core
  budget and a memory budget. A remote run submits one scheduler allocation per job, chains each dependent onto its
  prerequisite allocations, and honors a wall-time request instead.
- **Handoff condition:** The response reports that dispatch started. That proves the pool was launched or the
  scheduler accepted the allocations, and it says nothing about job outcomes.
- **Skip condition:** None.
- **Who drives it:** Local execution is agent-driven, so the agent's own MCP server process holds the run and loses it
  on restart. Remote execution is scheduler-driven, so the jobs continue independently of this session.

### Phase 6: Monitor

- **Plugin / Skill:** `/batch-processing`, and `/remote-execution` for the scheduler's own view
- **Actions:** Poll `get_processing_status_tool`. No tool on this server blocks or waits, so progress is discovered
  only by polling. There is no timing tool. Timing is `detailed=True` on the status and job read tools, which adds
  `started_at` and `completed_at`. On the remote path, `read_scheduler_jobs_tool` answers from the scheduler itself,
  so it also covers an allocation this machine did not submit and one that has already settled.
- **Handoff condition:** Locally, the run reports itself inactive and carries either a recorded outcome or a
  tracker-derived summary. Remotely, every covered allocation has reached a terminal scheduler state.
- **Skip condition:** None while any dispatched job is outstanding.
- **Cancellation:** `cancel_processing_tool` ends a run early. Local cancellation is cooperative and remote
  cancellation kills queued and running allocations alike, both owned by `/batch-processing`.

### Phase 7: Close

- **Plugin / Skill:** none. Closure carries no tool of its own.
- **Actions:** Nothing to invoke. A local batch closes itself through `orchestration/closure.py`'s `close_batch` once
  its pool drains. A remote batch closes through `close_settled_batches`, triggered as a side effect by a remote
  status read, a remote cancellation, and a further remote submission.
- **Handoff condition:** `list_prepared_batches_tool` reports the batch with its outcome recorded.
- **Skip condition:** None. Closure is what makes a finished batch answerable, because it writes the outcome record
  that survives the process and retires the prepared document behind it.
- **One-way transition:** Closure deletes the prepared document, so a closed identifier cannot be executed again.
  Re-running the work means preparing a new batch, which then queues only what is still outstanding.

### Phase 8: Verify

- **Plugin / Skill:** `/processing-results`
- **Actions:** There is no output-verification tool and no feather-query tool on this server. Verification runs
  through the totals and breakdowns of `read_project_jobs_tool`, `get_processing_status_tool`, and
  `read_dataset_state_tool`, each read through the skill that owns it. `animal` is a `String` column in every project
  artifact, so the manifest, jobs, plan, and dataset-state tables join on animal and session without casting. Read
  each failure's recorded message, then return the affected jobs to the scheduled state and prepare a fresh batch.
- **Handoff condition:** Every job the batch dispatched records success, or each failure has an understood cause and
  an accepted disposition.
- **Skip condition:** None. A dispatched batch is not finished until its records have been read.
- **Cross-plugin handoffs:**
  - `mesoscope:mesoscope-vr-processing-schema` for the Mesoscope-VR file name and column rosters, and the per-pipeline
    `mesoscope:mesoscope-vr-*` skill named in its related-skills table for what a given stage computes
  - `video:log-processing` and `communication:log-processing` for the upstream camera and microcontroller stages
  - `cindra:single-recording-processing` for the single-recording two-photon stages

### Phase 9: Define a dataset

- **Plugin / Skill:** `/dataset-definition`
- **Actions:** Build the dataset hierarchy under the project root with `define_forging_dataset_tool`, then snapshot it
  with `generate_dataset_state_tool` and read it with `read_dataset_state_tool`. Admission refuses a session whose
  acquisition system requires a pipeline the session has not completed, which is why Phase 8 precedes this one. An
  animal the dataset already holds is frozen against widening, and rebuilding it is an explicit request.
- **Handoff condition:** The response reports the session and animal counts the dataset is meant to hold, and
  `list_project_datasets_tool` lists the dataset under its project.
- **Skip condition:** The hierarchy already covers every session to be forged.
- **Cross-plugin handoffs:**
  - `assets:datasets` to inspect, read, or repair a dataset once it exists, and for dataset description authoring
  - `assets:data-assets` for the animal records a dataset carries alongside its data

### Phase 10: Forge

- **Plugin / Skill:** `/dataset-forging` for the pipeline, `/batch-processing` for the mechanics
- **Actions:** Run Phases 3 through 8 again with the pipeline set to `forging` and dataset roots in place of session
  roots. The pipeline's own stages, their ordering, and the per-animal versus per-session scope of each are owned by
  `/dataset-forging`.
- **Handoff condition:** `read_dataset_state_tool` records success for every job the dataset tracks.
- **Skip condition:** No dataset is wanted from this project.
- **Cross-plugin handoffs:**
  - `mesoscope:mesoscope-vr-dataset-assembly` for what the Mesoscope-VR assembler writes into a session's row set

### Phase 11: Read project state

- **Plugin / Skill:** `/project-state`
- **Actions:** Write the project's state artifacts with `generate_project_manifest_tool`, which publishes the job
  artifact first and the manifest second, then read them with `read_project_manifest_tool`, `read_project_jobs_tool`,
  and `get_manifest_status_tool`. `manifest` is a pipeline identity but not a batch pipeline, so it never passes
  through preparation and execution and is generated by its own tool instead.
- **Handoff condition:** The manifest reports every session in the project with each pipeline's recorded state.
- **Skip condition:** Preparation rewrote these artifacts in Phase 4, so they are already current when nothing has run
  since. Regenerate when the project changed outside this session.
- **Also the survey entry:** Enter here first when the request names a project rather than a unit list, because the
  manifest is what says which sessions still owe which pipeline.

---

## Decision tree: which skill to start from

```text
Is the Sollertia working directory configured on this host?
├─ no  → start at Phase 1 (assets:working-directory)
└─ yes
    └─ Will the jobs run on a remote compute server?
        ├─ yes
        │   └─ Is the server configuration populated in all five fields?
        │       ├─ no  → /server-configuration
        │       └─ yes → /remote-execution for the server unit paths, then rejoin at Phase 3
        └─ no
            └─ Is it known which units to process?
                ├─ no  → assets:session-discovery, or /project-state to survey what the project still owes
                └─ yes
                    └─ Is a batch already prepared for them?
                        ├─ no      → /job-planning to size the work, then /batch-processing to prepare
                        ├─ unknown → /batch-processing, whose listing recovers every recorded identifier
                        └─ yes
                            └─ Has it been executed?
                                ├─ no  → /batch-processing to execute
                                └─ yes
                                    └─ Has every dispatched job succeeded?
                                        ├─ no  → /processing-results to read the failures, then re-prepare
                                        └─ yes
                                            └─ Is a dataset wanted from this project?
                                                ├─ no  → /project-state to report where the project stands
                                                └─ yes → /dataset-definition, then /dataset-forging
```

---

## Cross-plugin handoffs at a glance

The `video:`, `communication:`, and `cindra:` entries below resolve through the ataraxis and cindra marketplaces.
Every other entry resolves inside the sollertia marketplace.

| You need to...                                           | Use...                                          |
|----------------------------------------------------------|-------------------------------------------------|
| Set the working directory or the data root               | `assets:working-directory`                      |
| Create the project directory the units live under        | `assets:project-hierarchy`                      |
| Build the unit-path list a local batch consumes          | `assets:session-discovery`                      |
| Name the units of a batch that lives on the server       | `/remote-execution`                             |
| Preprocess a session so it carries `raw_data`            | `experiment:data-management`                    |
| Read a session marker or inspect session metadata        | `assets:session-data`                           |
| Author the compute server's credentials and root         | `/server-configuration`                         |
| See what a batch will occupy before running it           | `/job-planning`                                 |
| Prepare, execute, monitor, cancel, reset, or clean       | `/batch-processing`                             |
| Recover a lost batch identifier                          | `/batch-processing`                             |
| Read the scheduler's own view of an allocation           | `/remote-execution`                             |
| Learn what must exist on disk before a job can run       | `/processing-input-format`                      |
| Interpret a pipeline's outputs and its failures          | `/processing-results`                           |
| Snapshot or read a project's per-session state           | `/project-state`                                |
| Compose, grow, or rebuild a dataset hierarchy            | `/dataset-definition`                           |
| Run the dataset assembly pipeline                        | `/dataset-forging`                              |
| Inspect, read, or repair an already forged dataset       | `assets:datasets`                               |
| Look up an animal's surgery, implant, or drug records    | `assets:data-assets`                            |
| Look up a Mesoscope-VR processed file name or column     | `mesoscope:mesoscope-vr-processing-schema`      |
| Learn what the Mesoscope-VR assembler writes per session | `mesoscope:mesoscope-vr-dataset-assembly`       |
| Learn what a Mesoscope-VR module parser computes         | `mesoscope:mesoscope-vr-module-parsing`         |
| Learn how Mesoscope-VR cue sequences become trials       | `mesoscope:mesoscope-vr-trial-decomposition`    |
| Learn what the Mesoscope-VR pupil-tracking pass computes | `mesoscope:mesoscope-vr-video-tracking`         |
| Resolve a Mesoscope-VR session's cindra configuration    | `mesoscope:mesoscope-vr-imaging-configuration`  |
| Learn how Mesoscope-VR fluorescence frames are aligned   | `mesoscope:mesoscope-vr-fluorescence-alignment` |
| Understand an upstream camera log archive                | `video:log-processing`                          |
| Understand an upstream microcontroller log archive       | `communication:log-processing`                  |
| Understand the single-recording two-photon stages        | `cindra:single-recording-processing`            |
| Add a pipeline, a stage, or a registry donation          | `/library-extension`                            |
| Decide where a concern belongs in this library           | `/data-processing-design`                       |
| Run a command by hand while the MCP server is down       | `/cli-reference`                                |
| Restore the `slf mcp` connection                         | `/forging-mcp-environment-setup`                |

---

## Related skills

| Skill                            | Relationship                                                                   |
|----------------------------------|--------------------------------------------------------------------------------|
| `/batch-processing`              | Runs Phases 4 through 6 and owns the eight batch and orchestration tools       |
| `/job-planning`                  | Runs Phase 3 and owns the six planning and resource tools                      |
| `/remote-execution`              | Runs Phase R2 and the scheduler half of Phases 5 and 6                         |
| `/server-configuration`          | Runs Phase R1, the precondition of every remote call                           |
| `/dataset-definition`            | Runs Phase 9 and owns the four dataset tools                                   |
| `/dataset-forging`               | Runs Phase 10 by routing the forging pipeline through `/batch-processing`      |
| `/project-state`                 | Runs Phase 11 and owns the four project management tools                       |
| `/processing-results`            | Runs Phase 8 by reading the breakdowns the tool-owning skills expose           |
| `/forging-mcp-environment-setup` | Restores the server behind this whole pipeline, and owns the response contract |
| `experiment:pipeline`            | The acquisition lifecycle that hands preprocessed sessions to Phase 2          |

---

## Proactive behavior

- Name the current phase and the skill that owns it before invoking any tool, so the user can see where the work
  stands and what ends the phase.
- Invoke the phase-specific skill rather than reproducing its tool calls here, because this skill carries no tool
  signatures and its summaries are deliberately shorter than the owning skill's.
- Route a request that names a pipeline but no units back to Phase 2 or Phase R2 before planning anything.
- Route a status read that reports nothing running and no outcome to `/batch-processing` for identifier recovery,
  rather than preparing a second batch over the same work.
- Settle the local versus remote choice before Phase 3, since a batch runs where it was prepared.

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Pipeline routing:
- [ ] sollertia-forgery MCP server is connected
- [ ] Named the current phase and the skill that owns it before calling any tool
- [ ] Confirmed the current phase's handoff condition before advancing to the next phase
- [ ] Invoked the phase-specific skill rather than reproducing its tool calls inline
- [ ] Chose the local or the remote path before Phase 3 and kept every unit path on that host
- [ ] Did not report a batch as finished from the dispatch flag execute_jobs_tool returns
- [ ] Recovered a lost batch identifier through the prepared-batch listing rather than re-preparing
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
