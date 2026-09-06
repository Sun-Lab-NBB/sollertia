---
name: job-planning
description: >-
  Sizes every runnable job of a session or a dataset and records the figures a batch and a scheduler are sized against.
  Covers per-unit plan caches, the project plan projection, the declared resource model, and pre-batch resource
  inspection. Use when planning a project's jobs, when reading planned cores and memory, when a job reports as unplanned
  or unsized, or when sizing a local budget or a remote submission.
user-invocable: false
---

# Job planning

Reads a unit's acquisition data, registers every runnable job on that unit's tracker, and records each job's cores,
memory, and upstream jobs into a per-unit `job_plan.yaml` that the project projection then gathers into one table.

This skill is the **exclusive** owner of `plan_session_jobs_tool`, `plan_dataset_jobs_tool`,
`generate_project_plan_tool`, `read_project_plan_tool`, `inspect_job_resources_tool`, and `read_resource_model_tool`.
No other skill in the marketplace may document or call these six tools.

Every return key each of the six carries, with the condition under which it is present, lives in one reference
file loaded on demand: [references/tool-responses.md](references/tool-responses.md).

---

## Scope

**Covers:**
- Per-unit planning of sessions and datasets, and the plan cache each one writes
- The `regenerate_plan` flag and the resource-model version stamp that overrides it
- The project plan projection, its `PROJECT_PLAN_SCHEMA` columns, and how to read them back with filters
- The declared resource model: per-job-type core allocations, hard ceilings, soft reservations, and `RESERVED_CORES`
- Pre-batch inspection of the cores and memory a pipeline's outstanding jobs will need
- Sizing refusals, unplanned jobs, and what each one costs the batch that follows

**Does not cover:**
- Preparing, executing, monitoring, resetting, or cleaning a batch. Owned by `/batch-processing`.
- Submitting a batch to the scheduler and reading its allocations. Owned by `/remote-execution`.
- The compute-server record a `host="remote"` call resolves through. Owned by `/server-configuration`.
- The project manifest and the project job artifact that record what each job has since done. Owned by `/project-state`.
- Building the dataset hierarchy whose forging jobs this skill plans. Owned by `/dataset-definition`.
- Producing the session paths a plan call consumes. Owned by `assets:session-discovery`.
- The per-system stage schemas the sizing models read through the registries. Owned by
  `mesoscope:mesoscope-vr-processing-schema`.
- MCP server connectivity and the plugin-wide response contract. Owned by `/forging-mcp-environment-setup`.

**Handoff rules:** Planning ends at the figures. Invoke `/batch-processing` to prepare and execute a local batch
against them, `/remote-execution` to submit one to a scheduler, and `/project-state` to read what each job recorded.

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for every planning operation. Do not import the orchestration entry
points such as `planning.resolve_session_plan` or `planning.generate_project_plan` directly, and do not run `slf plan`
yourself. If the MCP tools are unavailable, invoke `/forging-mcp-environment-setup` to diagnose and resolve
connectivity. The `slf` CLI is the human path, and `/cli-reference` owns it.

You MUST have confirmed session paths from `assets:session-discovery` before calling `plan_session_jobs_tool`, and a
dataset root that `/dataset-definition` built before calling `plan_dataset_jobs_tool`. Do not guess, infer, or
discover unit paths from within this skill.

You MUST leave `regenerate_plan` at its default of `False` unless the user asks for a deliberate retune, because a
prepared batch or a live submission may already have been sized against the figures a cache holds.

You MUST read the declared core allocations, ceilings, and reservations from `read_resource_model_tool` rather than
quoting them from memory or from this skill. Several allocations are resolved at import time from the installed
dependency that owns the stage, so they change with a dependency bump and no edit in this library.

---

## Available tools

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

### Planning tools

```python
plan_session_jobs_tool(session_paths: list[str], host: str = "local", *, regenerate_plan: bool = False)
plan_dataset_jobs_tool(dataset_paths: list[str], host: str = "local", *, regenerate_plan: bool = False)
```

Both delegate to one private helper and differ only in the parameter name and the unit kind they pass.

