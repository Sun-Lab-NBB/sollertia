# Mesoscope-VR session data lifecycle

Enumerates the `preprocess_session_data` step order, the Mesoscope-VR-only preprocessing steps, and the session purge
and animal migration paths. See [`../SKILL.md`](../SKILL.md) for the state machine, the orchestrator, the runtime GUIs,
and the workflow for adding a new runtime mode.

---

## Preprocessing order

`preprocess_session_data(session_data)` (`mesoscope_vr/data_preprocessing.py`) is the Mesoscope-VR orchestration around
the shared primitives that `experiment:data-management` owns. A session whose `nk.bin` marker survives never finished
initialization, so it is purged instead of preprocessed, and each destination listed in
`MesoscopeData.unconfigured_destinations` produces one WARNING.

| Order | Step                                                              | Owner                       |
|-------|-------------------------------------------------------------------|-----------------------------|
| 1     | `rename_mesoscope_directory(mesoscope_data)`                      | Mesoscope-VR                |
| 2     | `assemble_session_logs(session_data, processes=...)`              | `cross_system`              |
| 3     | `rename_session_videos(session_data)`                             | `cross_system`              |
| 4     | `_launch_face_tracking(...)`, experiment sessions only, async     | Mesoscope-VR                |
| 5     | `_pull_mesoscope_data(...)`                                       | Mesoscope-VR                |
| 6     | `_preprocess_mesoscope_directory(...)`                            | Mesoscope-VR                |
| 7     | `_preprocess_google_sheet_data(...)`                              | Mesoscope-VR Sheets wrapper |
| 8     | `_purge_window_checking_behavior_data(...)`, window checking only | Mesoscope-VR                |
| 9     | `_join_face_tracking(...)`                                        | Mesoscope-VR                |
| 10    | `push_session_data(session_data, destinations, threads=15)`       | `cross_system`              |

Steps 5 through 8 run inside a `try` whose `except BaseException` calls `_terminate_face_tracking(...)` and re-raises,
so an abort never abandons the child holding the GPU. Constants: `_PREPROCESSING_WORKER_COUNT` is
`resolve_worker_count(reserved_cores=1)`, `_STORAGE_TRANSFER_THREAD_COUNT = 15`,
`_FACE_TRACKING_TERMINATION_TIMEOUT = 30.0` seconds, and `_INFERENCE_LOG_TAIL_CHARACTERS = 2000`.

---

## Mesoscope-VR-only steps

- `rename_mesoscope_directory` renames the shared `mesoscope_data` directory to the session-specific path, only when the
  session path is absent and the shared path holds files, then recreates an empty shared directory.
- `_launch_face_tracking` returns `None` when either `conda_environment` or `dlc_project_path` is unset, or when the
  face-camera video is missing. Otherwise it runs `conda run -n <env> slvt infer` with `--config-path`, `--videos`,
  `--shuffle`, `--device cuda`, `--gpus 0`, `--batch-size`, `--chunks`, `--compile-model`, `--no-progress`, and `--crop`
  when configured. It writes predictions beside the video in raw `camera_data` and redirects output to a temporary log
  file rather than a pipe, because a full pipe buffer would deadlock the long-running child.
- `_join_face_tracking` waits for the child, then raises `RuntimeError` when the exit code is non-zero or no `.h5`
  prediction file sits beside the video, which aborts the transfer and retains the local copy for a retry. The transient
  log is removed on success and retained on failure, and the failure message carries its tail.
- `_pull_mesoscope_data` raises `RuntimeError` unless `MotionEstimator.me`, `fov.roi`, and `zstack.tiff` are all
  present, strips `*.bin` markers, creates `raw_data/raw_mesoscope_frames` only after that verification, and then
  transfers with `remove_source=True`.
- `_preprocess_mesoscope_directory` re-verifies the same three files, seeds the animal's persistent ScanImagePC
  `fov.roi` and `MotionEstimator.me` when absent, copies all three into the session `mesoscope_data` directory, and
  emits `frame_invariant_metadata.json`, `frame_variant_metadata.npz`, and `cindra_parameters.json` alongside the
  LERC-recompressed frame stacks.
- `_preprocess_google_sheet_data` returns early with a WARNING when neither sheet id is set, otherwise resolves
  `get_credentials(CredentialsTypes.GOOGLE)`, validates the session type against `MESOSCOPE_VR_SESSIONS`, and loads the
  descriptor through `DESCRIPTOR_REGISTRY`. Window-checking sessions call `update_surgery_quality` with the descriptor
  value clamped into 0 to 3, and every other session type writes the water log entry from the animal weight and the
  summed training and experimenter-given volumes. Both handles close in a `finally`.

---

## Purge and migration

`purge_session(session_data)` builds its candidate set from the local session parent, every configured storage
destination's session path, and the ScanImagePC session-specific path, then delegates to
`delete_session_directories(..., require_confirmation=not nk_path.exists())`. A declined confirmation returns without
further change, and a completed deletion also clears residual files from the shared ScanImagePC `mesoscope_data`
directory.

`migrate_animal_between_projects(animal, source_project, target_project)` raises `FileNotFoundError` when the target
project is absent, then picks one of two strategies. With no configured storage destination it relocates each locally
stored session on premises. With at least one, the first configured destination becomes the source of truth, and each
session is pulled from that destination, re-preprocessed, and purged against it. Both strategies then relocate the
ScanImagePC persistent directory at `mesoscope_directory/<project>/<animal>` and the VRPC persistent directory, and
delete the redundant `<root>/<source_project>/<animal>` directory under the mesoscope mount, the data root, and every
configured storage root.

`sle mesoscope delete` runs the data-root containment check and then delegates to `purge_session` (the `delete` command
in `interfaces/mesoscope_vr.py`). It therefore prompts for an interactive confirmation on any session whose `nk.bin`
marker is already cleared, and deletes without a prompt only for a session that never finished initializing. The MCP
`delete_session_tool` refuses to act without an explicit `confirm_deletion`, returning an `Error:` string when it is
`None` and an abandonment notice when it is `"no"` (`interfaces/mesoscope_vr_tools.py`). You MUST route an
agent-initiated deletion through the MCP tool and warn the user before passing `"yes"`.
