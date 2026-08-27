---
name: mesoscope-vr-cli-reference
description: >-
  Documents the human-facing `sle mesoscope` command group. Covers all fourteen Click nodes and all thirty-three
  options with their short form, long form, type, default, and required or flag status, the MCP tool each command maps
  to, and the per-command failure modes. Use when a user asks what an `sle mesoscope` command or option does, or when
  a run or maintenance command must be prepared for an experimenter. Use it also when the MCP server is unavailable
  and the user must be told what to run by hand.
user-invocable: false
---

# Mesoscope-VR CLI reference

> **The `sle mesoscope` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Print the command, ask the user to run
> it, and ask them to paste the output back.

**The one exemption** is `--help`. `sle mesoscope --help` and `sle mesoscope COMMAND --help` may be run, and no other
`sle mesoscope` invocation is exempt. `experiment:experiment-mcp-environment-setup` owns the `sle --help` smoke test.

---

## Scope

**Covers:**
- The complete `sle mesoscope` command surface: every Click node, its purpose, and the MCP tool it maps to
- Every declared option: short form, long form, type, default, required or flag status, and effect
- Per-command failure modes, the exception each raises, and the guard order inside each command body
- Which commands have no MCP equivalent, which MCP tools have no CLI equivalent, and where paired surfaces differ
- What to tell a user to run when the MCP server cannot be restored in this session

**Does not cover:**
- The system-agnostic `sle` root, the `sle mcp` command, and the `sle get` surface. Owned by
  `experiment:cli-reference`.
- What a runtime does once a command starts: the state machine, the two GUIs, the per-mode logic, threshold seeding,
  and the pre-start checkpoint. Owned by `/mesoscope-vr-runtime`.
- The system configuration YAML field schema the `configure system` command writes. Owned by `/mesoscope-vr`.
- The experiment configuration field schema the `configure experiment` command writes. Owned by
  `/mesoscope-vr-experiment-schema`.
- The session descriptor and hardware-state schemas the descriptor tools carry. Owned by `/mesoscope-vr-session-schema`.
- The Zaber and mesoscope-objective position schemas the four position tools carry. Owned by `/mesoscope-vr-snapshots`.
- The shared preprocessing, transfer, and purge primitives behind `preprocess`, `delete`, and `migrate`. Owned by
  `experiment:data-management`.
- Diagnosing why the MCP server is down, and the response envelope its tools return. Owned by
  `experiment:experiment-mcp-environment-setup`.

**Handoff rules:** If the user wants a configuration or data-management operation performed rather than explained, use
the MCP tool and invoke the owning skill. If the MCP tools are unavailable, invoke
`experiment:experiment-mcp-environment-setup` first, and fall back to the handoff table below only after the server
cannot be restored. Every `run` subcommand and `maintain` is handed to the experimenter regardless of server health.

---

## Agent requirements

You MUST answer CLI questions from this skill or from `sle mesoscope COMMAND --help`, never from memory. When a user's
report disagrees with this reference, ask them to run that help command and read the installed build's answer.

**The `run` subcommands and `maintain` drive real laboratory hardware.** They actuate the water valve, the gas puff
valve, the wheel brake, the Zaber motors, and the mesoscope, with a head-fixed animal on the rig. They are operated by
an experimenter standing at the rig, and every one of them opens a blocking GUI that expects a human at the keyboard.
You MUST explain and prepare these commands, print the exact command line the experimenter types, and never execute
one. This holds even when the user asks you to start a session for them.

Your own path for everything that has an MCP equivalent is the MCP tool. Reach for `check_mesoscope_bridge_tool`,
`preprocess_session_tool`, `delete_session_tool`, and `migrate_animal_tool` rather than printing the paired command,
and print a command line only for a surface no tool covers, or after the server is confirmed unrecoverable.

---

## Command surface

The group declares fourteen Click nodes: the `mesoscope` group, the `configure` and `run` subgroups, and eleven leaf
commands. `interfaces/entry_points.py::_register_subcommands` attaches the group to the top-level `sle` group, so
every node below is reached as `sle mesoscope ...`. Every node lives in `interfaces/mesoscope_vr.py`.

