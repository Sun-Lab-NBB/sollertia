---
name: processing-pipeline
description: >-
  Placeholder for the Sollertia post-acquisition processing pipeline orchestration guide. The processing
  plugin does not yet have content because the relevant functionality is being absorbed from the
  deprecated sl-behavior package into other Sollertia libraries. Use this skill to understand the
  current state and the eventual phase ordering.
user-invocable: true
---

# Sollertia processing pipeline (placeholder)

The Sollertia processing plugin is currently a placeholder while post-acquisition processing
functionality is being absorbed from the deprecated `sl-behavior` package into other Sollertia
libraries. This skill marks the eventual structure of the processing pipeline and explains where the
work currently lives.

---

## Current state (as of plugin v0.1.0)

The processing plugin contains:

- **No MCP server registration.** See `/processing-mcp-environment-setup` for the migration context.
- **No phase-specific skills.** The eventual skill set will mirror the ataraxis communication and
  video plugins' processing pipelines: extraction configuration, log archive discovery, batch job
  dispatch, results format reference.
- **This placeholder pipeline skill.** Reserves the orchestration role and documents what the pipeline
  will look like once content lands.

---

## Eventual pipeline shape

When the absorption completes, the processing pipeline is expected to look approximately like this:

```text
Handoff       Archive        Extraction     Batch          Results        Analysis
from        → Discovery   →  Configuration → Processing → Verification → Handoff
Experiment      |              |              |              |              |
   |        /processing-    /processing-   /processing-   /processing-   downstream
/data-      archive-        extraction-    log-           log-           tooling
management  discovery       configuration  processing     processing-
                                                          results
```

Each phase will be backed by the same idioms used in the ataraxis processing pipelines:

| Phase                       | Expected source tooling                                     |
|-----------------------------|-------------------------------------------------------------|
| Handoff from experiment     | experiment plugin `/data-management` (already exists)       |
| Archive discovery           | likely `ataraxis@communication:log-input-format` derivatives |
| Extraction configuration    | likely `ataraxis@communication:extraction-configuration` reuse |
| Batch processing            | likely `ataraxis@communication:log-processing` reuse        |
| Results verification        | likely `ataraxis@communication:log-processing-results` reuse|
| Analysis handoff            | per-project notebooks / scripts (out of scope for this plugin) |

---

## What to do today

1. **Use the experiment plugin's `/data-management`** for everything that is already covered there
   (preprocessing, migration, deletion).
2. **Use the ataraxis communication and video plugins directly** for log archive processing — they are
   the canonical implementations and will remain so. The Sollertia processing plugin will eventually
   add a thin Sollertia-data-class-aware layer on top, not replace them.
3. **Do not invoke this skill for actual processing work** until the absorption completes. It exists
   only to communicate the current state and the future structure.

---

## Related skills

| Skill                                          | Relationship                                                  |
|------------------------------------------------|---------------------------------------------------------------|
| `/processing-mcp-environment-setup`            | Sibling placeholder describing the eventual MCP wiring        |
| experiment plugin `/data-management`           | Currently owns the only post-acquisition operations           |
| experiment plugin `/pipeline`                  | Phase 8 of the experiment lifecycle currently terminates here |
| `ataraxis@communication:log-processing`        | Reference implementation for batch log processing             |
| `ataraxis@video:log-processing`                | Reference implementation for camera log processing            |
