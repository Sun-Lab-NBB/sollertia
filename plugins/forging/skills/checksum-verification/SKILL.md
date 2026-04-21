---
name: checksum-verification
description: >-
  Orchestrates batch checksum verification and regeneration via the sollertia-forgery MCP server:
  batch preparation, job execution, progress monitoring, cancellation, retry, cleanup, and
  project-wide status overview. Use when verifying or regenerating data integrity checksums for
  one or more sessions, checking checksum status across a project, or managing checksum tracker
  lifecycle.
user-invocable: true
---

# Checksum verification

Orchestrates the batch checksum verification workflow: prepare execution manifests from confirmed
session paths, dispatch jobs for background execution, monitor progress, and clean up tracker
artifacts. Each session produces exactly one checksum resolution job.

---

## Scope

**Covers:**
- Batch checksum verification for one or more sessions
- Checksum regeneration (overwrite stored checksums instead of verifying)
- Background job execution with budget-bounded concurrency
- Progress monitoring (per-job status and timing)
- Cancellation, failed job reset, and retry
- Project-wide checksum status overview
- Tracker cleanup after completion

**Does not cover:**
- Session discovery and filtering (see the assets plugin's `/session-discovery` — required prerequisite)
- Manifest generation or reading (see `/project-manifest`)
- Session transfer or deletion (see `/session-transfer`)
- Behavior processing (see `/behavior-processing`)
- MCP server connectivity (see `/forging-mcp-environment-setup`)

---

## Agent requirements

You MUST use the sollertia-forgery MCP tools for all checksum operations. Do not import
`sollertia_forgery.managing.checksum` directly.

You MUST have confirmed session paths from the assets plugin's `/session-discovery` before calling
`prepare_checksum_batch_tool`. Do not guess or derive paths manually.

You MUST respect the single-execution-session constraint: only one checksum batch may run at a
time per `sl-mcp` process. Cancel any active session before starting a new batch.

---

## Available tools

### Preparation and execution

| Tool                            | Purpose                                                            |
|---------------------------------|--------------------------------------------------------------------|
| `prepare_checksum_batch_tool`   | Creates execution manifest without starting execution (idempotent) |
| `execute_checksum_jobs_tool`    | Dispatches prepared jobs for background execution                  |

**`prepare_checksum_batch_tool` parameters:**

| Parameter         | Type          | Default      | Description                                                      |
|-------------------|---------------|--------------|------------------------------------------------------------------|
| `session_paths`   | `list[str]`   | (required)   | Session root paths from the assets plugin's `/session-discovery` |

Each session produces exactly one job: `checksum_resolution` with the session name as the
specifier. The tracker is created at `{session_root}/raw_data/checksum_processing_tracker.yaml`.
Idempotent — calling on sessions with existing trackers returns current state instead of
reinitializing.

**Return structure:**

```text
success:            True when preparation completed
sessions{}:         Per-session manifest keyed by session_path:
  tracker_path:     Path to checksum_processing_tracker.yaml
  session_name:     Human-readable session name
  jobs[]:           Job descriptors (feed directly to execute tool):
    job_id:         Hexadecimal job identifier
    job_name:       "checksum_resolution"
    specifier:      Session name
    status:         SCHEDULED / SUCCEEDED / FAILED / RUNNING
    session_path:   Session root path (dispatch metadata)
    session_name:   Session name (dispatch metadata)
    tracker_path:   Tracker file path (dispatch metadata)
  summary{}:        total / succeeded / failed / running / scheduled
total_sessions:     Count of processed sessions
total_jobs:         Total job count across all sessions
invalid_paths:      Paths that could not be resolved (optional)
```

**`execute_checksum_jobs_tool` parameters:**

| Parameter             | Type         | Default | Description                                                     |
|-----------------------|--------------|---------|-----------------------------------------------------------------|
| `jobs`                | `list[dict]` | (req)   | Job descriptors from the prepare manifest                       |
| `workers_per_job`     | `int`        | `-1`    | CPU cores per checksum job; `-1` for saturating allocation      |
| `max_parallel_jobs`   | `int`        | `-1`    | Max concurrent jobs; `-1` for saturating allocation             |
| `regenerate_checksum` | `bool`       | `False` | Overwrite stored checksums instead of verifying when `True`     |

The tool distributes the CPU budget across concurrent jobs using saturating allocation. When both
parameters are `-1`, the allocator maximizes parallelism at the preferred per-job worker count,
reduces parallelism when workers would drop below a minimum floor, and enforces a hard cap of
20 cores per job. Each job runs in a separate subprocess with its resolved worker count bound
at dispatch time — the process pool is sized to `max_parallel_jobs`, and each subprocess
internally spawns its own worker pool sized to `workers_per_job`.

Four resolution scenarios are supported:

| `workers_per_job` | `max_parallel_jobs` | Behavior                                                  |
|-------------------|---------------------|-----------------------------------------------------------|
| `-1` (auto)       | `-1` (auto)         | Saturating allocation resolves both from CPU budget       |
| fixed (>0)        | `-1` (auto)         | Workers capped at 20; parallel jobs derived from capacity |
| `-1` (auto)       | fixed (>0)          | Workers derived from budget / parallel, capped at 20      |
| fixed (>0)        | fixed (>0)          | Both used directly; workers still capped at 20            |

### Monitoring

| Tool                         | Purpose                                                   |
|------------------------------|-----------------------------------------------------------|
| `get_checksum_status_tool`   | Per-job status of the active execution session            |
| `get_checksum_timing_tool`   | Per-job timing and session-level throughput               |

Neither tool takes parameters. Both read from the in-memory execution state plus on-disk trackers.

`get_checksum_status_tool` returns `active` (bool), `canceled` (bool), per-job `status` entries,
and a `summary` with counts for each status category.

`get_checksum_timing_tool` returns per-job `started_at`, `elapsed_seconds` (running jobs),
`duration_seconds` (completed jobs), and session-level `total_elapsed_seconds` and
`throughput_jobs_per_hour`.

### Lifecycle

| Tool                                     | Purpose                                                     |
|------------------------------------------|-------------------------------------------------------------|
| `cancel_checksum_tool`                   | Clears pending queue; active jobs complete naturally        |
| `reset_checksum_jobs_tool`               | Resets specific or all jobs to `SCHEDULED` for retry        |
| `clean_checksum_tracker_tool`            | Deletes tracker files and companion lock files              |

**`reset_checksum_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                             |
|----------------|--------------------|------------|---------------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to `checksum_processing_tracker.yaml`     |
| `job_ids`      | `list[str] / None` | `None`     | Job IDs to reset; if omitted, every job is reset        |

**`clean_checksum_tracker_tool` parameters:**

| Parameter       | Type        | Default    | Description                                              |
|-----------------|-------------|------------|----------------------------------------------------------|
| `session_paths` | `list[str]` | (required) | Session root paths whose checksum trackers to delete     |

Loads `SessionData` for each session to resolve `raw_data_path`, then removes the tracker YAML
and its `.lock` file. Refuses to run while an execution session is active.

**Do not call this tool unless the user explicitly requests tracker cleanup.** The tracker YAML
is the on-disk source of truth consumed by `generate_project_manifest_tool` to populate the
`integrity` flag. Because the manifest uses a dependency cascade (`integrity=false` forces
`cindra`, `behavior`, and `video` to `false` regardless of their own tracker state), deleting
checksum trackers before the manifest is generated produces a manifest that misrepresents the
entire project as unprocessed. Preserve trackers by default; only clean on explicit user request,
and prefer to do so after `/project-manifest` has been generated.

### Project-wide overview

| Tool                                        | Purpose                                             |
|---------------------------------------------|-----------------------------------------------------|
| `get_checksum_batch_status_overview_tool`   | Aggregates checksum status across all sessions      |

**Parameters:**

| Parameter        | Type  | Default    | Description                                           |
|------------------|-------|------------|-------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to search recursively for tracker YAMLs |

Discovers all `checksum_processing_tracker.yaml` files under the root and aggregates per-session
status. Does not require an active execution session — reads directly from on-disk trackers.

---

## Verification workflow

### Pre-run checklist

```text
- [ ] Sessions discovered via the assets plugin's /session-discovery (session_paths confirmed with user)
- [ ] User confirmed which sessions to verify
- [ ] No active checksum session (get_checksum_status_tool → active: false)
- [ ] Resource allocation decision made with user (default -1/-1 for saturating auto)
```

### Workflow steps

1. **Prepare batch** — Call `prepare_checksum_batch_tool` with confirmed `session_paths`. Inspect
   for `invalid_paths` and existing tracker states.

2. **Present manifest** — Show the job list to the user:

   ```text
   **Checksum Batch** — 5 sessions, 5 jobs

   | Session                          | Status    |
   |----------------------------------|-----------|
   | 2026-03-04-14-30-00-000000       | SCHEDULED |
   | 2026-03-05-10-15-00-000000       | SCHEDULED |
   | 2026-03-06-09-00-00-000000       | SUCCEEDED |
   ```

   Sessions with existing trackers showing `SUCCEEDED` can be skipped unless the user wants
   re-verification.

3. **Confirm resource allocation** — Present the default allocation (`-1` = saturating auto for
   both `workers_per_job` and `max_parallel_jobs`). On constrained systems, reduce
   `workers_per_job` or `max_parallel_jobs` to limit memory usage.

4. **Flatten and execute** — Collect all job descriptors from `sessions[*].jobs` into a flat list.
   Call `execute_checksum_jobs_tool` with the flat list and confirmed settings.

5. **Monitor** — Call `get_checksum_status_tool` periodically until all jobs reach terminal state.
   Optionally call `get_checksum_timing_tool` for elapsed time and throughput.

6. **Handle completion:**
   - All `SUCCEEDED` → **leave trackers in place** and regenerate the project manifest via
     `/project-manifest`. Do NOT call `clean_checksum_tracker_tool` unless the user explicitly
     requests it; the manifest reads these trackers to set `integrity=true`, and cleaning them
     first cascades `behavior`, `cindra`, and `video` to `false` in the manifest output
     regardless of their actual processing state.
   - Some `FAILED` → inspect `error_message`, reset with `reset_checksum_jobs_tool`, re-prepare,
     and re-execute
   - Active session died → cancel, clean, re-prepare from scratch

### Regeneration variant

To regenerate checksums instead of verifying, set `regenerate_checksum=True` in step 4. This
overwrites the stored checksum values with freshly computed ones.

---

## Status formatting

When presenting batch status, format as a table:

```text
**Checksum Verification Status**

Summary: 3/5 jobs complete | 1 running | 1 scheduled | 0 failed

| Session                          | Status    | Duration |
|----------------------------------|-----------|----------|
| 2026-03-04-14-30-00-000000       | SUCCEEDED | 45.2s    |
| 2026-03-05-10-15-00-000000       | SUCCEEDED | 38.7s    |
| 2026-03-06-09-00-00-000000       | SUCCEEDED | 41.1s    |
| 2026-03-07-11-00-00-000000       | RUNNING   | 12.3s    |
| 2026-03-08-15-45-00-000000       | SCHEDULED | --       |
```

For project-wide overview via `get_checksum_batch_status_overview_tool`:

```text
**Checksum Overview** — /data/projects/my_project

| Session path                              | Status    | Succeeded | Failed | Total |
|-------------------------------------------|-----------|-----------|--------|-------|
| /data/projects/my_project/animal_1/sess_1 | completed | 1         | 0      | 1     |
| /data/projects/my_project/animal_1/sess_2 | failed    | 0         | 1      | 1     |
| /data/projects/my_project/animal_2/sess_1 | scheduled | 0         | 0      | 1     |
```

---

## Error routing

| Error                                          | Resolution                                                          |
|------------------------------------------------|---------------------------------------------------------------------|
| `An execution session is already active`       | Wait for completion or call `cancel_checksum_tool` first            |
| `No valid jobs to execute`                     | Verify job descriptors have required keys                           |
| `Tracker file not found`                       | Re-prepare the batch to regenerate trackers                         |
| `Session path does not exist`                  | Verify path exists; re-run the assets plugin's `/session-discovery` |
| `Unable to load session`                       | Check that `session_data.yaml` exists at the session root           |
| Clean refused (session still active)           | Wait for completion or cancel before cleaning                       |
| Per-job failure                                | Inspect `error_message`; reset and retry                            |
| MCP tool unavailable                           | Invoke `/forging-mcp-environment-setup`                             |

---

## Related skills

| Skill                                | Relationship                                                         |
|--------------------------------------|----------------------------------------------------------------------|
| `/forging-mcp-environment-setup`     | Prerequisite: MCP server connectivity                                |
| assets plugin `/session-discovery`   | Prerequisite: provides confirmed session_paths                       |
| `/project-manifest`                  | Downstream: regenerate manifest after verification completes         |
| `/session-transfer`                  | Peer: verify integrity before transferring sessions                  |
| `/behavior-processing`               | Peer: behavior processing depends on verified integrity              |

---

## Verification checklist

```text
Checksum Verification:
- [ ] Verified MCP server connectivity (invoked /forging-mcp-environment-setup if unavailable)
- [ ] Received confirmed session_paths from the assets plugin's /session-discovery
- [ ] Prepared batch and reviewed manifest
- [ ] Confirmed resource allocation with user (workers_per_job / max_parallel_jobs)
- [ ] Verified no active checksum session before dispatch
- [ ] Executed jobs and monitored until all reached terminal state
- [ ] Investigated and retried failed jobs if needed
- [ ] Preserved tracker files (DO NOT clean unless the user explicitly requested it — trackers feed the project manifest)
- [ ] Regenerated project manifest via /project-manifest to update integrity status
```