| Click node                           | Kind    | Purpose                                                               | MCP equivalent                    |
|--------------------------------------|---------|-----------------------------------------------------------------------|-----------------------------------|
| `sle mesoscope`                      | group   | Entry point for the system. Dispatches and prints the listing         | None, dispatch only               |
| `sle mesoscope configure`            | group   | Dispatches the two configuration file authoring commands              | None, dispatch only               |
| `sle mesoscope configure system`     | command | Writes the default system configuration file to the working directory | `write_system_configuration_tool` |
| `sle mesoscope configure experiment` | command | Builds an experiment configuration from a named task template         | None                              |
| `sle mesoscope maintain`             | command | Opens the hardware maintenance GUI. Acquires no data                  | None, and none is possible        |
| `sle mesoscope check-bridge`         | command | Probes the ScanImagePC `runAcquisition` MQTT control loop             | `check_mesoscope_bridge_tool`     |
| `sle mesoscope run`                  | group   | Parses the four session identifiers its four subcommands share        | None, dispatch only               |
| `sle mesoscope run window-checking`  | command | Runs the cranial window quality session                               | None, and none is possible        |
| `sle mesoscope run lick-training`    | command | Runs the lick training session                                        | None, and none is possible        |
| `sle mesoscope run run-training`     | command | Runs the run training session                                         | None, and none is possible        |
| `sle mesoscope run experiment`       | command | Runs the named experiment session                                     | None, and none is possible        |
| `sle mesoscope preprocess`           | command | Preprocesses one locally stored session's data                        | `preprocess_session_tool`         |
| `sle mesoscope delete`               | command | Purges one session from every acquisition and storage machine         | `delete_session_tool`             |
| `sle mesoscope migrate`              | command | Moves one animal's sessions from a source project to a target project | `migrate_animal_tool`             |

`configure system` is paired rather than equivalent, because the CLI writes a defaults-only file where the tool writes
a caller-supplied payload. The divergence table below states the difference. See `/mesoscope-vr-runtime` for what each
`run` subcommand and `maintain` then does once the command starts.

**Note on `-h`:** the group sets `context_settings` to `{"max_content_width": 120}` alone, leaving Click's
`help_option_names` at its `["--help"]` default, so `-h` is never a help alias. No node in this group binds `-h` to an
option either, so `-h` aborts on the unknown option at exit code 2. Every node inherits the 120-character content
width from the group's context, so every `--help` rendering wraps at that width.

---

## Option reference

The group declares thirty-three options across eight nodes. "Required" means Click rejects the invocation without the
option at exit code 2. "Optional" with a `None` default means Click accepts the omission and something downstream
decides what the omission means.

### `sle mesoscope configure system`

Declares no options. The command writes the Mesoscope-VR defaults and binds the host-machine to this acquisition
system, with nothing to parameterize.

### `sle mesoscope configure experiment`

| Short | Long                     | Type    | Default    | Form     | Effect                                                              |
|-------|--------------------------|---------|------------|----------|---------------------------------------------------------------------|
| `-p`  | `--project`              | `str`   | (required) | required | The project under whose configuration directory the file is written |
| `-e`  | `--experiment`           | `str`   | (required) | required | The experiment name, used as the stem of the created file           |
| `-t`  | `--template`             | `str`   | (required) | required | The task template filename stem, given without the `.yaml` suffix   |
| `-sc` | `--state-count`          | `int`   | `1`        | optional | The number of default-valued runtime states to generate             |
| none  | `--reward-size`          | `float` | `5.0`      | optional | Default lick-trial water reward volume, in microliters              |
| none  | `--reward-tone-duration` | `int`   | `300`      | optional | Default lick-trial reward tone duration, in milliseconds            |
| none  | `--puff-duration`        | `int`   | `100`      | optional | Default occupancy-trial gas puff duration, in milliseconds          |
| `-f`  | `--force`                | flag    | `False`    | flag     | Replaces an existing configuration file at the destination          |

The three duration and volume options carry no short form at all, so the long form is the only way to give them. Every
defaulted option declares `show_default=True`, so `--help` prints the value above. `-f` reaches the library as its
`overwrite` argument and is the only way to replace an existing file.

### `sle mesoscope maintain` and `sle mesoscope check-bridge`

Neither command declares an option. Both take their entire input from the active system configuration file.

### `sle mesoscope run` group options

