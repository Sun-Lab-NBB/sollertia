# Generation failure modes

Every symptom `CreateTask` can produce, its root cause, and its resolution. The `/task-generator` skill links here
rather than carrying the table inline, because the table is a lookup rather than part of the generation workflow.

`CreateTask.CreateFromTemplate` returns a string prefixed `success: ` or `error: `. Each Symptom below that carries the
prefix quotes that raw form, which is what `read_console_tool` returns for the `CreateTask → New Task` menu path.
`McpBridge.GenerateTask` strips the prefix and trims the remainder into the tool's `error` field, so `create_task_tool`
callers match the unprefixed text that `/task-prefabs` Troubleshooting lists.

| Symptom                                                                              | Root cause                                                                         | Resolution                                                                                      |
|--------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| `error: Cross-template cue-texture conflict detected`                                | Two templates declare the same `(cue name, length_cm)` with different textures     | Rename the cue, change its length, or unify the textures, then re-run                           |
| `error: Cross-template cue-texture preflight aborted: failed to load '<file>'`       | Another template in the catalog no longer loads, and the preflight loads every one | Fix the named template, since generation is blocked for all until it loads                      |
| `error: Template filename '<name>' is invalid.`                                      | The filename stem carries something outside `[A-Za-z0-9_]`                         | Rename the YAML file, because the segment name reserves the hyphen as its only separator        |
| `error: Cue '<name>' references texture '<t>' but no file found at <path>`           | Load-time check: the file is absent from `Textures/`                               | Hand the texture off to the user for import, then re-run                                        |
| `BuildCuePrefabs: Failed to load texture '<t>'`                                      | On disk but not imported into the AssetDatabase as a `Texture2D`                   | Import it with `refresh_assets_tool`, then re-run                                               |
| `BuildCuePrefabs: Cue '<name>' at <n> cm ... built from a different texture`         | The cached `Cue_<name>_<n>cm.mat` was built from a different texture               | Delete both cue assets to rebuild them, or re-key the cue by name or length                     |
| `error: Generation requires hand-authored assets that are missing from the project:` | `ValidateHandAuthoredAssets`: a hand-authored material or base prefab is absent    | Restore every named path from version control                                                   |
| `error: Template '<name>' declares a longest segment of <n> Unity units, ...`        | The default track length cannot fill `segments_per_corridor`                       | Shorten the longest `cue_sequence`, lower `segments_per_corridor`, or raise `cm_per_unity_unit` |
| `error: Failed to build segment prefabs.`                                            | A cue prefab for a `cue_sequence` is missing, or a `trigger_type` has no branch    | Read the preceding `BuildSegmentPrefabs:` error with `read_console_tool`                        |
| `error: No segment found at <path>`                                                  | The segment prefab was written but could not be loaded back                        | Call `refresh_assets_tool` to reimport, then re-run                                             |
| Segment length warning in Console                                                    | Cue lengths do not sum to the measured prefab length                               | Either regenerate the segment or fix template cues                                              |
| Zone geometry looks wrong in scene view                                              | Template's cm values or `cm_per_unity_unit` mismatch                               | Recheck the YAML, then regenerate                                                               |
| Cue textures appear mirrored on Left or Right wall                                   | Quad scale sign is flipped (Right uses negative X)                                 | Intentional, because each wall shows a correctly-oriented cue                                   |
| Play Mode runs but nothing moves, and a `Task:` error is logged                      | A `Task.cs` startup bailout fired (see [Runtime contract](#runtime-contract))      | Read the error with `read_console_tool`, then apply the fix its Runtime contract bullet names   |

---
