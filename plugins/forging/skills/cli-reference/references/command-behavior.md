# Command behavior and failure modes

Per-command behavior of the `slf` CLI, the exception each failure raises, and the exit code the user observes.

---

## How a failure reaches the user

No command body is wrapped in a catching decorator, so every failure the body raises leaves a Python traceback and a
non-zero exit. The message text identifies the fault, so ask the user to paste the traceback rather than the status.

| Path                                                                       | Mechanism                                 | Observable outcome                         |
|----------------------------------------------------------------------------|-------------------------------------------|--------------------------------------------|
| `slf omp` finding no runtime to link                                       | `raise SystemExit(1)`                     | Exit 1, the only deliberate non-zero exit  |
| A missing required option, a rejected choice, or a path that must exist    | Click's own parameter validation          | Usage message, exit 2                      |
| `-sp` omitted before a `process` subcommand, `-pp` before a `manifest` one | `click.UsageError` from the shared object | Usage message naming the omission, exit 2  |
| Any guard or pipeline error on every other command                         | The exception propagates uncaught         | Traceback, non-zero exit                   |
| `slf plan session` or `slf plan dataset` meeting an unplannable unit       | Caught per path inside the loop           | `<name>: planned nothing. <error>`, exit 0 |
| `slf reset` or `slf clean` meeting an unloadable unit                      | Warning logged, the unit is skipped       | Exit 0                                     |

`slf dataset-state` catches nothing, so one bad dataset path aborts the whole loop partway through. There is no
version, verbosity, or global host option anywhere: `slf` addresses this machine, and the remote host is reachable only
through `slf server`.

---

## The macOS OpenMP precondition

`shared_assets/openmp.py::verify_openmp_runtime` runs at the head of every pipeline except `manifest`, so on macOS
`slf process`, `slf forge`, and `slf checksum` raise `RuntimeError` naming `slf omp` as the remedy before doing any work
when `libomp.dylib` does not load. The MCP server reports healthy on such a host while every parallel job fails, which
makes this a first-order environment diagnostic. A bare `slf omp` is always a dry run, and writing the link usually runs
through sudo. The `## OpenMP runtime prerequisite` section of `/forging-mcp-environment-setup` owns its four outcomes.

---

## The job identifier contract on `slf process` and `slf forge`

| Case                  | `-id`    | Behavior                                                                      |
|-----------------------|----------|-------------------------------------------------------------------------------|
| Local mode            | omitted  | Discovers and runs every available job for the unit, honoring the stage flags |
| Remote mode           | supplied | Runs only the matching job, chosen entirely by the identifier                 |
| `slf process runtime` | either   | Always runs its single job. The identifier is never forwarded                 |

`interfaces/process.py::runtime_command` does not pass the shared job identifier to
`runtime/pipeline.py::run_runtime_processing_pipeline`, which declares no such parameter, so `-id` is silently ignored
there. Local mode on `slf forge` skips every succeeded stage, while remote mode runs the named job regardless.

---

## `slf manifest print`

Three guards fire in order, and each raises uncaught. Giving neither `-n` nor `-s` raises `ValueError`. No manifest on
disk raises `FileNotFoundError` naming the generation command, because printing reads an existing snapshot and never
regenerates one. An animal that did not participate raises `ValueError`. Both views print together, notes first.

---

## `slf clean` holds no running-batch guard

`clean_processing_output_tool` refuses while a batch is running locally, and `slf clean` performs no such check, so it
will remove output from under a running batch. Confirm no batch is executing before handing the command over. Each
removed path is reported as the bytes it held, a single space, then the path. Cleaning `checksum` removes its tracker
alone, since that pipeline owns no output directory, while cleaning `forging` removes the whole dataset hierarchy.

---

## The four `slf server` reporting commands

`slf server print` requires at least one of `-j` and `-q`, raising `ValueError` when neither is given, and prints both
views in one call when both are given, accounting first. Both views read `-id`, and only `-j` reads `-st` and `-et`,
which `-q` accepts and ignores. A non-zero `sacct` or `squeue` return raises `RuntimeError` quoting the scheduler's own
error stream, and an empty result logs a warning and returns. `slf server discover` prints one line per discovered
session, reporting no forged dataset and no absolute path, so it cannot supply the paths a `host="remote"` tool takes.

`slf server batches` prints a `batch_id | progress | outstanding_h | jobs | running | stranded | gone | pipelines`
table. In it, `outstanding_h` reads in hours or `unknown`, while `running`, `stranded`, and `gone` count the allocations
carrying each resolved verdict, and `progress` is one of `progressing`, `stalled`, and `awaiting_closure`. A
`scheduler_read_error` prints first as a WARNING, and `-a` adds a
`batch_id | allocation | job_name | specifier | scheduler | tracker | verdict | remediation` table beneath it. Each
stalled batch then prints one WARNING line naming the allocations both scheduler records disclaim and the remedy, which
names `slf server retire-batch`. A read that closes every batch it covered prints that message as a warning and stops.

`slf server retire-batch` prints the snapshot error if any, then one SUCCESS line per retired batch with the allocations
it held. An `allocation | job_name | specifier | verdict | canceled | reset | snapshot | dropped` table follows, one
row per allocation, with the four booleans naming which steps ran. The message and the local outcome directory print
last. Where their tool answers with an error rather than a report, both commands raise `RuntimeError` through
`console.error` and it propagates uncaught, which is how a `running` allocation without `--force`, and a failed snapshot
without `--drop-without-outcome`, reach the user.