| Short | Long              | Type    | Default | Form     | Effect                                                   |
|-------|-------------------|---------|---------|----------|----------------------------------------------------------|
| `-u`  | `--user`          | `str`   | `None`  | optional | The ID of the user supervising the session               |
| `-p`  | `--project`       | `str`   | `None`  | optional | The name of the project the animal belongs to            |
| `-a`  | `--animal`        | `str`   | `None`  | optional | The ID of the animal undergoing the session              |
| `-w`  | `--animal-weight` | `float` | `None`  | optional | The animal's weight in grams at the start of the session |

**All four are parsed on the group**, so they must be supplied **before** the subcommand name, as in
`sle mesoscope run -u <user> -p <project> -a <animal> -w <grams> lick-training`. Click itself requires none of them.
The group callback bundles them into a frozen `_SharedSessionParameters` on the Click context, and each subcommand
demands only what its session consumes, which is why `window-checking` runs without `-w`.

### `sle mesoscope run window-checking`

Declares no options of its own. It consumes `-u`, `-p`, and `-a` from the group, and ignores `-w`.

### `sle mesoscope run lick-training`

| Short  | Long                   | Type    | Default | Form     | Effect                                                           |
|--------|------------------------|---------|---------|----------|------------------------------------------------------------------|
| `-t`   | `--maximum-time`       | `int`   | `None`  | optional | Session cap in minutes. Help text states a 20 minute fallback    |
| `-min` | `--minimum-delay`      | `int`   | `None`  | optional | Minimum seconds between two rewards. Help text states 6 seconds  |
| `-max` | `--maximum-delay`      | `int`   | `None`  | optional | Maximum seconds between two rewards. Help text states 18 seconds |
| `-v`   | `--maximum-volume`     | `float` | `None`  | optional | Water cap in milliliters. Help text states 1.0 mL                |
| `-ur`  | `--unconsumed-rewards` | `int`   | `None`  | optional | Unconsumed reward ceiling. Help text states 1, and `0` lifts it  |

### `sle mesoscope run run-training`

| Short  | Long                   | Type    | Default | Form     | Effect                                                              |
|--------|------------------------|---------|---------|----------|---------------------------------------------------------------------|
| `-t`   | `--maximum-time`       | `int`   | `None`  | optional | Session cap in minutes. Help text states a 40 minute fallback       |
| `-is`  | `--initial-speed`      | `float` | `None`  | optional | Starting speed threshold in cm/s. Help text states 0.8 cm/s         |
| `-id`  | `--initial-duration`   | `float` | `None`  | optional | Starting duration threshold in seconds. Help text states 1.5 s      |
| `-it`  | `--increase-threshold` | `float` | `None`  | optional | Water in mL that triggers a threshold step. Help text states 0.1 mL |
| `-ss`  | `--speed-step`         | `float` | `None`  | optional | Speed threshold step in cm/s. Help text states 0.05 cm/s            |
| `-ds`  | `--duration-step`      | `float` | `None`  | optional | Duration threshold step in seconds. Help text states 0.1 s          |
| `-v`   | `--maximum-volume`     | `float` | `None`  | optional | Water cap in milliliters. Help text states 1.0 mL                   |
| `-mit` | `--maximum-idle-time`  | `float` | `None`  | optional | Below-threshold grace in seconds. Help text states 0.3 s            |
| `-ur`  | `--unconsumed-rewards` | `int`   | `None`  | optional | Unconsumed reward ceiling. Help text states 1, and `0` lifts it     |

**`-id` on this command is `--initial-duration`, a float in seconds.** It is not a job or session identifier, and no
node in this group takes one.

### `sle mesoscope run experiment`

| Short | Long                   | Type  | Default    | Form     | Effect                                                          |
|-------|------------------------|-------|------------|----------|-----------------------------------------------------------------|
| `-e`  | `--experiment`         | `str` | (required) | required | The name of the experiment configuration to carry out           |
| `-ur` | `--unconsumed-rewards` | `int` | `None`     | optional | Unconsumed reward ceiling. Help text states 1, and `0` lifts it |

`-e` is the only required option on any `run` subcommand, and it names a configuration `configure experiment` wrote.

### `sle mesoscope preprocess` and `sle mesoscope delete`

| Short | Long             | Type         | Default    | Form     | Effect                                           |
|-------|------------------|--------------|------------|----------|--------------------------------------------------|
| `-sp` | `--session-path` | `click.Path` | (prompted) | required | The session directory to preprocess or to remove |

