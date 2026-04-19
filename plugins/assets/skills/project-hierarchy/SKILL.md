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

The per-project experiment YAML directory is `configuration/`, not `experiments/`. The per-session
`session_data.yaml` marker lives **inside `<session>/raw_data/`**, not at the session root.

Subjects are a parallel hierarchy that crosses projects — a single animal may participate in multiple
projects over its lifetime, and `SubjectData` records are keyed by subject ID, not by project.

Datasets are a higher-level grouping that aggregates sessions across projects and animals; a dataset
is identified by a `dataset.yaml` marker discoverable anywhere under the data root. Datasets are
owned by the forging plugin's `/datasets` skill.

---

## MCP tool surface

### Discovery (read-only — natural shares)

| Tool                          | Purpose                                                                  |
|-------------------------------|--------------------------------------------------------------------------|
| `discover_projects_tool`      | Lists all projects under the working directory                           |
| `discover_animals_tool`       | Lists animals within a project                                           |
| `discover_sessions_tool`      | Walks the working directory and lists sessions (filterable)              |
| `discover_experiments_tool`   | Lists experiment configurations under a project                          |
| `discover_subjects_tool`      | Lists all subjects (optionally filtered by project)                      |

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

1. **Verify prerequisites:**
   - The `sollertia-shared-assets` MCP server is connected (else `/assets-mcp-environment-setup`).
   - The working directory is set (else `/working-directory`).
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
   `root_directory` is required for both calls. The slsa working directory is a separate concept
   (cache root for platform configuration) and does not implicitly resolve the data root.
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
   The animal filter kwarg is `animal_id`, not `animal`.
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
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] discover_projects_tool was called before creating a new project (avoid duplicates)
- [ ] create_project_tool succeeded
- [ ] discover_projects_tool returned the new project after creation
- [ ] Did not call any write_* or set_* tool from this skill — only create_project_tool
- [ ] Handed off to /experiment-configuration for any experiment authoring
- [ ] Handed off to /session-data, /session-descriptors, /subject-metadata, or forging plugin's
      /datasets for any read that goes deeper than the hierarchy itself
```

---

## Related skills

| Skill                          | Relationship                                                       |
|--------------------------------|--------------------------------------------------------------------|
| `/working-directory`           | Required prerequisite — must be run first                          |
| `/assets-mcp-environment-setup`       | Run first if the MCP server is not connected                       |
| `/experiment-configuration`    | Consumes new projects to author experiment YAMLs                   |
| `/session-data`                | Reads `SessionData` markers discovered via this skill              |
| `/session-descriptors`         | Reads per-session descriptors discovered via this skill            |
| `/subject-metadata`            | Reads subject records discovered via this skill                    |
| forging plugin `/datasets`     | Aggregates sessions discovered via this skill                      |