| Parameter                         | Type        | Default    | Description                                                      |
|-----------------------------------|-------------|------------|------------------------------------------------------------------|
| `session_paths` / `dataset_paths` | `list[str]` | (required) | Unit root directories. Paths ON THE SERVER when `host="remote"`. |
| `host`                            | `str`       | `"local"`  | `"local"` or `"remote"`.                                         |
| `regenerate_plan`                 | `bool`      | `False`    | Keyword-only. Re-estimates figures a cache already holds.        |

A response reports `host`, `total_units`, `total_jobs`, `elapsed_seconds`, and a `units` list whose entries carry
each unit's `unit_path`, `unit_name`, `job_count`, and `summed_memory_mb`, or its `unit_path` and the `error` that
stopped it. See [references/tool-responses.md](references/tool-responses.md) for every key and its condition.

**Note:** One unit's failure never aborts the others, so read every entry rather than the totals alone.

**Note:** `unsized_jobs` is a **local-only** diagnostic. A remote plan reconstructs its per-unit summary out of the
projection rather than out of the plan cache, so a refusal recorded on the server carries no reason back here.

**Note:** Both tools ALWAYS rewrite the project projection, because the per-unit loop is followed by an unconditional
projection pass. A plan call is therefore also a `generate_project_plan_tool` call, and calling both is redundant.

**Note:** Every named unit must belong to one project, since the plan and state artifacts are written per project.
A call spanning two projects is refused before any unit is read.

### Projection tools

```python
generate_project_plan_tool(project_path: str, host: str = "local")
```

Runs the projection alone, over an empty unit list, then reads the resulting table back and reports its totals.

A response reports `plan_path`, `total_jobs`, `summed_memory_mb`, `largest_job_memory_mb`, `widest_job_cores`, and a
`pipeline_totals` list holding one entry per unit kind and pipeline pair.

**Note:** The per-pipeline count key is `jobs`, not `job_count`. The two planning tools use `job_count` per unit, and
mixing the two names silently reads a missing key.

**Note:** This tool does not mirror a remote project onto this machine. `project_path` is interpreted on the target
host, because it writes there rather than reading a copy.

**Note:** A unit carrying no plan cache contributes no rows. Treat a job absent from the projection as **unplanned**,
never as free, and plan its unit before preparing anything against the table.

```python
read_project_plan_tool(
    project_path: str, host: str = "local", unit_kind: str | None = None, animal: str | None = None,
    dataset: str | None = None, pipelines: list[str] | None = None, job_names: list[str] | None = None,
    limit: int | None = None, start_row: int = 0, *, include_items: bool = False, detailed: bool = False,
)
```

A bare call reports `plan_path`, the same four totals `generate_project_plan_tool` returns, and a `breakdown` over
`unit_kind`, `animal`, `dataset`, `pipeline`, and `job_name`. It lists **no** jobs. Naming any filter, or passing
`include_items=True`, adds the `jobs` page. `detailed=True` appends `job_id`, `memory_modeled`, and `prerequisite_ids`
to each listed job, and narrows the default page from 200 rows to 50.

**Note:** The totals and the breakdown span every planned job whatever the filters say, so narrowing the listing never
distorts the figures a submission is sized against.

**Note:** A read with no projection on disk returns `No plan projection exists at '<plan_path>'. Plan the project's
units, then run generate_project_plan_tool before reading it.` Plan the units rather than re-reading.

### Inspection tools

```python
inspect_job_resources_tool(
    pipeline: str, session_paths: list[str], options: dict[str, Any] | None = None, host: str = "local",
    job_names: list[str] | None = None, limit: int | None = None, start_row: int = 0,
    *, include_items: bool = False, detailed: bool = False,
)
```

Reports what one pipeline's outstanding jobs would cost on the named units, running none of them. `pipeline` takes one
of the six batch pipelines. `session_paths` names session roots for every session pipeline and dataset roots for
`forging`. `options` carries exactly one key across the whole library, `regenerate_checksum`, which only the `checksum`
pipeline reads.