Both commands declare the identical option, typed
`click.Path(exists=True, file_okay=False, dir_okay=True, path_type=Path)`, marked `required=True`, and carrying a
`prompt=` string. The `prompt=` means an omitted `-sp` opens an interactive prompt rather than aborting, so the command
blocks on a human instead of exiting 2. Click rejects a path that does not exist and a path that is a file.

### `sle mesoscope migrate`

| Short | Long            | Type  | Default    | Form     | Effect                                            |
|-------|-----------------|-------|------------|----------|---------------------------------------------------|
| `-s`  | `--source`      | `str` | (required) | required | The name of the project the data is migrated from |
| `-d`  | `--destination` | `str` | (required) | required | The name of the project the data is migrated to   |
| `-a`  | `--animal`      | `str` | (required) | required | The ID of the animal whose sessions are migrated  |

`-d` reaches the library as its `target_project` keyword, so a report quoting `target_project` is naming this option.

### Short forms that collide

The same short form means different things in different commands, so quote the long form whenever one is handed to a
user. `-t` expands to `--template` on `configure experiment` and to `--maximum-time` on both training commands. `-s`
expands to `--source` on `migrate`, while `-sc` and `-sp` are separate options on other commands rather than `-s` with
an argument. `-d` expands to `--destination` on `migrate`, while `-ds` is `--duration-step` on `run-training`. `-p`
means `--project` on both `configure experiment` and the `run` group, and `-e` means `--experiment` on both
`configure experiment` and `run experiment`, so those two carry the same sense throughout the group.

---

## Command behavior and failure modes

### How a failure reaches the user

Only `check-bridge` catches anything, so every other failure leaves a Python traceback and a non-zero exit. The message
text identifies the fault, so ask the user to paste the traceback rather than the exit status.

| Path                                                                | Mechanism                                  | Observable outcome                        |
|---------------------------------------------------------------------|--------------------------------------------|-------------------------------------------|
| A missing required option, or a `-sp` path that is absent or a file | Click's own parameter validation           | Usage message, exit 2                     |
| A `run` subcommand missing an identifier the group did not receive  | `click.UsageError` through `console.error` | Usage message naming the omission, exit 2 |
| Any guard inside a command body                                     | `console.error` raises the named class     | Traceback, non-zero exit                  |
| `check-bridge` meeting any exception at all                         | Caught, echoed at WARNING level            | Warning line, exit 0                      |

`console.error(message=..., error=X)` raises `X` after logging it, so every guard named in this section terminates the
command. There is no version option, no verbosity option, and no dry-run option anywhere on this surface.

### `configure system` replaces without asking

The command takes no overwrite guard and no confirmation. It writes the Mesoscope-VR defaults over any existing
Mesoscope-VR configuration file, then removes every other acquisition system's configuration file from the same
directory once that write succeeds, so the host-machine belongs to exactly one system. Re-running it on a configured
rig therefore discards the hand-edited hardware parameters the file held. Warn the user before handing it over, and
prefer `write_system_configuration_tool` with the current payload when the goal is editing rather than initializing.

### `configure experiment` guard order

Three guards fire in a fixed order, and each raises uncaught.

| Order | Condition                                              | Raises              |
|-------|--------------------------------------------------------|---------------------|
| 1     | The named project does not exist under the data root   | `ValueError`        |
| 2     | The destination file exists and `-f` was not given     | `FileExistsError`   |
| 3     | The named task template is not in the templates folder | `FileNotFoundError` |

The existence check precedes the template check, so a wrong `-t` on an experiment that already exists reports the file
collision rather than the bad template name. The template error lists every available template stem, which makes a
deliberate bad `-t` on a fresh experiment name the fastest way to enumerate the installed templates.

### `check-bridge` never reports failure as failure

The command wraps `check_mesoscope_bridge()` in a `try/except Exception`. An exception from the call itself, which
includes a missing or wrong-system configuration file and a broker that refuses the connection, is echoed as a WARNING
and the command returns at exit code 0. A successful probe echoes the returned status at SUCCESS when reachable and at
WARNING when not. The user therefore sees two visually similar warnings for two different faults, so ask for the message
text. An unreachable bridge means the operator has not launched `runAcquisition` on the ScanImagePC.

### The `run` group defers its own requirements

