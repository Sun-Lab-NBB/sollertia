---
name: project-hierarchy
description: >-
  Discovers the Sollertia project hierarchy (projects, animals, experiments, subjects,
  sessions) via the sollertia-shared-assets MCP server. Wraps the single
  get_data_root_overview_tool that builds the tree from SessionData contents. Use when
  enumerating projects, animals, or sessions, or walking the project tree.
user-invocable: true
---

# Sollertia project hierarchy

Reader skill for the Sollertia project hierarchy. Every project / animal / session listing is
built from a single call to `get_data_root_overview_tool` on the `slsa mcp` MCP server. The tool
walks every `session_data.yaml` marker under the data root and groups results by the identity
fields inside each `SessionData`, so stray directories cannot surface as phantom projects or
animals.

---

## Scope

**Covers:**
- Discovering projects, animals, experiments, subjects, and sessions
- The directory layout of a Sollertia project tree
- The relationship between projects, animals, sessions, experiments, and subjects

**Does not cover:**
- Creating new projects. Project directories are created **implicitly** by the sollertia-experiment
  session-creation flow when the first session lands; there is no dedicated project-creation MCP
  tool. Defer to the experiment plugin's `/managing-session-data` for session creation.
- Authoring per-project `MesoscopeExperimentConfiguration` (see `/experiment-configuration`)
- Reading or writing `SessionData` (see `/session-data`)
- Per-session inventory and health reports (see `/session-data`, which owns `inspect_sessions_tool`)
- Reading or writing session descriptors (see `/session-descriptors`)
- Reading subject metadata (see `/subject-metadata`)
- Reading or writing datasets (see forging plugin's `/datasets`)
- Initial working directory setup (see `/working-directory`)

This skill's single tool is read-only and may be called as a **natural share** by any other skill
that needs to enumerate the hierarchy.

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
  across animals within that project (see forging plugin's `/datasets`) — they are a downstream
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

Each subject (animal) is expected to belong to **exactly one project at a time**. An animal that
surfaces under multiple project entries in `get_data_root_overview_tool` output is an error
state. Remediation — migrating a subject from one project to another — is owned by the
experiment plugin's `/data-management` skill,
which exposes the migration tool that transfers all the animal's session data from the source
project to the destination project.

The project ↔ animal binding is determined by the `project_name` field inside each session's
`SessionData` — not by directory placement. `get_data_root_overview_tool` groups the `projects`
list by `SessionData.project_name`, and the `animals` list under each project reflects every
animal with at least one session naming that project. An animal whose sessions name different
projects will appear under every project it has contributed to — a healthy data root has each
animal under exactly one project. Subject-level metadata (surgery, implants, drugs, injections)
is owned by `/subject-metadata` and is outside the scope of this skill.

Datasets are a higher-level grouping that aggregates sessions across animals **within a single
project**. The `DatasetData` schema carries a single `project` field — there is no
cross-project dataset shape. A dataset is identified by a `dataset.yaml` marker discoverable
anywhere under the data root. Datasets are owned by the forging plugin's `/datasets` skill.

---

## MCP tool surface

### Discovery (read-only — natural share)

| Tool                          | Purpose                                                                                                                                                                                                                                               |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `get_data_root_overview_tool` | Builds the project → animal → session hierarchy from `SessionData` contents, with per-project aggregate counts (animals, sessions-by-type, lifecycle status, `experiment_count`, `dataset_count`) and a flat `sessions` list for downstream filtering |
| `discover_experiments_tool`   | Lists experiment configurations under a project                                                                                                                                                                                                       |

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
   `/assets-mcp-environment-setup`). The caller must know the absolute path to the data root.
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
   plugin's `/data-management` to migrate the subject if needed.
3. **Hand off to `/subject-metadata`** to read individual subject records (surgery, implants,
   injections, drugs).

### Bootstrap a new project

Project directories are created implicitly by the sollertia-experiment session-creation flow
when the first session lands — there is no dedicated MCP tool. Hand off to the experiment
plugin's `/managing-session-data`, which owns session creation and will `mkdir(parents=True)`
the project and animal directories as part of `SessionData.create`. After the first session is
created, confirm the project is visible with `get_data_root_overview_tool`.

---

## Verification checklist

```text
- [ ] sollertia-shared-assets MCP server is connected
- [ ] get_data_root_overview_tool was used for any project / animal / session enumeration
- [ ] Project-creation workflows were handed off to the experiment plugin's /managing-session-data
- [ ] Did not call any write_* or set_* tool from this skill — this skill is read-only
- [ ] Handed off to /experiment-configuration for any experiment authoring
- [ ] Handed off to /session-data, /session-descriptors, /subject-metadata, or forging plugin's
      /datasets for any read that goes deeper than the hierarchy itself
```

---

## Related skills

| Skill                                             | Relationship                                                                |
|---------------------------------------------------|-----------------------------------------------------------------------------|
| `/assets-mcp-environment-setup`                   | Run first if the MCP server is not connected                                |
| experiment plugin `/managing-session-data`        | Creates sessions; project directories appear implicitly during that flow    |
| `/experiment-configuration`                       | Consumes projects to author experiment YAMLs                                |
| `/session-discovery`                              | Chains `get_data_root_overview_tool` through `filter_sessions_tool`         |
| `/session-data`                                   | Owns `inspect_sessions_tool` for per-session inventory and health reports   |
| `/session-descriptors`                            | Reads per-session descriptors                                               |
| `/subject-metadata`                               | Reads subject records                                                       |
| forging plugin `/datasets`                        | Aggregates sessions                                                         |
