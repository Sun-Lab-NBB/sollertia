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

The slsa MCP tools operate on **one destination at a time** — whichever `root_directory` is
passed. The same canonical project exists across multiple destinations (acquisition machines,
storage tiers) during its lifecycle, but **every destination uses the identical
`<root>/<project>/<animal>/<session>/` shape**, so tools that walk the hierarchy can be pointed
at any destination's root with no schema change.

Two slsa-side caveats matter when reasoning across destinations:

- **`persistent_data/` directories live only on acquisition machines** at
  `<root>/<project>/<animal>/persistent_data/` (cross-session calibration state, descriptor
  seeds, position snapshots — see `/session-data`). They sit at the animal level and do not
  appear in `discover_sessions_tool` output.
- **`configuration/` lives only at the primary acquisition machine's project root by default**;
  storage destinations rely on the per-session frozen copy under `raw_data/`.

The concrete set of destinations a system declares (acquisition machines, hot/cold storage
tiers) and which filesystem path each binds to is **system-specific** and owned by the
experiment plugin's `/system-configuration` and `/managing-session-data`. Defer to those skills
for the Mesoscope-VR topology (VRPC, ScanImage PC, compute server, NAS) and for any additional
acquisition system that registers its own destinations.

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

| Tool                        | Purpose                                                                                        |
|-----------------------------|------------------------------------------------------------------------------------------------|
| `discover_projects_tool`    | Lists all projects under the data root, with `animal_count` and `experiment_count` per project |
| `discover_animals_tool`     | Lists animals within a project, with per-animal `session_count`                                |
| `discover_sessions_tool`    | Lists sessions under the data root (filterable by `project`, `animal_id`, and `session_types`) |
| `discover_experiments_tool` | Lists experiment configurations under a project                                                |
| `discover_subjects_tool`    | Lists all subjects (optionally filtered by project)                                            |

These tools may be called by any skill that needs to enumerate the hierarchy. They do not mutate state.

`discover_sessions_tool` also emits a flat `session_paths` list of eligible session roots that
downstream batch pipelines consume directly. For date-range and animal-level filtering workflows
that chain into the sollertia-forgery batch pipelines, defer to `/session-discovery` instead — that
skill owns the discover → filter → hand-off pattern and documents every filter parameter and return
key in full.

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
3. **Hand off to `/session-data`** to read individual `SessionData` markers, to `/session-descriptors`
   to read the per-session descriptors, or to `/session-discovery` when the workflow needs date-range
   filtering and a flat `session_paths` handoff to a batch pipeline.

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

| Skill                           | Relationship                                                          |
|---------------------------------|-----------------------------------------------------------------------|
| `/assets-mcp-environment-setup` | Run first if the MCP server is not connected                          |
| `/experiment-configuration`     | Consumes new projects to author experiment YAMLs                      |
| `/session-discovery`            | Wraps `discover_sessions_tool` with date filtering and batch hand-off |
| `/session-data`                 | Reads `SessionData` markers discovered via this skill                 |
| `/session-descriptors`          | Reads per-session descriptors discovered via this skill               |
| `/subject-metadata`             | Reads subject records discovered via this skill                       |
| forging plugin `/datasets`      | Aggregates sessions discovered via this skill                         |