A response reports a `totals` block of `jobs`, `widest_job_cores`, `largest_job_memory_mb`, and `summed_memory_mb`,
a `breakdown` per job type, and a `units` list. It adds `total_cores` and `total_memory_mb` for `host="local"`, and
a `jobs` page with the paging keys whenever a filter is named or the listing is requested.

**Note:** This tool is **not** read-only. Discovery runs exactly as it does for a batch, so the call primes every
pipeline of every named unit that declares a priming step, plans every named unit, creates and aligns each unit's
tracker, and rewrites the project's plan and state artifacts. Do not describe it as free and do not poll it. Its
`replan` is hard-wired to `False`, so it never re-estimates a cached figure.

**Note:** `totals.jobs` counts the dispatchable jobs alone, and the response carries no top-level blocked total. Each
resolved `units[]` entry does carry a `blocked_count`, and summing that column across the resolved entries gives the
batch's whole blocked figure, so the outstanding work is `totals.jobs` plus that sum. `/batch-processing` owns the
per-job blocked listing.

**Note:** `summed_memory_mb` sums every job rather than naming a peak, so it is not a budget. `totals` and `breakdown`
ignore `job_names`, which narrows only the listing. An empty `job_names` list is a filter matching nothing, which is
silently different from omitting the argument.

**Note:** `total_cores` and `total_memory_mb` are absent for `host="remote"`, deliberately, because they would describe
this workstation rather than the compute node. A remote caller names what each job requests and the scheduler enforces
its own limits.

```python
read_resource_model_tool(
    pipelines: list[str] | None = None, job_names: list[str] | None = None,
    limit: int | None = None, start_row: int = 0,
)
```

Reports the width, the ceiling, and the reservation every job type declares, alongside this machine's capacity. It
takes no `host`, no `detailed`, and no `include_items`, and the listing is always present.

A response reports this machine's `total_cores`, `reserved_cores`, and `total_memory_mb`, then `total_job_types`,
`widest_job_cores`, a per-pipeline `breakdown`, and a `job_types` list carrying each type's `job_name`, `pipeline`,
`cores`, and the `concurrency_limit` and `concurrency_reservation` it declares.

**Note:** This is the one genuinely read-only tool of the six. Nothing on disk is touched and no tracker is written,
so reading it costs nothing and answers what a stage declares without planning a unit.

**Note:** An absent `concurrency_limit` means "bounded by the core and memory budgets alone", and an absent
`concurrency_reservation` means "competes at full width". An absent key is not a value of `0` and not a `null`.

**Note:** `total_cores`, `reserved_cores`, and `total_memory_mb` always describe THIS machine, since the tool takes no
host. Use them to size a local budget only, never to size a remote submission.

---

## What a plan records

Planning is the expensive half of the lifecycle, because estimation reads each unit's raw acquisition data, opening
video containers and image headers. That is why it is a call of its own rather than a step inside preparation.

| Unit kind | Pipelines sized                                   | Cache location                           |
|-----------|---------------------------------------------------|------------------------------------------|
| session   | every member of `shared_assets.SESSION_PIPELINES` | `<session>/processed_data/job_plan.yaml` |
| dataset   | `ProcessingPipelines.FORGING` alone               | `<dataset root>/job_plan.yaml`           |

The cache is a `planning._JobPlan` carrying `unit_name`, `unit_kind`, `model_version`, an `entries` list, and an
`unsized_jobs` map. Each `planning._JobPlanEntry` holds `pipeline`, `job_name`, `specifier`, `cores`, `memory_mb`,
`resident_mb`, `memory_modeled`, and `prerequisite_ids`, and its `job_id` is derived from the job name and the
specifier.

Sizing runs **before** any tracker write, so a job the plan cannot cover never gets registered at all.
`planning._align_tracker` then registers the runnable and planned jobs and retires a refused job by withholding it
from the universe, unless the tracker already records it under one of `planning._RETAINED_STATUSES`.

The cache is additive and sticky. A unit already carrying a plan keeps every recorded figure and is only extended with
jobs the cache does not hold, so re-running a plan call is cheap and safe.

