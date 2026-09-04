---
name: server-configuration
description: >-
  Authors and modifies the ServerConfiguration YAML that authorizes SSH and SLURM execution on the sollertia-forgery
  remote compute server. Covers the flat five-field payload, the masked-password write refusal, the overwrite guard, and
  the on-disk location. Use when preparing a host for remote batch execution, when rotating the server account,
  password, host, data root, or environment name, or when a remote call reports an unreadable server configuration.
user-invocable: false
---

# Sollertia server configuration

Authors and modifies the `ServerConfiguration` YAML that `sollertia-forgery` loads before every remote operation, using
the `slf mcp` MCP server. This skill is the **exclusive** owner of `read_server_configuration_tool` and
`write_server_configuration_tool`. No other skill in the marketplace may document or call these two tools.

This configuration authorizes compute rather than storage. It names one account on one remote compute server, the data
root every server-side path resolves against, and the conda environment every remote job activates. `connect_to_server`
in `orchestration/remote.py` builds a `Server` from `get_server_configuration()`, so every call that names
`host='remote'` reads this file first and fails with its error text when the file is absent or incomplete.

---

## Scope

**Covers:**
- Authoring the complete five-field `ServerConfiguration` payload through `write_server_configuration_tool`
- Reading the active configuration, with the password masked, through `read_server_configuration_tool`
- Rotating the account, the password, the host, the server data root, or the shared conda environment name
- The on-disk location of `server_configuration.yaml` and the failure text a missing or blank configuration produces

**Does not cover:**
- Preparing, submitting, monitoring, or cancelling remote batches. Owned by `/batch-processing`.
- Remote project discovery and SLURM queue and accounting reads. Owned by `/remote-execution`.
- The working directory every path in this skill resolves against. Owned by `assets:working-directory`.
- MCP connectivity diagnosis and the plugin-wide response envelope. Owned by `/forging-mcp-environment-setup`.
- The human-facing `slf server configure` command surface. Owned by `/cli-reference`.

**Handoff rules:** a request to run work on the server, rather than to authorize it, belongs to `/batch-processing` for
preparation and execution and to `/remote-execution` for discovery and scheduler reads. A request that stalls because
the working directory is unset belongs to `assets:working-directory`, since neither tool here resolves a path without
it.

---

## Agent requirements

You MUST read and write the server configuration through the two MCP tools this skill owns. Do not call
`create_server_configuration_file` from Python, and do not run `slf server configure` on the user's behalf, because the
CLI is the human path and `/cli-reference` owns it.

You MUST obtain the account password from the user for every write. The read tool never reports the stored password, so
no read can supply it and no complete payload can be assembled without asking.

---

## What lives in the server configuration

`ServerConfiguration` in `server/server_configuration.py` is a flat `YamlConfig` dataclass of exactly five `str` fields,
each defaulting to the empty string. There is no nesting, no SSH key path, no port, no per-project mapping, and no
transfer setting. `to_yaml` writes the fields in declaration order, so the stored document reads in the order below.

| Field         | Meaning                                                                                               |
|---------------|-------------------------------------------------------------------------------------------------------|
| `username`    | The username used for server authentication                                                           |
| `password`    | The password used for server authentication                                                           |
| `host`        | The hostname or IP address used to reach the server                                                   |
| `root`        | The absolute path, on the server, to the single root directory that stores all Sollertia data         |
| `environment` | The name of the shared conda environment, on the server, that every remote job activates before `slf` |

`root` and `environment` are the two fields a misconfiguration usually lands in, because neither one fails at
connection time.

**`root`** is the base every remote path resolves against. `Server.root` returns it as a `Path`, and a project resolves
as `root` joined with the project name. That join is where `discover_remote_project_tool` looks, and it composes every
absolute server-side path a remote tool reports. A wrong root authenticates cleanly and then reports that the server
holds no directory for the project. Point it at the data root itself, never at a project, an animal, or a session.

**`environment`** is the one shared conda environment on the server that every remote job activates before invoking the
`slf` CLI. `Job.__init__` writes `eval $(conda shell.bash hook)`, `conda init bash`, and `source activate` for the named
environment as the preamble of every SLURM script. `environment_command` and `_environment_commands` in
`orchestration/hosts.py` wrap every `slf` invocation issued over SSH in an equivalent activation, running
`eval "$(conda shell.bash hook)" && source activate <environment>` inside `bash -lc`. The scheduler and filesystem
commands the transport issues, such as `sbatch`, `sacct`, `squeue`, and `find`, run outside it because they need no
environment. All pipelines share that single environment, so it must hold `sollertia-forgery` and every processing
library the pipelines drive. A name that does not resolve on the server passes submission and fails the job.

