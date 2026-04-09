---
name: datasets
description: >-
  Discovers, reads, and writes dataset-level YAML files (DatasetData, DatasetSession membership) for
  sollertia-shared-assets via the sl-configure MCP server. Owns the dataset write tools and schema
  introspection. Use when curating a dataset, adding sessions to an existing dataset, or building
  tooling that needs dataset-level introspection.
user-invocable: true
---

# Sollertia datasets

Discovers, reads, and writes the dataset-level YAML files that group sessions for downstream analysis
and sharing. Uses the `sl-configure mcp` MCP server.

---

## Scope

**Covers:**
- Discovering datasets across projects
- Reading the `DatasetData` marker file
- Writing or amending dataset membership (`DatasetSession`)
- Schema introspection for dataset YAML files

**Does not cover:**
- Authoring system or experiment configuration (see `/system-configuration`,
  `/experiment-configuration`)
- Authoring per-session metadata (see `/session-data`)
- Initial working directory setup (see `/working-directory`)
- Post-acquisition processing of dataset contents (deferred to the processing plugin)

---

## Anatomy of a dataset

A dataset is a directory containing a `dataset.yaml` marker file. The marker captures:

- Dataset identity (name, owner, description)
- The list of `DatasetSession` entries — each entry references a session by canonical path / ID
- Optional dataset-level metadata (acquisition date range, intended use, sharing status)

`discover_datasets_tool` walks the working directory and recognizes any directory containing a
`dataset.yaml` marker.

Datasets are **a higher-level grouping than sessions**. Where a session is a single recording, a
dataset is a curated collection of sessions intended to be processed and analyzed together (e.g.,
"all mesoscope experiment sessions for project X between dates Y and Z that passed quality control").

---

## MCP tool surface

| Tool                            | Purpose                                                                |
|---------------------------------|------------------------------------------------------------------------|
| `discover_datasets_tool`        | Lists all datasets under the working directory (filterable by project) |
| `read_dataset_tool`             | Reads the `DatasetData` marker for a given dataset                     |
| `write_dataset_tool`            | Writes a new dataset marker or overwrites an existing one              |
| `describe_dataset_schema_tool`  | Returns the field schema for `DatasetData` and `DatasetSession`        |

---

## Authoring workflow

### Step 1: Verify prerequisites

- MCP server connected (else `/mcp-environment-setup`)
- Working directory set (else `/working-directory`)

### Step 2: Discover existing datasets

```text
discover_datasets_tool()
```

If a dataset already covers the sessions the user wants to group, prefer reading and amending it
rather than creating a new dataset.

### Step 3: Inspect the schema

```text
describe_dataset_schema_tool()
```

### Step 4: Identify member sessions

Use `discover_sessions_tool` (owned by `/project-hierarchy`) to enumerate candidate sessions. Filter by
project, animal, session type, and date range as needed. Confirm the membership list with the user
before writing.

### Step 5: Author the dataset

Build the dataset dictionary including a `sessions` list of `DatasetSession` entries. Each
`DatasetSession` references a session by its canonical identifier so the dataset stays valid even if
session directories are relocated.

### Step 6: Write and verify

```text
write_dataset_tool(dataset={ ... full nested dict ... })
read_dataset_tool(dataset_path="<absolute>")
```

Confirm the returned dataset matches the intended membership.

---

## Modifying an existing dataset

Common operations:

| Goal                              | Pattern                                                                  |
|-----------------------------------|--------------------------------------------------------------------------|
| Add a session                     | Read → append `DatasetSession` to `sessions` list → write                |
| Remove a session                  | Read → remove the entry → write — confirm with the user first            |
| Rename a dataset                  | Read → mutate `name` → write                                             |
| Change the owner / description    | Read → mutate the relevant fields → write                                |

For any operation that removes membership or changes identity, **confirm with the user first** —
downstream tooling and analysis notebooks may have references that break silently.

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] describe_dataset_schema_tool was used as the source of truth for field names
- [ ] discover_datasets_tool was called before creating new datasets (avoid duplicates)
- [ ] Member sessions were discovered via discover_sessions_tool, not guessed
- [ ] User confirmed membership additions and removals before writing
- [ ] read_dataset_tool returned the expected dataset after every write
```

---

## Related skills

| Skill                    | Relationship                                                    |
|--------------------------|-----------------------------------------------------------------|
| `/working-directory`     | Required prerequisite — must be run first                       |
| `/mcp-environment-setup` | Run first if the MCP server is not connected                    |
| `/project-hierarchy`     | Provides `discover_sessions_tool` for dataset membership lookup |
| `/session-data`          | Sibling — sessions are the membership unit of datasets          |
