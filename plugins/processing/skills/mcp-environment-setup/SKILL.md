---
name: processing-mcp-environment-setup
description: >-
  Placeholder for the Sollertia processing MCP server environment setup. The processing plugin does not
  yet wire an MCP server because the relevant functionality is being absorbed from the deprecated
  sl-behavior package into other Sollertia libraries. Use this skill to understand the current state
  and the migration path.
user-invocable: true
---

# Sollertia processing MCP environment setup (placeholder)

The Sollertia processing plugin does not yet wire an MCP server. This skill exists as a placeholder to
document the current state of post-acquisition processing in the Sollertia ecosystem and the migration
path.

---

## Current state (as of plugin v0.1.0)

- **No `mcpServers` entry** is registered in the processing plugin's `plugin.json`. There is no
  command for the Claude assistant to launch.
- **The deprecated `sl-behavior` package** previously provided a single MCP server (`sl-behavior` /
  `mcp_server.py`) that exposed batch behavior data processing tools (job discovery, dispatch, status
  monitoring). That package is being absorbed into other Sollertia libraries and should not be relied
  on for new work.
- **The eventual home** for post-acquisition processing tools is undecided at the time of writing.
  Likely candidates include `sollertia-experiment` (extending the existing `sl-manage` server with a
  processing subcommand) or a new `sollertia-processing` library.

When the absorption completes, this skill will be rewritten to mirror the structure of the
configuration and experiment plugin MCP environment setup skills (command verification, environment
diagnosis, dual-distribution model, conda / venv / uv resolution, restart guidance).

---

## Until the migration completes

If a user asks about post-acquisition behavior processing today:

1. **Acknowledge that the plugin is a placeholder.** No MCP tools are exposed.
2. **Hand off to the experiment plugin's `/data-management` skill** for the operations that are already
   covered there (preprocessing a session via `sl-manage mcp`, animal migration, session deletion).
3. **Direct the user to the relevant per-library tooling** if they need processing operations that are
   not in `sl-manage mcp` yet — for example, running `sl-behavior` CLI commands directly (with the
   warning that the package is being deprecated).

Do **not** scaffold new processing-plugin code or skills speculatively. The plugin is intentionally
empty to mark the namespace and reserve its place in the marketplace until the absorption decisions are
made.

---

## Related skills

| Skill                                          | Relationship                                                  |
|------------------------------------------------|---------------------------------------------------------------|
| experiment plugin `/data-management`           | Currently owns the only post-acquisition operations available via MCP |
| experiment plugin `/pipeline`                  | Phase 8 (handoff to processing) currently terminates here     |
| `/processing-pipeline`                         | Sibling placeholder describing the eventual phase ordering    |