Click requires none of the four group options, so an omission is caught by the subcommand rather than by the parser.
Each `require_*` accessor on `_SharedSessionParameters` raises `click.UsageError` through `console.error` when the
subcommand reads a value the operator omitted, and the message names the exact short and long form to supply. Placing
one of the four after the subcommand name aborts on the unknown option instead, because the subcommand does not declare
it. `window-checking` reads three of the four and never reads `-w`.

### `preprocess` and `delete` are fenced to the local data root

Both commands resolve `get_data_root()` and the supplied path before comparing them, which defeats a `..` segment and
a symlink that would otherwise route an out-of-root session past the check, or refuse an in-root session reached
through a link. A session outside the resolved data root raises `FileNotFoundError` naming both paths. This fences the
commands to sessions on the host-machine, and in particular keeps them off sessions on long-term storage mounts.

`delete` then purges the session from every acquisition machine and every long-term storage destination, with no
confirmation prompt and no undo. Never hand this command over without stating that plainly, and prefer
`delete_session_tool`, which refuses to act until the caller supplies its explicit confirmation value.

### `migrate` requires the target project to exist

The command raises `FileNotFoundError` when the target project directory is absent on the host-machine. Each session
is an isolated unit that cleans up after itself on failure, so re-running the command after fixing the error resumes
from the session that failed rather than starting over.

---

## How the CLI diverges from the MCP path

The response envelope every tool on this server returns, and the tri-state confirmation its destructive tools take,
are documented in the `## Response contract` section of `experiment:experiment-mcp-environment-setup`.

### CLI commands with no MCP equivalent

- **`configure experiment`.** No tool builds an experiment configuration from a task template. The CLI command is the
  only way to create one, and the resulting file's field schema is owned by `/mesoscope-vr-experiment-schema`.
- **`maintain` and the four `run` subcommands.** Each opens a blocking GUI and drives hardware with an animal on the
  rig, so no tool exists and none should. These are the experimenter's commands.

### MCP tools with no CLI equivalent

The Mesoscope-VR tool module registers fifteen tools, of which five pair with a command. The remaining ten have no CLI
surface at all.

| Tool                                        | What the CLI cannot do                                                  |
|---------------------------------------------|-------------------------------------------------------------------------|
| `read_system_configuration_tool`            | Read the active system configuration back as a structured payload       |
| `validate_system_configuration_tool`        | Report the configuration's validity and its per-path mount status       |
| `check_system_mounts_tool`                  | Report every declared filesystem path with a reachable and failed count |
| `verify_camera_configuration_tool`          | Diff each camera's live GenICam nodes against its stored configuration  |
| `describe_system_configuration_schema_tool` | Return the recursive field description of the configuration dataclasses |
| `read_session_zaber_positions_tool`         | Read a session's stored Zaber motor positions                           |
| `write_session_zaber_positions_tool`        | Write a session's Zaber motor positions                                 |
| `read_session_mesoscope_positions_tool`     | Read a session's stored mesoscope positions                             |
| `write_session_mesoscope_positions_tool`    | Write a session's mesoscope positions                                   |
| `read_session_system_configuration_tool`    | Read the per-session snapshot of the system configuration               |

Every structured read of Mesoscope-VR state is therefore MCP-only. The CLI writes state and echoes human status lines,
and reads nothing back.

### Where a paired command and tool differ

| CLI command        | Nearest MCP tool                  | Divergence                                                                                                                                                                                                                      |
|--------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `configure system` | `write_system_configuration_tool` | The CLI writes defaults only, takes no payload and no overwrite guard, and unbinds the host from every other system. The tool takes a full payload, validates it, and refuses an existing file unless `overwrite` is true       |
| `check-bridge`     | `check_mesoscope_bridge_tool`     | The same probe. The CLI swallows every exception into a WARNING at exit 0, while the tool returns an `error` key, and the CLI echoes rather than returning the `reachable` flag                                                 |
| `preprocess`       | `preprocess_session_tool`         | The CLI resolves both operands before the data-root check and the tool compares them unresolved. The CLI validates the path through Click and prompts when it is omitted, and raises where the tool returns an `Error: ` string |
| `delete`           | `delete_session_tool`             | The tool refuses to act until the caller passes an explicit confirmation value, and the CLI purges immediately. The same resolve and error-reporting differences as `preprocess` apply                                          |
| `migrate`          | `migrate_animal_tool`             | The same three arguments against the same library call. The CLI names them `-s`, `-d`, and `-a`, and the tool names them `source_project`, `destination_project`, and `animal_id`                                               |

