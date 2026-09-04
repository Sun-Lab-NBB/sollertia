# Planning tool responses

Every return key the six planning and inspection tools carry, with the conditions under which a key is present. The
response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

---

## `plan_session_jobs_tool` and `plan_dataset_jobs_tool`

```text
success:              Boolean flag
host:                 The host argument, echoed back
total_units:          Entries in `units`
total_jobs:           Sum of every entry's `job_count`
elapsed_seconds:      Millisecond-resolved wall time the planning pass took
units[]:              One entry per named unit, in the order they were named:
  unit_path:          The requested path, echoed back. Always present
  unit_name:          The planned unit's own name. Present on success
  job_count:          Jobs the plan now holds for the unit. `0` on a failed entry
  summed_memory_mb:   Sum over every planned job. Present on success, and never a budget
  unsized_jobs{}:     Each sizing refusal mapped to its reason. Present only when the plan recorded one
  error:              The exception that stopped this unit. Present instead of the success keys
```

A local success entry carries `unit_path`, `unit_name`, `job_count`, and `summed_memory_mb`, plus `unsized_jobs` when
the plan recorded a refusal. A remote entry carries the same four keys, read back out of the projection rather than
parsed from command output, and never carries `unsized_jobs`. A unit whose plan raised carries `unit_path`, `error`,
and `job_count: 0`, whatever the host.

A remote unit that planned nothing reports `The project's plan projection holds no job for this unit, so it planned
nothing.` in its `error` key.

---

## `generate_project_plan_tool`

```text
success:                 Boolean flag
project_path:            The requested path, echoed back
host:                    The host argument
plan_path:               `<project>/<project stem>_plan.feather`
elapsed_seconds:         Wall time the projection took
total_jobs:              Rows in the projection, `0` when it holds none
summed_memory_mb:        Sum of every row's `memory_mb`
largest_job_memory_mb:   Largest single-job figure
widest_job_cores:        Widest single-job core count
pipeline_totals[]:       One entry per `(unit_kind, pipeline)` pair, sorted by unit kind then pipeline:
  unit_kind:             `"session"` or `"dataset"`
  pipeline:              The pipeline identifier
  jobs:                  The row count for the pair. The key is `jobs`, NOT `job_count`
  summed_memory_mb:      Sum of the pair's `memory_mb`
  widest_job_cores:      Widest core count in the pair
```

An empty projection reports all four totals as `0` and `pipeline_totals` as an empty list.

---

## `read_project_plan_tool`

```text
success:                 Boolean flag
project_path:            The mirror path for `host="remote"`, the requested path for `local`
plan_path:               The projection this read resolved
total_jobs, summed_memory_mb, largest_job_memory_mb, widest_job_cores:
                         The same totals block generate_project_plan_tool returns, spanning EVERY planned
                         job regardless of the filters
breakdown:               Counts over `unit_kind`, `animal`, `dataset`, `pipeline`, and `job_name`, also
                         spanning every planned job. An axis holding more distinct values than the shared
                         cap reports how many it holds in place of its counts
jobs[]:                  Present only when a filter is named or `include_items=True`:
  unit_kind, animal, session, dataset, pipeline, job_name, specifier, cores, memory_mb
  job_id, memory_modeled, prerequisite_ids:   Appended by `detailed=True`
rows, matched_rows, start_row, next_start_row:   The paging keys, alongside `jobs`
```

Listed jobs drop nulls and empty lists, so a session row carries no `dataset` key, a dataset row carries no `animal`
or `session` key, and a job with no upstream stage carries no `prerequisite_ids` key. A `cores` of `0`, a `memory_mb`
of `0`, and a `memory_modeled` of `false` all survive the drop.

The default page is 200 rows, or 50 under `detailed=True`. A `limit` at or below zero lists every match, and a
negative `start_row` clamps to zero.

---

## `inspect_job_resources_tool`