`get_server_configuration` accepts a configuration only when all five fields are non-empty, and it checks nothing else.
Reachability, the validity of `root`, and the existence of the named environment are never verified locally, so the
first proof that a configuration is correct is a remote call that succeeds.

The password is stored in plaintext in the YAML, and `Server` passes it straight to paramiko password authentication.
There is no key-file code path anywhere in `server/`, so filesystem permissions on the working directory are the only
protection the platform provides for it.

---

## MCP tool surface

| Tool                              | Purpose                                                                             |
|-----------------------------------|-------------------------------------------------------------------------------------|
| `read_server_configuration_tool`  | Loads the stored configuration and reports its five fields with the password masked |
| `write_server_configuration_tool` | Validates a complete payload by round trip, then persists it as the configuration   |

`read_server_configuration_tool` takes no parameters. Neither tool accepts a `host` argument and neither opens a
connection, so both run entirely against the local working directory.

### Write parameters

| Parameter               | Type             | Default  | Meaning                                                                                |
|-------------------------|------------------|----------|----------------------------------------------------------------------------------------|
| `configuration_payload` | `dict[str, Any]` | required | The complete flat payload, keyed `username`, `password`, `host`, `root`, `environment` |
| `overwrite`             | `bool`           | `False`  | Keyword-only. Whether to replace an existing configuration file                        |

`overwrite` guards the existing file alone. Left at its default with a file already present, the write returns
`Unable to write the server configuration. A file already exists at '<path>'. Pass overwrite=True to replace it.` Every
edit after the first creation therefore passes `overwrite=True`.

### Response payloads

The read tool returns `data` holding `username`, `password`, `host`, `root`, and `environment`, with `password` set to
the literal `<masked>`. The write tool returns `file_path` and the same `data`, built from the round-tripped instance,
so it reports what was actually persisted rather than what was submitted.

The response envelope every tool on this server returns, and the staged-read contract its read tools follow, are
documented in the `## Response contract` section of `/forging-mcp-environment-setup`.

### Failure modes

**The password is never readable.** `read_server_configuration_tool` always substitutes the literal `<masked>`, and
`write_server_configuration_tool` refuses a payload whose `password` equals that literal, reporting that persisting it
would replace the real password and leave every later connection unable to authenticate. A read, mutate, and write
round trip therefore fails every time. Re-prompt the user for the real password on every write, including a write that
changes only `host` or `root`.

**An omitted field is persisted rather than rejected.** Every field defaults to the empty string, so a partial payload
writes empty strings and returns success. A payload of `{}` with `overwrite=True` blanks the whole configuration. The
damage surfaces only at the next read, as the placeholder-values error.

**An unconfigured host reports an error, not an empty payload.** A missing file and a blank file both come back as
`success: false`, distinguished only by the message. An absent file names `slf server configure` and asks the user to
create the file. A file with any empty field reports that it is unconfigured or carries placeholder values for one or
more of `username`, `password`, `host`, `root`, and `environment`. Read the message rather than branching on absence.

**Unknown keys vanish and types go unchecked.** `ServerConfiguration.from_yaml` builds through dacite with type
checking disabled and no strict mode, so an extra key never reaches the file and a non-string value is persisted as
written.

---

## Path conventions

The configuration lives in the configuration subdirectory of the working directory rather than in the working directory
itself.

```text
<working directory>/configuration/server_configuration.yaml
```

`get_server_configuration_path` composes that path from `get_working_directory()`, the `CONFIGURATION_DIRECTORY`
constant whose value is `configuration`, and `_SERVER_CONFIGURATION_FILENAME` whose value is
`server_configuration.yaml`. The looser phrasing is widespread. Both tool docstrings place the file in the working
directory, and the not-found message from `get_server_configuration` reads "in the Sollertia platform working
directory" while interpolating the full file path. Report the subdirectory path when a user
asks where the file lives.

The write tool creates the configuration subdirectory when it is absent, because the scratch file it validates through
is written with `direct_write`. An unset working directory is the one path failure it cannot repair, and it reports
`Unable to resolve the server configuration path.` followed by the underlying message. Hand that case off to
`assets:working-directory`.