### The three rules behind the table

1. **Direction of travel.** Every paired command writes or acts, and every unpaired tool reads. An agent asking what
   the rig currently holds has only the MCP path.
2. **Guard placement.** The CLI's guards are path containment checks, and the tool's guards are explicit confirmation
   and overwrite arguments. Neither surface carries both.
3. **Hardware reach.** No command with an MCP equivalent touches hardware during a session, and every command that
   does has no equivalent by design.

---

## Fallback: what to tell a user when MCP is unavailable

Confirm the server is genuinely unrecoverable through `experiment:experiment-mcp-environment-setup` before handing one
of these over.

| Blocked MCP tool                  | Tell the user to run                                                          |
|-----------------------------------|-------------------------------------------------------------------------------|
| `write_system_configuration_tool` | `sle mesoscope configure system`, then edit the written file by hand          |
| `check_mesoscope_bridge_tool`     | `sle mesoscope check-bridge`                                                  |
| `preprocess_session_tool`         | `sle mesoscope preprocess -sp <session>`                                      |
| `delete_session_tool`             | `sle mesoscope delete -sp <session>`, after stating that it purges every copy |
| `migrate_animal_tool`             | `sle mesoscope migrate -s <source> -d <destination> -a <animal>`              |

Two caveats. The `configure system` substitute writes defaults and discards the current hardware parameters, so tell
the user to copy the existing file aside first. The `delete` substitute carries none of the confirmation the tool
demands, so obtain the user's explicit go-ahead in the conversation before printing it.

Everything else genuinely blocks until the server is back. That covers reading the active system configuration,
validating it, checking the declared mounts, diffing the camera configurations, describing the configuration schema,
and reading or writing the per-session Zaber positions, mesoscope positions, and configuration snapshot. Say so
plainly rather than improvising a substitute out of a shell command.

---

## Related skills

| Skill                                         | Relationship                                                                       |
|-----------------------------------------------|------------------------------------------------------------------------------------|
| `experiment:cli-reference`                    | Owns the `sle` root, the `sle mcp` command, and the `sle get` surface              |
| `experiment:experiment-mcp-environment-setup` | Owns the `sle --help` smoke test, the response contract, and MCP recovery          |
| `/mesoscope-vr-runtime`                       | Owns what each `run` subcommand and `maintain` does once the command starts        |
| `/mesoscope-vr`                               | Owns the system configuration field schema `configure system` writes               |
| `/mesoscope-vr-experiment-schema`             | Owns the experiment configuration field schema `configure experiment` writes       |
| `/mesoscope-vr-session-schema`                | Owns the descriptor and hardware-state schemas the descriptor tools carry          |
| `/mesoscope-vr-snapshots`                     | Owns the Zaber and mesoscope-objective position schemas the position tools carry   |
| `experiment:data-management`                  | Owns the preprocessing, transfer, and purge primitives the three data commands run |
| `assets:task-templates`                       | Owns the task templates `configure experiment` instantiates through `-t`           |
| `assets:working-directory`                    | Owns the working directory `configure system` writes the configuration file into   |
| `assets:project-hierarchy`                    | Owns the on-disk hierarchy the `-sp`, `-p`, and `-a` values address                |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Answering a CLI question, reader-judged:
- [ ] Answered from this skill or from sle mesoscope COMMAND --help, never from memory
- [ ] Quoted the long option form, and never presented -h as a help alias
- [ ] Named the MCP tool the command maps to, or said plainly that none exists
- [ ] Never read -id as a job identifier, since it is --initial-duration on run-training
- [ ] Invoked no sle mesoscope command other than --help

Handing a user a CLI command, reader-judged:
- [ ] Placed every shared identifier before the subcommand name on sle mesoscope run
- [ ] Warned that configure system discards the current hardware parameters before offering it
- [ ] Obtained an explicit go-ahead before printing sle mesoscope delete
- [ ] Confirmed the MCP server is unrecoverable through experiment-mcp-environment-setup first

Preparing a hardware command, reader-judged:
- [ ] Printed the run or maintain command for the experimenter instead of running it
- [ ] Stated that an experimenter must be at the rig with the animal before the command starts
- [ ] Deferred to /mesoscope-vr-runtime for what the runtime then does
```