```text
success:                  Boolean flag
pipeline:                 The pipeline argument
host:                     The host argument
units[]:                  One entry per named unit, raw and unprojected, in one of two shapes:
  resolved:               `unit_path`, `unit_name`, `job_count` (dispatchable jobs), and `blocked_count`
  unresolved:             `unit_path`, `unit_name`, `error`, and `job_count: 0`, with NO `blocked_count` key
total_units:              Entries in `units`
totals:
  jobs:                    Dispatchable jobs alone. Blocked jobs are NOT counted
  widest_job_cores:        Widest single-job core count, `0` when there are no jobs
  largest_job_memory_mb:   Largest single-job figure, `0` when there are none
  summed_memory_mb:        Sum over every job, which is NOT what a batch commits at once
breakdown:
  job_name:                Counts per job type across every resolved job, unelided
total_cores:              Cores a batch may commit on this machine. Present ONLY for `host="local"`
total_memory_mb:          This machine's physical memory. Present ONLY for `host="local"`
jobs[]:                   Present only when `job_names` is named or `include_items=True`:
  job_id, job_name, specifier, unit_path, cores, memory_mb
  prerequisite_ids, options:              Appended by `detailed=True`
rows, matched_rows, start_row, next_start_row:   The paging keys, alongside `jobs`
```

`totals` and `breakdown` are computed over every resolved job and ignore `job_names`, which narrows only `jobs`,
`rows`, and `matched_rows`. An empty `job_names` list is a filter matching nothing, so it returns `jobs: []` with
`rows: 0` and `matched_rows: 0`, which is silently different from omitting the argument.

A failure returns this tool's preparation error text, which names preparing rather than inspecting. Never read that
wording as evidence that a batch was registered, because this tool issues no batch identifier.

---

## `read_resource_model_tool`

```text
success:                  Boolean flag
total_cores:              Cores left to a batch on this machine after the reserve
reserved_cores:           Cores held back for host-system operations
total_memory_mb:          This machine's physical memory
total_job_types:          Declared job types, IGNORING the filters
widest_job_cores:         Widest declared allocation, IGNORING the filters
breakdown:
  pipeline:               Counts per pipeline across every declared type, IGNORING the filters
job_types[]:              ALWAYS present, one entry per matched type:
  job_name:               The tracker job name
  pipeline:               The pipeline that dispatches it
  cores:                  The cores one job of this type declares
  concurrency_limit:      The hard ceiling. OMITTED when the type declares none
  concurrency_reservation:   The soft reservation. OMITTED when the type declares none
rows, matched_rows, start_row, next_start_row:   The paging keys
```

Entries are rendered pipeline by pipeline in declaration order, and within each pipeline in the order its stages run.
The default page is 200 rows, since this tool takes no `detailed` argument. Empty `pipelines` or `job_names` lists
match nothing while the summary fields still report the whole model.

An absent `concurrency_limit` means the type is bounded by the core and memory budgets alone. An absent
`concurrency_reservation` means it competes at full width. Neither absence is a value of `0` or a `null`.

---

## Error texts these tools return

| Tool                         | Cause                                    | Exact text                                                                                                                     |
|------------------------------|------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Any tool taking `host`       | A host outside the two labels            | `Unsupported host '<host>'. Available: local, remote.`                                                                         |
| Both plan tools              | No unit named                            | `No processing unit was named.`                                                                                                |
| Both plan tools              | The planning pass raised                 | `Unable to plan the <host> units. <exception>`                                                                                 |
| `generate_project_plan_tool` | The projection raised                    | `Unable to project the plans under '<project_path>'. <exception>`                                                              |
| `read_project_plan_tool`     | A mirror or resolve failure              | `Unable to read the <host> project. <exception>`                                                                               |
| `read_project_plan_tool`     | No projection on disk                    | `No plan projection exists at '<plan_path>'. Plan the project's units, then run generate_project_plan_tool before reading it.` |
| `read_project_plan_tool`     | A filter names a value no row holds      | `No planned job has '<column>' in <unknown>. Available: <available>.`                                                          |
| `read_project_plan_tool`     | A filter names a column the table lacks  | `Unknown column '<column>'. Available: <sorted columns>.`                                                                      |
| `read_resource_model_tool`   | A named pipeline is not a batch pipeline | `No batch pipeline is named <unknown>. Available: <available>.`                                                                |
| `read_resource_model_tool`   | A named type is not declared             | `No job type is named <unknown>. Available: <sorted names>.`                                                                   |
| `read_resource_model_tool`   | A dispatched type declares no width      | `Unable to report the declared resource model. <exception>`                                                                    |

A plan call spanning two projects is refused by the project resolver, and its text is returned with no wrapper prefix
in front of it.
