---
name: project-hierarchy
description: >-
  Discovers the Sollertia project hierarchy (projects, animals, experiments, subjects,
  sessions) and creates new projects via the sollertia-shared-assets MCP server. Owns
  get_data_root_overview_tool (hierarchy discovery) and create_project_tool (project creation).
  Use when enumerating projects, animals, or sessions, walking the project tree, or creating a project.
user-invocable: false
---

# Sollertia project hierarchy

Skill for discovering and creating entries in the Sollertia project hierarchy. Project / animal /
session listings come from `get_data_root_overview_tool` on the `slsa mcp` MCP server; new projects
are materialized with `create_project_tool`. By default, the overview walks every `session_data.yaml`
marker under the data root and groups results by the identity fields inside each `SessionData`, so
stray directories cannot surface as phantom projects or animals. Its `directories` strategy
additionally surfaces empty project and animal directories that hold no sessions yet.

---

## Scope

**Covers:**
- Discovering projects, animals, experiments, subjects, and sessions
- Creating new projects (the `create_project_tool` MCP tool, equivalent to the `slsa configure project` CLI)
- The directory layout of a Sollertia project tree
- The relationship between projects, animals, sessions, experiments, and subjects

**Does not cover:**
- Authoring per-project experiment configuration YAMLs (see `/experiment-configuration`; for the
  Mesoscope-VR concrete schema, `mesoscope:mesoscope-vr-experiment-schema`)
- Reading or writing `SessionData` (see `/session-data`)
- Per-session inventory and health reports (see `/session-data`, which owns `inspect_sessions_tool`)
- Reading or writing session descriptors (see `/session-descriptors`)
- Reading subject metadata (see `/data-assets`)
- Reading or writing datasets (see `forging:datasets`)
- Initial working directory setup (see `/working-directory`)

This skill's discovery tool (`get_data_root_overview_tool`) is read-only and may be called as a
**natural share** by any other skill that needs to enumerate the hierarchy. Project creation
(`create_project_tool`) is a write operation owned by this skill.

---

## What is a project

A **project** is the top-level scientific grouping of acquired data: a logical container for
**all sessions belonging to one research investigation**, organized by animal. Every session,
every experiment configuration, and every per-animal calibration artifact is anchored to exactly
one project. Project names are arbitrary strings chosen by the experimenter (commonly the project
abbreviation used in templates, e.g. `MaalstroomicFlow`, `StateSpaceOdyssey`); the only
slsa-side constraint is that the project name matches the `project_name` field inside each
session's `SessionData`.

A project bundles three kinds of state that this skill covers:

1. **Shared configuration** — per-project experiment YAMLs under `configuration/`, authored by
   `/experiment-configuration` once and reused across many sessions and animals.
2. **Animal subtrees** — one directory per participating animal, each containing that animal's
   sessions.
3. **Per-session subtrees** — timestamped session directories owned by `/session-data` and the
   sibling session-* skills. See `/session-data` for what a session is and how it is named.

Acquisition-machine-specific artifacts (per-animal `persistent_data/` caches, multi-destination
topology, acquisition vs. storage-tier placement rules) are **out of scope** for this skill —
they are owned by the experiment plugin. `get_data_root_overview_tool` treats the
supplied `root_directory` as a single opaque destination and does not interpret which node in a
distributed system it points at.

### Why the structure is shaped this way

- **Project at the top** captures the unit of scientific ownership. Multiple animals contribute
  to one project; session-level analysis joins across animals within a project. Datasets are
  also project-scoped — each dataset belongs to exactly one project and aggregates sessions
  across animals within that project (see `forging:datasets`) — they are a downstream
  aggregation, not a primary hierarchy.
- **Animals nested under project, not the inverse.** Each animal belongs to exactly one project
  at a time. Inverting the hierarchy ("animal at top, projects under it") would require
  duplicating animal metadata per project and would not match the session-path shape
  `<root>/<project>/<animal>/<session>/` that every tool consumes.
- **`configuration/` is per-project**, not per-animal or per-session, because experiment
  paradigms are a project-scope decision. A frozen per-session snapshot lives inside each
  session's `raw_data/experiment_configuration.yaml`; the master copy under `configuration/`
  is what new sessions are instantiated from.
- **Sessions are flat under each animal** — no `experiments/<exp-name>/<session>/` grouping —
  because the timestamped session name already gives chronological ordering and each session's
  marker file (`SessionData.experiment_name`) names the experiment used. Forcing experiment
  grouping into the directory structure would prevent training and window-checking sessions
  (which have no experiment) from sharing a sibling slot.

---

## The project hierarchy

A Sollertia data root (the project hierarchy root, distinct from the slsa working directory) contains
zero or more projects. Each project is a top-level directory under the data root and has the
following nested structure:

```text
<root-directory>/
└── <project>/
    ├── configuration/                          # per-system experiment configuration YAMLs
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

Each subject (animal) is expected to belong to **exactly one project at a time**. An animal that
surfaces under multiple project entries in `get_data_root_overview_tool` output is an error
state. Remediation — migrating a subject from one project to another — is owned by the
`experiment:data-management` skill,
which exposes the migration tool that transfers all the animal's session data from the source
project to the destination project.

The project ↔ animal binding is determined by the `project_name` field inside each session's
`SessionData` — not by directory placement. `get_data_root_overview_tool` groups the `projects`
list by `SessionData.project_name`, and the `animals` list under each project reflects every
animal with at least one session naming that project. An animal whose sessions name different
projects will appear under every project it has contributed to — a healthy data root has each
animal under exactly one project. Subject-level metadata (surgery, implants, drugs, injections)
is owned by `/data-assets` and is outside the scope of this skill.

Datasets are a higher-level grouping that aggregates sessions across animals **within a single
project**. The `DatasetData` schema carries a single `project` field — there is no
cross-project dataset shape. A dataset is identified by a `dataset.yaml` marker discoverable
anywhere under the data root. Datasets are owned by `forging:datasets` skill.

### How the hierarchy is modeled and enumerated

The library models this tree with two path-grammar dataclasses — `ProjectData` (a `<root>/<project>`
view) and `AnimalData` (a `<root>/<project>/<animal>` view). Both are pure path views: they resolve
locations from the root and names without requiring the directories to exist, and either can be rebound
onto a different root (such as a mounted storage tier) via `for_root`. `ProjectData.experiment_configs()`
lists a project's `configuration/*.yaml` files; `AnimalData.session_path(<name>)` resolves a session
directory. Agents do not instantiate these directly — they surface through the MCP tool and the
`slsa get` CLI described below.

Project enumeration has **two strategies that disagree on empty projects**:

- **Markers (authoritative, default).** Buckets projects by the `project_name` inside each discovered
  `session_data.yaml`. Only projects that hold at least one session surface, so stray directories cannot
  appear as phantom projects.
- **Directories.** Walks the project / animal directory layout directly, so it also surfaces freshly
  created projects — and projects whose sessions have been migrated to long-term storage — that hold no
  sessions. This is what the `slsa get projects` CLI command uses.

`get_data_root_overview_tool` accepts a `strategy` argument selecting between them. It defaults to
`markers` and, when called with `directories`, augments the result with the empty project and animal
directories the marker walk misses. The distinction matters right after `create_project_tool` /
`slsa configure project`: the new, empty project is invisible under the default `markers` strategy (no
session markers yet) but visible under `directories` and to `slsa get projects`.

---

## MCP tool surface

### Discovery (read-only — natural share)

| Tool                          | Purpose                                                                                                                                                                                                                                               |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `get_data_root_overview_tool` | Builds the project → animal → session hierarchy from `SessionData` contents, with per-project aggregate counts (animals, sessions-by-type, lifecycle status, `experiment_count`, `dataset_count`) and a flat `sessions` list for downstream filtering |

`get_data_root_overview_tool` accepts a `strategy` argument (`markers` default, or `directories` to also
surface empty project and animal directories — see the two-strategies section above).

### Creation (write)

| Tool                  | Purpose                                                                                                                            |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------|
| `create_project_tool` | Creates a new project structure (`<root>/<project>/configuration/`) under the configured data root or an explicit `root_directory` |

For per-project experiment-configuration enumeration, use `discover_experiments_tool` via
`/experiment-configuration` — it owns that tool.

The `slsa get` CLI group reports the same hierarchy from the persisted data root without going through
the MCP server: `slsa get projects` lists project directories (directories strategy — includes empty
projects), and `slsa get experiments -p <project>` lists a project's experiment-configuration stems (via
`ProjectData.experiment_configs()`). Use these for a quick host-side check; use `get_data_root_overview_tool`
when an agent needs the full session-level tree.

`get_data_root_overview_tool` is the one-call replacement for all previous project / animal /
session enumeration tools. Per-project aggregates (`sessions_by_type`, `experiment_count`,
`dataset_count`) are carried inside each `projects[*]` entry; callers that want the overview of
a single project filter the tool's output by `project == "<name>"`. The flat `sessions` list is
shaped for downstream chaining with `filter_sessions_tool` (see `/session-discovery`).

### Response shape (partial)

```text
{
  "root_directory": "<root>",
  "projects": [
    {
      "name": "<project>",
      "path": "<root>/<project>",
      "animals": [
        {"id": "<animal>", "session_paths": [...], "session_count": N, "counts": {...}},
        ...
      ],
      "session_count": N,
      "counts": {"uninitialized": N, "incomplete": N, "acquired": N, "processed": N, "error": N},
      "sessions_by_type": {"mesoscope experiment": N, ...},
      "experiment_count": N,
      "dataset_count": N
    },
    ...
  ],
  "sessions": [<flat per-session entries>],
  "counts": {...},
  "total_projects": N, "total_animals": N, "total_sessions": N
}
```

---

## Workflows

### Survey what is on a data root

1. **Verify prerequisites:** The `sollertia-shared-assets` MCP server is connected (else
   `/assets-mcp-environment-setup`). The caller must know the absolute path to the data root — when the
   host has one persisted, recall it with `read_data_root_tool` (owned by `/working-directory`) rather
   than asking the user.
2. **Fetch the overview:**
   ```text
   get_data_root_overview_tool(root_directory="<absolute path to data root>")
   ```
3. **Read the response:**
   - `projects[*]` — the authoritative project list. A project appears only when at least one
     session names it, so stray directories at the root cannot masquerade as projects.
   - `projects[*].animals[*]` — animals per project, each with `session_paths` and status counts.
   - `sessions` — flat list suitable for chaining into `filter_sessions_tool`.

### Enumerate sessions for an animal

1. **Fetch the overview:**
   ```text
   get_data_root_overview_tool(root_directory="<absolute>")
   ```
2. **Filter client-side** to the target project and animal:
   ```text
   project_entry = next(p for p in response["projects"] if p["name"] == "<project>")
   animal_entry  = next(a for a in project_entry["animals"] if a["id"] == "<animal>")
   session_paths = animal_entry["session_paths"]
   ```
   Or — for date-range and animal-level filtering on the flat list — hand off to
   `/session-discovery`, which chains `get_data_root_overview_tool` through
   `filter_sessions_tool`.
3. **Hand off to `/session-data`** for per-session inventory and health reports (it owns
   `inspect_sessions_tool`), `/session-descriptors` for descriptor reads, or
   `/session-discovery` when the workflow needs date-range filtering and a flat `session_paths`
   handoff to a batch pipeline.

### Audit animals across projects

1. **Fetch the overview:**
   ```text
   get_data_root_overview_tool(root_directory="<absolute>")
   ```
2. **Scan `projects[*].animals[*].id` across projects**; any animal that appears under more than
   one project is an error state. Flag those IDs to the user and hand off to the experiment
   plugin's `experiment:data-management` to migrate the subject if needed.
3. **Hand off to `/data-assets`** to read individual subject records (surgery, implants,
   injections, drugs).

### Bootstrap a new project

Create a project with `create_project_tool`, which materializes `<root>/<project>/configuration/`
under the configured data root (or an explicit `root_directory`). The equivalent CLI command is
`slsa configure project -p <project_name>`, which always resolves the root from the persisted data
root. An explicit root is available through `create_project_tool` alone, via its optional
`root_directory` argument. The project directory must exist before `SessionData.create` (in
`experiment:data-management`) can create the first session. `SessionData.create` raises
`FileNotFoundError` when the project directory is missing.
A brand-new project holds no session markers, so confirm it with `get_data_root_overview_tool`'s
`directories` strategy (or `slsa get projects`). The default `markers` strategy lists it once it holds
a session.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] get_data_root_overview_tool was used for any project / animal / session enumeration
- [ ] Project creation used create_project_tool (explicit root supported) or slsa configure project (persisted root)
- [ ] Did not call write_* / set_* tools beyond create_project_tool. Hierarchy discovery remains read-only
- [ ] Handed off to /experiment-configuration for any experiment authoring
- [ ] Handed off to /session-data, /session-descriptors, /data-assets, or forging plugin's
      forging:datasets for any read that goes deeper than the hierarchy itself
```

---

## Related skills

| Skill                           | Relationship                                                                                           |
|---------------------------------|--------------------------------------------------------------------------------------------------------|
| `/assets-mcp-environment-setup` | Run first if the MCP server is not connected                                                           |
| `/working-directory`            | Required prerequisite — bootstraps the local working directory the agent uses to resolve project roots |
| `experiment:data-management`    | Creates sessions. Project directories must exist beforehand (`create_project_tool`)                    |
| `/experiment-configuration`     | Consumes projects to author experiment YAMLs                                                           |
| `/session-discovery`            | Chains `get_data_root_overview_tool` through `filter_sessions_tool`                                    |
| `/session-data`                 | Owns `inspect_sessions_tool` for per-session inventory and health reports                              |
| `/session-descriptors`          | Reads per-session descriptors                                                                          |
| `/data-assets`                  | Reads read assets (e.g., surgery/subject records)                                                      |
| `forging:datasets`              | Aggregates sessions                                                                                    |
