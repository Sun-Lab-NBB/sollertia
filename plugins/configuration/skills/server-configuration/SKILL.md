---
name: server-configuration
description: >-
  Authors and modifies the ServerConfiguration YAML file for sollertia-shared-assets via the sl-configure
  MCP server. Covers remote storage transfer settings, BioHPC compute server credentials, and the
  read/write tool surface for the server configuration. Use when setting up remote data transfer for a new
  Sollertia host or rotating server credentials.
user-invocable: true
---

# Sollertia server configuration

Authors and modifies the `ServerConfiguration` YAML file for `sollertia-shared-assets` using the
`sl-configure mcp` MCP server. This skill is the **exclusive** owner of `ServerConfiguration` reads and
writes — no other skill in the marketplace may call `read_server_configuration_tool` or
`write_server_configuration_tool`.

---

## Scope

**Covers:**
- Authoring `ServerConfiguration`
- Reading the active `ServerConfiguration`
- The relationship between server configuration and the remote storage transfer pipeline used by
  `sl-manage` after a session is preprocessed

**Does not cover:**
- `MesoscopeSystemConfiguration` authoring (see `/system-configuration`)
- Working directory or credentials setup (see `/working-directory`)
- Diagnosing MCP server connectivity (see `/mcp-environment-setup`)

---

## What lives in the server configuration

The `ServerConfiguration` captures everything required to transfer preprocessed sessions from the
acquisition PC to a long-term remote storage tier (typically a BioHPC compute server). It is **distinct
from** `MesoscopeSystemConfiguration`, which describes the local acquisition hardware.

The two configurations live side by side in the working directory but are owned by different skills
because they are written at different times and rotate independently:

- `MesoscopeSystemConfiguration` is set once when an acquisition PC is brought up and rarely changes.
- `ServerConfiguration` may be rotated when the BioHPC server is replaced, when SSH keys are rotated, or
  when the long-term storage path changes.

---

## MCP tool surface

| Tool                              | Purpose                                                                |
|-----------------------------------|------------------------------------------------------------------------|
| `read_server_configuration_tool`  | Reads the current server configuration YAML from the working directory |
| `write_server_configuration_tool` | Writes a new server configuration YAML to the working directory        |

The write tool accepts a full nested dictionary that maps onto the `ServerConfiguration` dataclass tree.
It refuses partial updates — to change a single field, read first, mutate the dictionary, then write the
whole thing back.

---

## Authoring workflow

### Step 1: Verify prerequisites

- The `sollertia-shared-assets` MCP server is connected (else hand off to `/mcp-environment-setup`).
- The working directory is set (else hand off to `/working-directory`).

### Step 2: Determine whether to create or modify

Call `read_server_configuration_tool`. If it returns a configuration, you are modifying. If it returns
an empty / not-found response, you are creating from scratch.

### Step 3: Gather values from the user

For a new host, the values you cannot guess are:

- **Server hostname or IP** — the BioHPC compute server address
- **SSH credentials path** — absolute path to the SSH key on the acquisition PC
- **Remote storage root** — absolute path on the remote server where preprocessed sessions are deposited
- **Project mapping rules** — if the host uses non-default per-project storage paths

### Step 4: Write the configuration

```text
write_server_configuration_tool(configuration={ ... full nested dict ... })
```

### Step 5: Verify

Call `read_server_configuration_tool` and confirm the returned configuration matches what you wrote.

---

## Common modification cases

| Change                                  | Field to mutate                                  |
|-----------------------------------------|--------------------------------------------------|
| Server hostname rotated                 | server hostname / IP field                       |
| SSH key rotated                         | SSH credentials path                             |
| Remote storage path moved               | remote storage root                              |
| Project added with custom storage path  | project mapping rules                            |

---

## Verification checklist

```text
- [ ] /working-directory has been run on this host
- [ ] sollertia-shared-assets MCP server is connected
- [ ] read_server_configuration_tool returned the expected configuration before any write
- [ ] write_server_configuration_tool succeeded without schema errors
- [ ] read_server_configuration_tool returned the expected configuration after the write
- [ ] The new configuration's SSH credentials path actually exists on the host filesystem
- [ ] The new configuration's remote storage root is reachable from the host
```

---

## Related skills

| Skill                                 | Relationship                                                            |
|---------------------------------------|-------------------------------------------------------------------------|
| `/working-directory`                  | Required prerequisite — must be run first                               |
| `/mcp-environment-setup`              | Run first if the MCP server is not connected                            |
| `/system-configuration`               | Sibling — both configurations live in the same working directory        |
| experiment plugin `/data-management`  | Triggers remote transfers using the values authored by this skill       |