**The `regenerate_plan` flag.** Recorded entries are reused only when three conditions all hold: a cache exists,
`regenerate_plan` is `False`, and the recorded `model_version` equals what `footprints.resolve_model_version()`
answers now. A cache stamped with a different model version is treated as absent and re-estimated whatever
`regenerate_plan` asks, so a dependency bump or a retune adopts itself without a flag.

**Planning primes before it sizes.** A pipeline whose job model lives in state a dependency writes has that state
materialized first, so planning a session carrying two-photon data writes its cindra configuration and per-plane
bootstrap before anything is sized. Priming is idempotent, but it is a write. A priming failure drops that whole
pipeline from the plan and is recorded as a pipeline skip, not as an `unsized_jobs` entry, so on a unit whose other
pipelines planned normally it leaves no trace in the tool response at all. It surfaces only in the message refusing a
unit no pipeline could plan, and in the `slf plan` console echo.

---

## The project plan projection

The projection is the cheap half of planning and the half that ships. It reads the caches alone, estimates nothing,
and gathers both the sessions and the datasets under a project root into a single `<project stem>_plan.feather` table
written at that root. Pull that file to size a submission without reading the data root it was planned against.

`planning.PROJECT_PLAN_SCHEMA` fixes one row per planned job:

| Column             | Polars dtype   | Meaning                                                               |
|--------------------|----------------|-----------------------------------------------------------------------|
| `unit_kind`        | `String`       | `"session"` or `"dataset"`                                            |
| `animal`           | `String`       | Null on a dataset row                                                 |
| `session`          | `String`       | Null on a dataset row                                                 |
| `dataset`          | `String`       | Null on a session row                                                 |
| `pipeline`         | `String`       | The pipeline that dispatches the job                                  |
| `job_id`           | `String`       | The tracked identifier, which joins the project job artifact directly |
| `job_name`         | `String`       | The tracker job name, which is the stage                              |
| `specifier`        | `String`       | The differentiator within the unit                                    |
| `cores`            | `UInt16`       | The cores this job occupies                                           |
| `memory_mb`        | `UInt32`       | The anonymous memory the sizing pass modeled                          |
| `resident_mb`      | `UInt32`       | That figure plus the pages the job maps, which SLURM is given         |
| `memory_modeled`   | `Boolean`      | Whether a model of the job's own input produced the figure            |
| `prerequisite_ids` | `List(String)` | The jobs that must succeed first, from the pipeline's own ordering    |

Rows are natural-sorted by `unit_kind`, `animal`, `session`, `dataset`, `pipeline`, `job_name`, and `specifier`, with
nulls last. `animal` is a `String` column here as it is in every other project artifact, so the plan joins the
manifest, the jobs table, and a dataset state table on animal and session without casting.

**`memory_modeled` is the assertion that no row carries a guessed figure.** The plan records an entry only for a job
its sizing pass actually modeled, so every projected row answers `true`. A job whose input the pass could not read is
left out of the plan entirely rather than written at a default size.

**`prerequisite_ids` carries the dependency graph in the table itself**, so a scheduler builds the submission order
from this one artifact without reopening any unit. Listing responses drop nulls and empty lists, so a session row
carries no `dataset` key, a dataset row carries no `animal` or `session` key, and a job with no upstream stage carries
no `prerequisite_ids` key at all.

---

## The resource model

**Read the figures live.** `read_resource_model_tool` reports every declared allocation, ceiling, and reservation at
zero disk cost, so quote its response rather than any number written down elsewhere. Several allocations are resolved
at **import time** from the installed dependency that owns the stage, which means they change with a dependency bump
and no edit in `orchestration/dispatch.py`. A figure copied into a report is stale the moment a dependency moves.

| Term             | Declared in                              | Behavior                                                             |
|------------------|------------------------------------------|----------------------------------------------------------------------|
| Core allocation  | `dispatch._JOB_CORE_ALLOCATIONS`         | The cores one job of the type occupies. Read via `resolve_job_cores` |
| Hard ceiling     | `dispatch._JOB_CONCURRENCY_LIMITS`       | Jobs of the type that may run at once, whatever capacity is idle     |
| Soft reservation | `dispatch._JOB_CONCURRENCY_RESERVATIONS` | Capacity offered to other work first, then released over the rest    |
| Host reserve     | `local.RESERVED_CORES`                   | Cores withheld from an auto-resolved core budget                     |