---

## Authoring workflow

### Step 1: Verify prerequisites first

- The `slf mcp` server is connected. If it is not, hand off to `/forging-mcp-environment-setup`.
- The working directory is set. If it is not, hand off to `assets:working-directory`, because both tools resolve their
  path against it and neither can proceed without one.

### Step 2: Read the current configuration

```text
read_server_configuration_tool()
```

A success response means a usable configuration already exists and this edit is a replacement. An error response means
the file is absent or blank, and its message says which. Keep the four visible fields of a success response as the
starting point for the payload, and treat the returned `password` as unusable.

### Step 3: Collect the five values from the user

Ask for every value the read did not supply, and never guess one.

- `username` and `password` for the server account. Collect the password on every write, since no read returns it.
- `host` as the hostname or IP address that reaches the server.
- `root` as the absolute path on the server to the single Sollertia data root that holds raw and processed data.
- `environment` as the name of the conda environment on the server that holds this library and every processing
  library its pipelines drive.

### Step 4: Write the complete payload

```text
write_server_configuration_tool(
    configuration_payload={
        "username": "<the server account name>",
        "password": "<the account's real password, never the masked placeholder>",
        "host": "<hostname or IP address>",
        "root": "<absolute path on the server to the Sollertia data root>",
        "environment": "<conda environment name on the server>",
    },
    overwrite=True,
)
```

Send all five keys in one flat mapping. Omitting a key writes an empty string in its place and leaves a configuration
that every later remote call rejects.

### Step 5: Verify the write

```text
read_server_configuration_tool()
```

Compare `username`, `host`, `root`, and `environment` against the payload you sent. The password cannot be confirmed
this way, because the read masks it, so the first real confirmation is a remote call that authenticates. Route that
confirmation to `/remote-execution`, whose discovery tool connects and resolves a project under `root` in one step.

---

## Common modification cases

Every row is a complete five-field write with `overwrite=True` and a freshly collected password, because the tool
persists the whole document and refuses the masked placeholder.

| Change                                          | Field carrying the new value         |
|-------------------------------------------------|--------------------------------------|
| Account password rotated                        | `password`                           |
| Server replaced, renamed, or re-addressed       | `host`                               |
| Data root moved on the server                   | `root`                               |
| Processing environment rebuilt under a new name | `environment`                        |
| Work moved to a different server account        | `username` and `password`            |
| Configuration blanked by a partial write        | all five, re-collected from the user |

---

## Related skills

| Skill                            | Relationship                                                                               |
|----------------------------------|--------------------------------------------------------------------------------------------|
| `/forging-mcp-environment-setup` | Run first if the `slf mcp` server is not connected. Owns the plugin-wide response contract |
| `assets:working-directory`       | Prerequisite that owns the working directory this configuration resolves against           |
| `/remote-execution`              | Consumer that discovers projects under `root` and reads the scheduler for submitted jobs   |
| `/batch-processing`              | Consumer that prepares and executes every `host='remote'` batch through this configuration |
| `/cli-reference`                 | Owns the human-facing `slf server configure` path and its options                          |
| `/pipeline`                      | Context: where remote authorization sits in the end-to-end processing flow                 |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Authoring the configuration:
- [ ] sollertia-forgery MCP server is connected
- [ ] The working directory was set before either tool was called
- [ ] read_server_configuration_tool was called first, and its response was read as success or error rather than as
      present or empty
- [ ] The user supplied the account's real password for this write, rather than the <masked> literal a read reported
- [ ] The payload was one flat mapping carrying username, password, host, root, and environment
- [ ] overwrite=True was passed whenever a configuration file already existed
- [ ] write_server_configuration_tool returned success, and its data reported the intended values
- [ ] read_server_configuration_tool was called after the write and its four visible fields matched the payload

Field correctness:
- [ ] root is the absolute path on the server to the single Sollertia data root, not a project, animal, or session path
- [ ] environment names a server conda environment holding sollertia-forgery and every library its pipelines drive
- [ ] The user was told the file sits under the configuration subdirectory of the working directory
- [ ] Remote authorization was confirmed by a connecting call routed to /remote-execution, not by the local read alone
- [ ] No acquisition-system-specific file name, column, or session type appears in this skill
```
