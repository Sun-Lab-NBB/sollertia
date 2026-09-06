# Resource model

The core allocation, hard ceiling, and soft reservation every job type declares, the host core reserve, and the
assets that refuse a job type declaring neither a core allocation nor a sizing model.

---

## Declared terms

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

---

## Unregistered job types

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