A **ceiling stands however much capacity is idle**. Spare cores and spare memory never lift it, because a type holding
one is waiting on something capacity does not supply, such as storage bandwidth or decoder throughput.

A **reservation binds only while other jobs can take the room it leaves**. Admission runs a first pass honoring every
reservation and a second pass without them, so a reserved type widens toward its full parallelism rather than idling
the host. A type may declare both, and its ceiling then stands in both passes while its reservation stands in the
first alone. A type declaring neither is bounded by the core and memory budgets alone, which is the normal case and
is never an error.

`RESERVED_CORES` holds cores back for host-system operations, and it applies **only to a non-positive core budget**.
An explicit budget is honored up to the machine's logical core count. `read_resource_model_tool` reports the constant
as `reserved_cores` and reports what remains as `total_cores`.

**A job type declaring neither a core allocation nor a sizing model is a hard error, never a job admitted at a default
size.** Four assets raise it, at two points in the lifecycle:

| Asset                            | When it fires                       | What it means                                           |
|----------------------------------|-------------------------------------|---------------------------------------------------------|
| `dispatch.resolve_job_cores`     | Planning, per unit                  | The type declares no entry in the core allocation table |
| `footprints.size_session_jobs`   | Planning, per session job           | The type routes to no sizing model                      |
| `footprints.size_dataset_jobs`   | Planning, per dataset job           | The type routes to no sizing model                      |
| `local.resolve_core_allocations` | Local execution, over the whole set | Names every unregistered type at once, sorted           |

The refusal text is the same claim each time: a job whose resources nothing resolves cannot be admitted to a batch.
Adding a stage is therefore an edit to the allocation table and the sizing router together, and `/library-extension`
owns that seam.

---

## How sizing works

Every figure is per job, never per type. Two jobs of one type legitimately differ in `cores` and `memory_mb`, because
each is estimated from the data it will actually process and a long recording is not charged the same as a short one.

- **A stage this library owns** reports its declared width and a memory figure modeled from that job's own input, such
  as an archive's message count, a prediction table's row and column counts, or a decoded frame's pixel count.
- **A stage a dependency owns** takes both the cores and the memory that dependency's own sizing pass picked for the
  job's input. The two archive extraction stages carry one addition, because a dependency models the children its
  stage spawns against its own interpreter while a pool opened here re-imports this package, so the difference is
  charged for each child the stage opens. The allocation table restates the dependency's width rather than deciding
  it, and it acts as a **cap** on that type rather than a width every job of the type is raised to.
- **No stage answers with a floor.** A job whose input cannot be read raises, and the refusal is recorded in
  `unsized_jobs` while every job beside it stays planned.
- **Every figure reaches one scale.** `footprints._round_to_gigabyte` raises each result to a whole gigabyte, and the
  package's own models additionally carry a tolerance before that rounding, so the reported memory is what to request.
- **Every job carries two memory figures.** `memory_mb` is the anonymous memory the job allocates, and it is the term
  a local process pool is budgeted against. `resident_mb` adds the bytes the job memory-maps and the shared library
  image every job holds resident, carries its own margin over that sum, and is the term each SLURM allocation
  requests. Only the two-photon stages that hold a plane binary open, which are `binarization`, `registration`, and
  `processing`, and the forging pipeline's `multiday_extraction` stage map anything, so every other job's resident
  figure is its anonymous one raised by the shared image alone. Every reported total counts `memory_mb` alone, so a
  remote submission is sized from the per-job `resident_mb` a listing carries rather than from the totals beside it.

`footprints.resolve_model_version()` digests every private uppercase constant in `footprints.py` into a
twelve-character identifier, and that identifier is stamped into each plan cache. Retuning any one of those constants
changes the identifier and invalidates every cache carrying the old one, which is what makes a retune adopt itself.

---

