# Experiment configuration authoring patterns

Goal-to-pattern recipes for the recurring experiment configuration edits, plus the walkthrough for moving an existing
experiment onto a new task template. See [`../SKILL.md`](../SKILL.md) for the generic configuration contract, the MCP
tool surface, the registry dispatch, the authoring workflow, and the verification checklist.

---

## Common patterns

| Goal                                 | Pattern                                                                                                                                                                                                                                                                                             |
|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reuse a template across projects     | Call `create_experiment_from_vr_template_tool` per project (one `file_path` per destination), then override per-project fields                                                                                                                                                                      |
| Change a per-trial parameter         | Edit the relevant field on `trial_structures["<trial>"]`, then read the system's schema skill for the trial class's fields                                                                                                                                                                          |
| Adjust a state's duration            | Edit `experiment_states["<state-key>"].state_duration_s` (state machine is a dict)                                                                                                                                                                                                                  |
| Add a new state to the state machine | Add a new key to the `experiment_states` dict, then re-validate                                                                                                                                                                                                                                     |
| Add a new spatial trial entry        | First hand off to `/task-templates` to add the `TrialStructure` to the template, then either re-run `create_experiment_from_vr_template_tool` with `overwrite=True` or amend this skill's experiment config via `write_experiment_configuration_tool` to add the matching runtime trial-class entry |

### Moving an experiment to a new template

1. Read the old configuration with `read_experiment_configuration_tool(file_path=...)`.
2. If the new template does not exist, hand off to `/task-templates` to author it.
3. Call `create_experiment_from_vr_template_tool(file_path=..., template_path=...)` pointing at the new template.
4. Port the customizations (state durations, per-trial runtime parameters, guidance counters) over manually.
