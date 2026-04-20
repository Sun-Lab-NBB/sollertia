---
name: project-hierarchy
description: >-
  Discovers and creates entries in the Sollertia project hierarchy (projects, animals, experiments,
  subjects, sessions) via the slsa MCP server. Owns create_project_tool. Covers project bootstrap,
  hierarchy traversal, and the relationship between projects, animals, sessions, and experiment
  configurations. Use when bootstrapping a new project, enumerating animals or sessions under a project,
  or building tooling that needs to walk the project tree.
user-invocable: true
---

# Sollertia project hierarchy

Discovers and creates entries in the Sollertia project hierarchy using the `slsa mcp` MCP server.
This skill is the **exclusive** owner of `create_project_tool` — no other skill in the marketplace may
call it.

---

## Scope

**Covers:**
- Creating new projects (`create_project_tool`)
- Discovering projects, animals, experiments, subjects, and sessions
- The directory layout of a Sollertia project tree
- The relationship between projects, animals, sessions, experiments, and subjects

**Does not cover:**
- Authoring per-project `MesoscopeExperimentConfiguration` (see `/experiment-configuration`)
- Reading or writing `SessionData` (see `/session-data`)
- Reading or writing session descriptors (see `/session-descriptors`)
- Reading subject metadata (see `/subject-metadata`)
- Reading or writing datasets (see forging plugin's `/datasets`)
- Initial working directory setup (see `/working-directory`)

The `discover_*` tools below are read-only and may be called as **natural shares** by other skills that
need to enumerate the hierarchy. Only `create_project_tool` is exclusive to this skill.

---

## What is a project

A **project** is the top-level scientific grouping of acquired data: a logical container for
**all sessions belonging to one research investigation**, organized by animal. Every session,
every experiment configuration, and every per-animal calibration artifact is anchored to exactly
one project. Project names are arbitrary strings chosen by the experimenter (commonly the project
abbreviation used in templates, e.g. `MaalstroomicFlow`, `StateSpaceOdyssey`); the only
slsa-side constraint is that the project name maps to a directory created at the data root by
`create_project_tool`.

A project bundles three kinds of state:

1. **Shared configuration** — per-project experiment YAMLs under `configuration/`, authored by
   `/experiment-configuration` once and reused across many sessions and animals.
2. **Animal subtrees** — one directory per participating animal, each containing that animal's
   sessions (and on the acquisition machine, animal-level cross-session calibration state).
3. **Per-session subtrees** — timestamped session directories owned by `/session-data` and the
   sibling session-* skills. See `/session-data` for what a session is and how it is named.

### Why the structure is shaped this way

- **Project at the top** captures the unit of scientific ownership. Multiple animals contribute
  to one project; session-level analysis joins across animals within a project. Datasets are
  also project-scoped — each dataset belongs to exactly one project and aggregates sessions
  across animals within that project (see forging plugin's `/datasets`) — they are a downstream
  aggregation, not a primary hierarchy.
- **Animals nested under project, not the inverse.** Each animal belongs to exactly one project
  at a time (enforced by the acquisition runtime — see the next section). Inverting the
  hierarchy ("animal at top, projects under it") would require duplicating animal metadata per
  project and would not match how the runtime resolves session paths.
- **`configuration/` is per-project**, not per-animal or per-session, because experiment
  paradigms are a project-scope decision. The frozen per-session snapshot lives inside each
  session's `raw_data/experiment_configuration.yaml` (copied by `SessionData.create`); the
  master copy under `configuration/` is what new sessions are instantiated from.
- **Sessions are flat under each animal** — no `experiments/<exp-name>/<session>/` grouping —
  because the timestamped session name already gives chronological ordering and each session's
  marker file (`SessionData.experiment_name`) names the experiment used. Forcing experiment
  grouping into the directory structure would prevent training and window-checking sessions
  (which have no experiment) from sharing a sibling slot.

### How a project is represented across destinations

The slsa MCP tools in this skill operate on **one destination at a time** — whichever
`root_directory` is passed. But the same canonical project exists, with the same internal
shape, across multiple destinations during its lifecycle. The destination model below describes
the canonical roles a destination can play; an acquisition system declares its own concrete set
of destinations and binds each one to a configured filesystem path. The Mesoscope-VR system —
currently the only one defined — happens to instantiate this model with **two acquisition
machines and two long-term storage destinations**, but other systems will compose differently
(e.g. a single acquisition machine with one storage tier).

#### Primary acquisition machine

The machine that runs the acquisition runtime and owns the authoring tier of the project
hierarchy. Sessions are **created here** under `<root>/<project>/<animal>/<session>/raw_data/`,
populated as data is acquired, then handed off to the long-term storage destinations during
preprocessing. Once preprocessing succeeds and the transfer is verified, the local copy is
deleted from this machine — it does not retain sessions long-term. This is also where
per-project `configuration/` (experiment-config YAMLs) lives; the frozen per-session copy
travels with each session into `raw_data/`, so storage destinations do not need a separate
project-level `configuration/` directory.

This destination also carries a per-animal `persistent_data/` slot at
`<root>/<project>/<animal>/persistent_data/` that survives across sessions (descriptor
parameters from the previous session, position snapshots, window screenshots — see
`/session-data` for what these seed in the next session). The persistent slot sits at the
animal level, not under any session, so it does not appear in `discover_sessions_tool` output.

In Mesoscope-VR this role is played by the **VRPC**.

#### Secondary acquisition machines (system-specific)

Some acquisition systems require additional machines that hold subsystem-specific assets the
primary acquisition machine then pulls into the session's `raw_data/` during preprocessing.
These secondary machines may carry their own per-animal `persistent_data/` slots for subsystem
calibration files that survive across sessions but are not part of the main project hierarchy.

In Mesoscope-VR this role is played by the **ScanImage PC**, which holds the mesoscope frame
staging area plus per-animal motion-estimator and ROI calibration files. Other acquisition
systems may have zero, one, or several such secondary machines depending on how their hardware
is split across hosts.

#### Hot storage tier

A long-term storage destination intended to be the working copy that downstream processing
pipelines read from and write into. Receives the preprocessed `raw_data/` per session from the
primary acquisition machine, and accumulates `processed_data/` outputs as processing pipelines
populate them in place (see `/session-data` for the lifecycle of `processed_data/`).

In Mesoscope-VR this role is played by the **compute server**.

#### Cold storage tier

A long-term storage destination intended as a backup of the raw acquired data. Receives the
same preprocessed `raw_data/` as the hot storage tier, in parallel during the preprocessing
transfer. Not used as a source by processing pipelines — it exists to survive a hot-storage
failure.

In Mesoscope-VR this role is played by the **NAS**.

#### Invariants across all destinations

- **Every destination uses the identical `<root>/<project>/<animal>/<session>/` shape.** The
  `<root>` path differs per destination, but the relative structure inside each project is
  always the same. Tools that walk the hierarchy can be pointed at any destination's root with
  no schema change.
- **`persistent_data/` directories live only on acquisition machines** (primary and any
  secondary), since they hold cross-session calibration state that is only meaningful to the
  acquisition runtime. Storage destinations don't carry `persistent_data/`.
- **`configuration/` lives only at the primary acquisition machine's project root by default**.
  Storage destinations rely on the per-session frozen copy under `raw_data/` for any
  configuration metadata they need.
- **The slsa MCP tools have no concept of multi-destination mirroring.** When you call
  `discover_projects_tool(root_directory=<X>)`, you see only what is on `X`. To audit the same
  project across destinations, call the tools once per destination root.

The project shape (`<root>/<project>/<animal>/<session>/`) is acquisition-system-agnostic and
should remain stable as additional systems are added; what varies between systems is the set of
destinations they declare and which configured filesystem path each role binds to.

---

## The project hierarchy

A Sollertia data root (the project hierarchy root, distinct from the slsa working directory) contains
zero or more projects. Each project is a top-level directory under the data root and has the
following nested structure:

```text
<root-directory>/
└── <project>/
    ├── configuration/                          # MesoscopeExperimentConfiguration files
    │   ├── <experiment-1>.yaml
    │   └── <experiment-2>.yaml
    ├── <animal-1>/
    │   ├── <session-1>/
    │   │   └── raw_data/
    │   │       └── session_data.yaml           # SessionData marker — discovered here
    │   ├── <session-2>/
    │   └── ...
    ├── <animal-2>/
    │   └── ...
    └── ...
```

Per-project experiment YAMLs live under `configuration/`. Each session's `session_data.yaml`
marker lives inside `<session>/raw_data/`.

Each subject (animal) belongs to **exactly one project at a time**. The acquisition runtime
(`sollertia-experiment`) enforces this: a session attempt for an animal already associated with a
different project is rejected, and an animal that somehow appears under multiple project
directories is treated as an error state ("often indicative of old migration pipeline use"). To
move a subject to a different project, use the experiment plugin's `/data-management` skill,
which exposes the migration tool that transfers all the animal's session data from the
source project to the destination project.

The `SubjectData` dataclass schema itself carries only an `id` field (no project field), but the
on-disk hierarchy is always `<root>/<project>/<subject>/...`, and the project ↔ subject binding
is determined by which project subdirectory the animal data lives under. `discover_subjects_tool`
returns a `projects: list[str]` per subject as a defensive measure: in healthy state every entry
will have a list of length one.

Datasets are a higher-level grouping that aggregates sessions across animals **within a single
project**. The `DatasetData` schema carries a single `project` field — there is no
cross-project dataset shape. A dataset is identified by a `dataset.yaml` marker discoverable
anywhere under the data root. Datasets are owned by the forging plugin's `/datasets` skill.

---

## MCP tool surface

### Discovery (read-only — natural shares)

| Tool                          | Purpose                                                                                          |
|-------------------------------|--------------------------------------------------------------------------------------------------|
| `discover_projects_tool`      | Lists all projects under the data root, with `animal_count` and `experiment_count` per project   |
| `discover_animals_tool`       | Lists animals within a project, with per-animal `session_count`                                  |
| `discover_sessions_tool`      | Walks the data root and lists sessions (filterable by `project`, `animal_id`, `session_type`)    |
| `discover_experiments_tool`   | Lists experiment configurations under a project                                                  |
| `discover_subjects_tool`      | Lists all subjects (optionally filtered by project)                                              |

These tools may be called by any skill that needs to enumerate the hierarchy. They do not mutate state.

### Aggregation (exclusive to this skill)

| Tool                          | Purpose                                                                                    |
|-------------------------------|--------------------------------------------------------------------------------------------|
| `get_project_overview_tool`   | Returns aggregate counts (animals, sessions by type, experiments, datasets) for a project  |

Use this as the starting point when the user asks "what is in project X" — one call, structured
summary, no need to walk the hierarchy manually.

### Creation (exclusive to this skill)

| Tool                  | Purpose                                                                          |
|-----------------------|----------------------------------------------------------------------------------|
| `create_project_tool` | Creates a new project directory under the working directory                      |

---

## Workflows

### Bootstrap a new project

1. **Verify prerequisites:** The `sollertia-shared-assets` MCP server is connected (else
   `/assets-mcp-environment-setup`). The caller must know the absolute path to the data root.
2. **Confirm the project does not already exist:**
   ```text
   discover_projects_tool(root_directory="<absolute path to data root>")
   ```
3. **Create the project:**
   ```text
   create_project_tool(
       project="<project-name>",
       root_directory="<absolute path to data root>",
   )
   ```
   `root_directory` is required for both calls. If the project directory already exists,
   `create_project_tool` is a **no-op** and returns `already_exists=True` (not an error) — so
   the prior `discover_projects_tool` call is the right way to surface duplicates to the user.
4. **Verify creation:**
   ```text
   discover_projects_tool(root_directory="<absolute path to data root>")
   ```
5. **Hand off to `/experiment-configuration`** if the user wants to author an experiment configuration
   for the new project. Project creation does not author any experiment YAML — that is a separate
   responsibility.

### Enumerate sessions for an animal

1. **Verify the project and animal exist:**
   ```text
   discover_projects_tool(root_directory="<absolute>")
   discover_animals_tool(project="<project>", root_directory="<absolute>")
   ```
2. **List sessions:**
   ```text
   discover_sessions_tool(
       root_directory="<absolute>",
       project="<project>",
       animal_id="<animal>",
   )
   ```
   The animal **filter kwarg** is `animal_id`. The corresponding key in each returned session
   summary dict is `animal` (not `animal_id`) — relevant if you re-filter the response in
   downstream code.
3. **Hand off to `/session-data`** to read individual `SessionData` markers, or to `/session-descriptors`
   to read the per-session descriptors.

### Audit subjects across projects

1. **List subjects:**
   ```text
   discover_subjects_tool(root_directory="<absolute>")
   ```
   `root_directory` is required. Pass `project="<name>"` to scope the listing to a single project.
2. **Hand off to `/subject-metadata`** to read individual subject records (surgery, implants,
   injections, drugs).

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] discover_projects_tool was called before creating a new project (avoid duplicates)
- [ ] create_project_tool succeeded (or returned already_exists=True, which is a no-op success)
- [ ] discover_projects_tool returned the new project after creation
- [ ] Did not call any write_* or set_* tool from this skill — only create_project_tool
- [ ] Handed off to /experiment-configuration for any experiment authoring
- [ ] Handed off to /session-data, /session-descriptors, /subject-metadata, or forging plugin's
      /datasets for any read that goes deeper than the hierarchy itself
```

---

## Related skills

| Skill                           | Relationship                                            |
|---------------------------------|---------------------------------------------------------|
| `/assets-mcp-environment-setup` | Run first if the MCP server is not connected            |
| `/experiment-configuration`     | Consumes new projects to author experiment YAMLs        |
| `/session-data`                 | Reads `SessionData` markers discovered via this skill   |
| `/session-descriptors`          | Reads per-session descriptors discovered via this skill |
| `/subject-metadata`             | Reads subject records discovered via this skill         |
| forging plugin `/datasets`      | Aggregates sessions discovered via this skill           |