## Planning workflow

### Execution model

Planning uses a **size-then-project** model:

1. **Plan** primes every pipeline of the named unit that declares a priming step, sizes the unit, writes its
   `job_plan.yaml`, and aligns its trackers. This is the pass that reads acquisition data, and it is idempotent: a unit
   already planned keeps its figures and gains only the jobs it lacks.

2. **Project** gathers every plan cache under the project root into one feather table. This reads caches alone, so it
   is cheap enough to re-run at will, and the two planning tools already run it for you.

### Workflow steps

1. **Confirm the units.** Take session paths from `assets:session-discovery` and dataset roots from
   `/dataset-definition`. Every unit of one call must sit under the same project root.

2. **Read the declared model first.** Call `read_resource_model_tool` bare. It costs nothing, names this machine's
   `total_cores`, `reserved_cores`, and `total_memory_mb`, and gives the per-type widths and ceilings you will quote
   later. Skip this only for a `host="remote"` run, where the figures describe the wrong hardware.

3. **Plan the units.** Call `plan_session_jobs_tool` or `plan_dataset_jobs_tool` with `regenerate_plan` left at
   `False`. Present `total_units`, `total_jobs`, and `elapsed_seconds` back to the user.

4. **Reconcile every unit entry.** Account for each entry carrying an `error`, and for each `unsized_jobs` map, before
   treating the plan as complete. A unit reporting `job_count: 0` planned nothing and will resolve nothing later.

5. **Read the projection.** Call `read_project_plan_tool` bare for the totals and the breakdown, then name a filter or
   pass `include_items=True` to list the jobs themselves. Add `detailed=True` when you need `prerequisite_ids`.

6. **Inspect the pipeline about to run.** Call `inspect_job_resources_tool` for that one pipeline and those units, and
   read `totals` and `breakdown`. Warn the user that this call writes plan caches, trackers, and project artifacts.

7. **Present the figures.** Render the plan table below, and quote `widest_job_cores` and `largest_job_memory_mb`
   against the machine capacity from step 2 so the user sees whether any single job exceeds the host.

8. **Hand off.** Route a local run to `/batch-processing` and a scheduler run to `/remote-execution`. Neither can size
   anything the projection does not already hold.

---

## Plan presentation

When presenting a plan to the user, format the per-pipeline rollup as a table and close with the machine capacity:

```text
**Project Plan**

Totals: 38 jobs | 139264 MB summed | widest job 16 cores | largest job 12288 MB

| Unit kind | Pipeline        | Jobs | Summed memory (MB) | Widest cores |
|-----------|-----------------|------|--------------------|--------------|
| dataset   | forging         | 6    | 24576              | 16           |
| session   | checksum        | 8    | 16384              | 8            |
| session   | video           | 24   | 98304              | 16           |

Host capacity: 14 cores available, 2 reserved, 65536 MB physical memory.
```

Report `summed_memory_mb` as the sum it is, never as a budget, since nothing commits every job at once. A single job
whose `memory_mb` exceeds the host's physical memory is the figure to raise, because the local engine admits an
oversized job on an idle pool rather than refusing it.

---

## Sizing refusals and error routing

A refused job is left out of the plan and reported in `unsized_jobs`, never fatal to its unit. The map key names the
pipeline and the job, and the value is the reason the pass gave.

| `unsized_jobs` key shape         | What it means                                                                       |
|----------------------------------|-------------------------------------------------------------------------------------|
| `<pipeline>/<job_name> (<spec>)` | That one job's sizing raised, or the pass answered with no footprint and no refusal |
| `<pipeline>/<job_name>`          | The same, for a stage whose specifier is empty, so the key carries no parentheses   |
| `<pipeline>/all jobs`            | The one-pass sizing for the whole pipeline raised, so each job was then sized alone |

