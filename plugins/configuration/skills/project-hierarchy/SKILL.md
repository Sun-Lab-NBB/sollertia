---
name: configure-project-hierarchy
description: >-
  Discovers and creates entries in the Sollertia project hierarchy (projects, animals, experiments,
  subjects, sessions) via the sl-configure MCP server. Owns create_project_tool. Covers project bootstrap,
  hierarchy traversal, and the relationship between projects, animals, sessions, and experiment
  configurations. Use when bootstrapping a new project, enumerating animals or sessions under a project,
  or building tooling that needs to walk the project tree.
user-invocable: true
---

# Sollertia project hierarchy

Discovers and creates entries in the Sollertia project hierarchy using the `sl-configure mcp` MCP server.
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
- Reading or writing datasets (see `/datasets`)
- Initial working directory setup (see `/working-directory`)

The `discover_*` tools below are read-only and may be called as **natural shares** by other skills that
need to enumerate the hierarchy. Only `create_project_tool` is exclusive to this skill.

---

## The project hierarchy

A Sollertia working directory contains zero or more projects. Each project is a top-level directory
under the working directory and has the following nested structure:

```
<working-directory>/
└── <project>/
    ├── <animal-1>/
    │   ├── <session-1>/        # SessionData lives here
    │   ├── <session-2>/
    │   └── ...
    ├── <animal-2>/
    │   └── ...
    ├── experiments/             # MesoscopeExperimentConfiguration files
    │   ├── <experiment-1>.yaml
    │   └── <experiment-2>.yaml
    └── ...
```

Subjects are a parallel hierarchy that crosses projects — a single animal may participate in multiple
projects over its lifetime, and `SubjectData` records are keyed by subject ID, not by project.

Datasets are a higher-level grouping that aggregates sessions across projects and animals.

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

### Creation (exclusive to this skill)

| Tool                  | Purpose                                                                          |
|-----------------------|----------------------------------------------------------------------------------|
| `create_project_tool` | Creates a new project directory under the working directory                      |

---

## Workflows

### Bootstrap a new project

1. **Verify prerequisites:**
   - The `sollertia-shared-assets` MCP server is connected (else `/mcp-environment-setup`).
   - The working directory is set (else `/working-directory`).
2. **Confirm the project does not already exist:**
   ```text
   discover_projects_tool()
   ```
3. **Create the project:**
   ```text
   create_project_tool(project="<project-name>")
   ```
4. **Verify creation:**
   ```text
   discover_projects_tool()
   ```
5. **Hand off to `/experiment-configuration`** if the user wants to author an experiment configuration
   for the new project. Project creation does not author any experiment YAML — that is a separate
   responsibility.

### Enumerate sessions for an animal

1. **Verify the project and animal exist:**
   ```text
   discover_projects_tool()
   discover_animals_tool(project="<project>")
   ```
2. **List sessions:**
   ```text
   discover_sessions_tool(project="<project>", animal="<animal>")
   ```
3. **Hand off to `/session-data`** to read individual `SessionData` markers, or to `/session-descriptors`
   to read the per-session descriptors.

### Audit subjects across projects

1. **List subjects:**
   ```text
   discover_subjects_tool()
   ```
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
- [ ] Handed off to /session-data, /session-descriptors, /subject-metadata, or /datasets for any read
      that goes deeper than the hierarchy itself
```

---

## Related skills

| Skill                          | Relationship                                                       |
|--------------------------------|--------------------------------------------------------------------|
| `/working-directory`           | Required prerequisite — must be run first                          |
| `/mcp-environment-setup`       | Run first if the MCP server is not connected                       |
| `/experiment-configuration`    | Consumes new projects to author experiment YAMLs                   |
| `/session-data`                | Reads `SessionData` markers discovered via this skill              |
| `/session-descriptors`         | Reads per-session descriptors discovered via this skill            |
| `/subject-metadata`            | Reads subject records discovered via this skill                    |
| `/datasets`                    | Aggregates sessions discovered via this skill                      |
