---
name: session-transfer
description: >-
  Orchestrates batch session transfer and deletion via the sollertia-forgery MCP server: batch
  preparation, job execution, progress monitoring, cancellation, retry, and cleanup. Use when
  transferring sessions to an archive location, deleting sessions, or managing transfer tracker
  lifecycle. Requires confirmed session paths from /session-discovery.
user-invocable: true
---

# Session transfer

Orchestrates the batch session transfer and deletion workflow: build job descriptors from confirmed
session paths, prepare execution manifests, dispatch jobs for background execution, monitor
progress, and clean up tracker artifacts.

---

## Scope

**Covers:**
- Batch session transfer (copy to destination, optionally remove source)
- Batch session deletion (remove source with no destination)
- Background job execution with budget-bounded concurrency
- Progress monitoring (per-job status and timing)
- Cancellation, failed job reset, and retry
- Tracker cleanup after completion

**Does not cover:**
- Session discovery and filtering (see `/session-discovery` — required prerequisite)
- Manifest generation or reading (see `/project-manifest`)
- Checksum verification (see `/checksum-verification`)
- Behavior processing (see `/behavior-processing`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for all transfer and deletion operations. Do not
import `sollertia_forgery.managing.transfer` directly.

You MUST have confirmed session paths from `/session-discovery` before calling
`prepare_transfer_batch_tool`. Do not guess or derive paths manually.

You MUST confirm the `tracker_directory` with the user. The tracker must be placed in a stable
directory that will NOT be affected by the transfer or deletion operations — typically the project
root or a parent directory of the sessions being moved.

You MUST request a secondary confirmation from the user before executing any batch that contains
deletion jobs. Present the full list of sessions marked for deletion and explicitly ask the user
to confirm that permanent removal is intended. Do not proceed on the initial confirmation alone.

You MUST respect the single-execution-session constraint: only one transfer batch may run at a
time per `sl-mcp` process. Cancel any active session before starting a new batch.

---

## Job types

Each job in the batch is either a **transfer** or a **deletion**, determined by the combination
of `destination_path` and `remove_source` in the job descriptor:

| `destination_path` | `remove_source`  | Job type  | Job name           | Effect                        |
|--------------------|------------------|-----------|--------------------|-------------------------------|
| Present            | `"false"` / omit | Transfer  | `session_transfer` | Copy to destination, keep src |
| Present            | `"true"`         | Transfer  | `session_transfer` | Copy to destination, rm src   |
| Absent             | `"true"`         | Deletion  | `session_deletion` | Remove source directory       |
| Absent             | `"false"` / omit | Invalid   | —                  | Error: no operation specified |

**Job descriptor fields for `prepare_transfer_batch_tool`:**

| Key                | Type  | Required | Description                                                        |
|--------------------|-------|----------|--------------------------------------------------------------------|
| `source_path`      | `str` | Yes      | Absolute path to the session root directory                        |
| `destination_path` | `str` | No       | Absolute path to the destination directory (omit for deletion)     |
| `remove_source`    | `str` | No       | String `"true"` or `"false"`; controls source removal              |

---

## Available tools

### Preparation and execution

| Tool                            | Purpose                                                            |
|---------------------------------|--------------------------------------------------------------------|
| `prepare_transfer_batch_tool`   | Creates execution manifest without starting execution (idempotent) |
| `execute_transfer_jobs_tool`    | Dispatches prepared jobs for background execution                  |

**`prepare_transfer_batch_tool` parameters:**

| Parameter           | Type         | Default    | Description                                                       |
|---------------------|--------------|------------|-------------------------------------------------------------------|
| `jobs`              | `list[dict]` | (required) | Transfer job descriptors (see Job types above)                    |
| `tracker_directory` | `str`        | (required) | Stable directory for `transfer_processing_tracker.yaml`           |

The tracker is created at `{tracker_directory}/transfer_processing_tracker.yaml`. Unlike checksum
trackers (which live inside each session), the transfer tracker is placed in a caller-specified
directory so it survives session deletion. Idempotent — calling with an existing tracker returns
current state.

**Return structure:**

```text
tracker_path:       Path to transfer_processing_tracker.yaml
jobs[]:             Job descriptors enriched with dispatch metadata:
  job_id:           Hexadecimal job identifier
  job_name:         "session_transfer" or "session_deletion"
  specifier:        Session name from SessionData
  status:           SCHEDULED / SUCCEEDED / FAILED / RUNNING
  source_path:      Source session root path
  session_name:     Human-readable session name
  destination_path: Destination path (transfer jobs only)
  remove_source:    Whether source is removed after transfer
  tracker_path:     Tracker file path (for dispatch)
summary{}:          total / succeeded / failed / running / scheduled
invalid_jobs:       Jobs that failed validation (optional)
```

**`execute_transfer_jobs_tool` parameters:**

| Parameter       | Type         | Default | Description                                                  |
|-----------------|--------------|---------|--------------------------------------------------------------|
| `jobs`          | `list[dict]` | (req)   | Job descriptors from the prepare manifest                    |
| `worker_budget` | `int`        | `-1`    | CPU cores for the session; `-1` for automatic resolution     |

Each job runs in a separate subprocess with `workers=1`. The worker budget controls how many
transfers or deletions run in parallel. Automatic resolution subtracts 2 reserved cores.

### Monitoring

| Tool                       | Purpose                                        |
|----------------------------|------------------------------------------------|
| `get_transfer_status_tool` | Per-job status of the active execution session |
| `get_transfer_timing_tool` | Per-job timing and session-level throughput    |

Neither tool takes parameters. Both read from the in-memory execution state plus on-disk trackers.

`get_transfer_status_tool` returns `active` (bool), `canceled` (bool), per-job `status` entries,
and a `summary` with counts for each status category.

`get_transfer_timing_tool` returns per-job `started_at`, `elapsed_seconds` (running jobs),
`duration_seconds` (completed jobs), and session-level `total_elapsed_seconds` and
`throughput_jobs_per_hour`.

### Lifecycle

| Tool                          | Purpose                                              |
|-------------------------------|------------------------------------------------------|
| `cancel_transfer_tool`        | Clears pending queue; active jobs complete naturally |
| `reset_transfer_jobs_tool`    | Resets specific or all jobs to `SCHEDULED` for retry |
| `clean_transfer_tracker_tool` | Deletes tracker file and companion lock file         |

**`reset_transfer_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                                |
|----------------|--------------------|------------|------------------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to `transfer_processing_tracker.yaml`        |
| `job_ids`      | `list[str] / None` | `None`     | Job IDs to reset; if omitted, every job is reset           |

**`clean_transfer_tracker_tool` parameters:**

| Parameter      | Type  | Default    | Description                                                     |
|----------------|-------|------------|-----------------------------------------------------------------|
| `tracker_path` | `str` | (required) | Absolute path to `transfer_processing_tracker.yaml` to delete   |

Removes the tracker YAML and its `.lock` file. Refuses to run while an execution session is
active.

---

## Transfer workflow

### Pre-run checklist

```text
- [ ] Sessions discovered via /session-discovery (session_paths confirmed with user)
- [ ] User confirmed which sessions to transfer or delete
- [ ] For transfers: destination directory confirmed with user
- [ ] Tracker directory confirmed with user (must survive the operation)
- [ ] No active transfer session (get_transfer_status_tool → active: false)
- [ ] Worker budget decision made with user (default -1 for auto)
```

### Workflow steps

1. **Discover sessions** — Use `/session-discovery` to obtain confirmed session paths.

2. **Confirm operation details** — Ask the user:
   - Which sessions to include?
   - Transfer or deletion for each?
   - For transfers: what is the destination directory?
   - Should the source be removed after transfer?

3. **Confirm tracker directory** — The tracker must be in a stable location not affected by the
   operations. Good choices:
   - Project root directory (when transferring or deleting a subset of sessions)
   - Parent directory of the project (when operating on the entire project)
   - Never place the tracker inside a session being transferred or deleted

4. **Build job descriptors** — Construct the `jobs` list:

   ```json
   [
     {
       "source_path": "/data/project/animal_1/session_1",
       "destination_path": "/archive/animal_1/session_1",
       "remove_source": "true"
     },
     {
       "source_path": "/data/project/animal_1/session_2",
       "remove_source": "true"
     }
   ]
   ```

   The first entry is a transfer (move); the second is a deletion.

5. **Prepare batch** — Call `prepare_transfer_batch_tool` with the jobs list and confirmed
   tracker directory. Inspect for `invalid_jobs`.

6. **Present manifest** — Show the job list:

   ```text
   **Transfer Batch** — 5 jobs (3 transfers, 2 deletions)

   | Session                    | Type     | Destination                        | Status    |
   |----------------------------|----------|------------------------------------|-----------|
   | 2026-03-04-14-30-00-000000 | Transfer | /archive/animal_1/session_1        | SCHEDULED |
   | 2026-03-05-10-15-00-000000 | Transfer | /archive/animal_1/session_2        | SCHEDULED |
   | 2026-03-06-09-00-00-000000 | Transfer | /archive/animal_2/session_1        | SCHEDULED |
   | 2026-03-07-11-00-00-000000 | Deletion | —                                  | SCHEDULED |
   | 2026-03-08-15-45-00-000000 | Deletion | —                                  | SCHEDULED |
   ```

7. **Secondary deletion confirmation** — If the batch contains any deletion jobs, present them
   separately and explicitly ask the user to confirm permanent removal before proceeding. Do not
   combine this with the manifest review in step 6.

8. **Execute** — Collect job descriptors into a flat list and call `execute_transfer_jobs_tool`.

9. **Monitor** — Call `get_transfer_status_tool` periodically. Use `get_transfer_timing_tool`
   for throughput estimates on large batches.

10. **Handle completion:**
   - All `SUCCEEDED` → clean tracker, then regenerate manifest via `/project-manifest`
   - Some `FAILED` → inspect `error_message`, reset with `reset_transfer_jobs_tool`, re-prepare,
     and re-execute
   - Active session died → cancel, clean, re-prepare from scratch

---

## Status formatting

```text
**Transfer Status**

Summary: 3/5 jobs complete | 1 running | 1 scheduled | 0 failed

| Session                          | Type     | Status    | Duration |
|----------------------------------|----------|-----------|----------|
| 2026-03-04-14-30-00-000000       | Transfer | SUCCEEDED | 120.5s   |
| 2026-03-05-10-15-00-000000       | Transfer | SUCCEEDED | 95.3s    |
| 2026-03-06-09-00-00-000000       | Transfer | SUCCEEDED | 108.7s   |
| 2026-03-07-11-00-00-000000       | Deletion | RUNNING   | 2.1s     |
| 2026-03-08-15-45-00-000000       | Deletion | SCHEDULED | --       |
```

---

## Error routing

| Error                                            | Resolution                                        |
|--------------------------------------------------|---------------------------------------------------|
| `An execution session is already active`         | Wait or call `cancel_transfer_tool` first         |
| `No valid transfer jobs to prepare`              | Verify job descriptors have `source_path`         |
| `No valid jobs to execute`                       | Verify job descriptors have all required keys     |
| `Source path does not exist`                     | Verify path; re-run `/session-discovery`          |
| `Unable to load session`                         | Check `session_data.yaml` at the session root     |
| `No destination_path ... remove_source not true` | Specify destination or set `remove_source="true"` |
| `Tracker file not found`                         | Re-prepare the batch to regenerate the tracker    |
| Clean refused (session still active)             | Wait for completion or cancel before cleaning     |
| Per-job failure                                  | Inspect `error_message`; reset and retry          |
| MCP tool unavailable                             | Invoke `/forging-mcp-environment-setup`           |

---

## Related skills

| Skill                            | Relationship                                                     |
|----------------------------------|------------------------------------------------------------------|
| `/forging-mcp-environment-setup` | Prerequisite: MCP server connectivity                            |
| `/session-discovery`             | Prerequisite: provides confirmed session_paths                   |
| `/checksum-verification`         | Peer: verify integrity before transferring                       |
| `/project-manifest`              | Downstream: regenerate manifest after transfer or deletion       |

---

## Verification checklist

```text
Session Transfer:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Received confirmed session_paths from /session-discovery
- [ ] Confirmed operation type (transfer vs deletion) for each session
- [ ] For transfers: confirmed destination directory with user
- [ ] Confirmed tracker directory (stable, outside affected sessions)
- [ ] Prepared batch and reviewed manifest with user
- [ ] For deletions: obtained secondary confirmation for permanent removal
- [ ] Confirmed worker budget with user
- [ ] Verified no active transfer session before dispatch
- [ ] Executed jobs and monitored until all reached terminal state
- [ ] Investigated and retried failed jobs if needed
- [ ] Cleaned tracker file after completion
- [ ] Regenerated project manifest via /project-manifest
```