| Symptom                                                    | Cause                                                      | Resolution                             |
|------------------------------------------------------------|------------------------------------------------------------|----------------------------------------|
| A unit entry carries `error` and `job_count: 0`            | No pipeline planned any job for it                         | Check the unit carries readable inputs |
| `unsized_jobs` names a job whose input file is missing     | The stage's input was never produced                       | Run the upstream pipeline first        |
| A pipeline's jobs are absent with no `unsized_jobs` entry  | Its priming or its job discovery raised, so it was skipped | Ask the user for the `slf plan` echo   |
| A prepared batch reports a unit as carrying unplanned jobs | The projection has no row for those ids                    | Plan that unit, then prepare again     |
| `read_project_plan_tool` reports no projection             | No unit under the root has been planned                    | Plan the units, which reprojects       |
| A cache is re-estimated despite `regenerate_plan=False`    | The stamped model version no longer matches                | Expected. Adopt the new figures        |

A unit that no pipeline could plan is refused with a message naming what each pipeline reported and what each sizing
pass refused. Read both halves: the first says the unit carries none of the data a pipeline consumes, and the second
says it carries the data in a state no model could read.

**Why planning must precede a remote submission.** The plan table is the only source of the cores and memory each
allocation requests. Preparation refuses a project whose host holds no plan table. A job present in the state artifact
but absent from the plan makes its whole unit unresolved, on the ground that every outstanding job must be planned
before it can be sized for a host or a scheduler.

---

## Related skills

The `video:`, `communication:`, and `cindra:` entries below resolve through the ataraxis and cindra marketplaces.
Every other entry resolves inside the sollertia marketplace.

| Skill                                      | Relationship                                                                |
|--------------------------------------------|-----------------------------------------------------------------------------|
| `/forging-mcp-environment-setup`           | Prerequisite: MCP server connectivity and the plugin-wide response contract |
| `/batch-processing`                        | Downstream: prepares and executes the batch these figures size              |
| `/remote-execution`                        | Downstream: submits a batch whose allocations come from the plan table      |
| `/dataset-definition`                      | Upstream: builds the dataset hierarchy whose forging jobs are planned here  |
| `/project-state`                           | Peer: records what each planned job has since done                          |
| `/processing-input-format`                 | Reference: what a unit must carry before a stage can be sized at all        |
| `/data-processing-design`                  | Context: the durable admission and tracker doctrine these figures feed      |
| `/library-extension`                       | Reference: the allocation table and sizing router a new stage must declare  |
| `/cli-reference`                           | Reference: the human-facing `slf plan` command surface                      |
| `/pipeline`                                | Context: where planning sits in the end-to-end pipeline                     |
| `assets:session-discovery`                 | Upstream: the exclusive producer of the session paths a plan call consumes  |
| `mesoscope:mesoscope-vr-processing-schema` | Reference: the per-system stage schemas the sizing models read              |
| `video:log-processing`                     | Reference: the library whose own sizing pass answers for camera extraction  |
| `communication:log-processing`             | Reference: the library whose own sizing pass answers for archive extraction |
| `cindra:single-recording-processing`       | Reference: the library whose resource classes supply several declared terms |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Planning:
- [ ] sollertia-forgery MCP server is connected
- [ ] Unit paths came from assets:session-discovery or /dataset-definition, not from a guess
- [ ] plan_session_jobs_tool or plan_dataset_jobs_tool ran before anything read the projection
- [ ] regenerate_plan was left False, or the user explicitly asked for a retune
- [ ] Every units[] entry was reconciled, including each error and each unsized_jobs map
- [ ] A job absent from the projection was reported as unplanned rather than as free
- [ ] A pipeline missing from a plan with no unsized_jobs entry was reported as a skip, not as a unit with no such data

Resource model:
- [ ] Core allocations, ceilings, and reservations were read from read_resource_model_tool, never quoted from memory
- [ ] An absent concurrency_limit or concurrency_reservation was read as undeclared, not as zero
- [ ] total_cores, reserved_cores, and total_memory_mb were used to size a local budget only
- [ ] summed_memory_mb was presented as a sum, never as a budget
- [ ] inspect_job_resources_tool was described as writing trackers and artifacts, not as read-only

Boundaries:
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
- [ ] Batch preparation, execution, and cancellation were routed to /batch-processing
- [ ] Scheduler submission and allocation reads were routed to /remote-execution
```
