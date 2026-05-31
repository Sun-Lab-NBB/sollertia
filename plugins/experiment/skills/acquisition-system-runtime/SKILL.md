---
name: acquisition-system-runtime
description: >-
  Documents the platform-general runtime pattern for a Sollertia data acquisition system: the
  configuration-time/runtime split, per-mode runtime logic entry points, the orchestrator's state
  machine and per-cycle event loop, typed event dispatch from the hardware lanes, descriptor
  consumption, and the visualizer/control-UI surface. Use when designing the runtime layer of a new
  acquisition system, adding a runtime mode/state, or auditing an existing runtime for pattern compliance.
user-invocable: false
---

# Acquisition system runtime

Documents the platform-general **runtime behavior** pattern for a Sollertia data acquisition system —
how the host-PC stack drives the binding classes through a session once they are composed. It is the
dynamic-behavior counterpart to `experiment:acquisition-system-design`, which covers the static
composition (configuration → calibration dataclasses → binding classes → orchestrator construction).

This is a **pattern skill** — it documents the conventions and contracts every Sollertia acquisition
runtime shares, not any single system's specific states or modes. For the concrete worked instance,
see `experiment:mesoscope-vr-runtime`.

---

## Scope

**Covers:**
- The configuration-time / runtime split (AI assists configuration; runtime is deterministic and AI-free)
- Per-mode runtime logic functions — the public entry points that run a session
- The orchestrator's dynamic responsibilities: the system-state machine, the per-cycle runtime loop, and lifecycle (start/stop/pause/resume)
- The two state axes: system state (hardware configuration) vs runtime state (within-session stage)
- Typed-event dispatch from the hardware lanes into the runtime loop
- Session-descriptor consumption (the "session plan") vs the DataLogger archive (the "session result")
- The real-time visualizer and interactive control-UI pattern
- Log message codes and their downstream consumption
- The CLI surface pattern for launching sessions
- Workflows for adding a runtime mode and for building a runtime for a new system

**Does not cover** (delegated):
- Static composition (configuration YAML, calibration dataclasses, binding-class construction/shutdown order) — see `experiment:acquisition-system-design`
- Concrete Mesoscope-VR runtime behavior (its states, modes, CLI, visualizer) — see `experiment:mesoscope-vr-runtime`
- Per-firmware-module wrapper APIs the orchestrator consumes — see `experiment:microcontroller-interface`
- The Unity VR task driver event source — see `experiment:vr-driver-interface`
- Session descriptors / `SessionTypes` authoring — owned by `assets:session-descriptors`
- Session-data lifecycle after acquisition (preprocess, transfer, delete) — see `experiment:data-management`

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: **AI assistance operates at configuration
time; runtime acquisition is fully deterministic and AI-independent.** The MCP tool surface
intentionally has no "start a session" tool, and there will not be one. A runtime is launched only
through the system's CLI, which reads validated configuration and descriptor files written during the
AI-assisted phases.

A runtime layer therefore has two consumers:

- **Configuration time (AI-assisted):** the CLI command builds a `SessionData` and a session
  descriptor from flags and writes them to disk.
- **Runtime (deterministic):** the per-mode logic function reads those files and drives the hardware
  with no further AI involvement.

---

## Runtime architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CLI command group (one subcommand per runtime mode)                         │
│  builds SessionData + descriptor (with flag overrides), writes them, then     │
│  calls the per-mode runtime logic function                                    │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  Per-mode runtime logic functions                                            │
│  start DataLogger → construct the orchestrator → load the descriptor →        │
│  drive state transitions and the runtime cycle loop → tear down in reverse    │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │ composes (Layer 3 of acquisition-system-design)
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  Runtime orchestrator (one per acquisition system)                           │
│  - owns the binding classes, DataLogger, and any asset-lane drivers          │
│  - owns the system-state machine + the within-session runtime state          │
│  - per cycle: pumps each hardware lane, dispatches typed events, updates UI   │
└──────────────────────────────────────────────────────────────────────────────┘
```

The orchestrator's *construction* and *shutdown ordering* are governed by
`experiment:acquisition-system-design` (Layer 3). This skill governs what it *does* between start and
stop.

---

## Per-mode runtime logic functions

Each runtime mode a system supports has one top-level function that is the public entry point for a
session of that mode. Every such function follows the same shape:

1. Validate inputs and prepare the session's output directories.
2. Start the `DataLogger`.
3. Construct the runtime orchestrator (Layer 3 of `experiment:acquisition-system-design`).
4. Load the session-specific descriptor (the runtime parameters written to disk by the CLI).
5. Drive the state transitions and the per-cycle runtime loop for the session's lifetime.
6. Tear down in the reverse order on completion or interrupt.

These functions are the only supported surface for starting a session — direct construction of the
orchestrator from notebooks or scripts is not supported. Modes that perform maintenance rather than
recording (calibration, positioning) follow the same shape without a session descriptor.

---

## The orchestrator's dynamic responsibilities

### Two state axes

A runtime distinguishes two orthogonal state axes:

- **System state** — the hardware configuration the system is currently in (which actuators are
  engaged, which sensors are active). It is a system-specific enumeration with one member per
  hardware mode. A dedicated method per state drives the binding classes into that configuration and
  logs the transition. System-state transitions are **idempotent** — re-entering the current state is
  a no-op.
- **Runtime state (stage)** — an integer "stage" code that advances *within* a session (e.g., across
  trial phases). A single method updates and logs it. The pause/resume machinery restores the
  pre-pause runtime stage when a session resumes from a paused (idle) state.

Both axes are logged to the DataLogger as distinct event codes so downstream processing can
reconstruct the full state timeline.

### The per-cycle runtime loop

The session's heartbeat is a single cycle method the logic function calls repeatedly. Each cycle fans
out to one bounded step per concern, so no single concern can starve the others:

- **Data pump** — drain each microcontroller lane's incoming data, update trackers, and forward
  derived quantities (position, licks) to any downstream lanes and the visualizer.
- **Event pump** — consume **at most one** typed event from each asynchronous asset-lane driver and
  dispatch it (see below).
- **UI pump** — service the interactive control UI (pause/resume, parameter modifiers, manual commands).
- **Auxiliary pumps** — any system-specific per-cycle bookkeeping (e.g., external-acquisition sync).

Keeping each pump bounded per cycle is the core latency contract: the loop must return promptly so the
keepalive to every microcontroller lane stays within its interval (see
`experiment:acquisition-system-design`'s keepalive enforcement).

### Typed-event dispatch from hardware lanes

Asynchronous hardware lanes (e.g., the VR task driver) surface their per-cycle messages as **typed
events** rather than raw payloads. The lane's driver parses the transport message and returns a small
typed value (an event kind plus any payload fields); the orchestrator switches on the kind and acts on
its own hardware (deliver a reward, pulse a brake, enter an emergency pause). This keeps transport
parsing inside the lane and hardware policy inside the orchestrator. The canonical example is the VR
task driver's `VRTaskEvent` — see `experiment:vr-driver-interface`.

### Lifecycle

`start()` brings the session up (semi-interactive hardware preparation), the cycle loop runs the
session, and `stop()` tears it down. Pause/resume implement an idle state that produces no valid data;
terminate handles end-of-session shutdown. Construction/teardown ordering is owned by
`experiment:acquisition-system-design`.

---

## Session-descriptor consumption

The session descriptor is the **session plan** — the runtime parameters for one session, authored as a
dataclass owned by `sollertia-shared-assets` (see assets plugin `/session-descriptors`). The runtime
treats it as **read-only**:

- The CLI instantiates the descriptor with defaults, applies per-flag overrides, and writes it to the
  session directory before the runtime begins.
- The logic function reads it from disk at session start and uses it to parameterize the state machine.
- Runtime-discovered values (total reward delivered, trials completed) are recorded in the **DataLogger
  archive** (the "session result"), never written back into the descriptor.

The descriptor is the plan; the DataLogger archive is the result. Keeping them separate makes a session
reproducible from its descriptor and auditable from its archive.

---

## Visualizer and control UI

Two operator-facing surfaces are standard:

- **A real-time visualizer** that renders behavior data, reading from the binding classes'
  `SharedMemoryArray`-backed property accessors (safe to read from the main process; see
  `experiment:microcontroller-interface`). Its display mode is selected per runtime mode.
- **An interactive control UI** for operator actions during a session (pause/resume, threshold
  modifiers, manual commands).

Both are constructed by the orchestrator and driven from the runtime loop's UI pump. Adding a runtime
mode with new display needs is a change to the visualizer's mode enumeration and its plotting logic.

---

## Log message codes

The orchestrator emits a system-specific `IntEnum` of log message codes to the DataLogger — at minimum
one for each state axis (system state, runtime state) plus codes for guidance/parameter changes and
periodic snapshots. New events take the next unused code. These codes are the contract consumed by the
downstream behavior pipeline (forging plugin), so a code's meaning is durable — never recycle a freed
value.

---

## CLI surface pattern

Runtime modes are exposed as subcommands of the system's CLI group. Shared session arguments (user,
project, animal, etc.) are supplied on the parent command and inherited by each mode subcommand. Each
subcommand builds a `SessionData`, builds the descriptor with overrides, writes the descriptor, and
calls the per-mode logic function. The CLI is the only public surface for starting a session.

---

## Workflow: adding a runtime mode

1. **Author the descriptor** (assets plugin `/session-descriptors`): add the `SessionTypes` member and
   the descriptor dataclass; bump `sollertia-shared-assets`.
2. **Extend the state machine** if the mode needs a new hardware configuration: add a state enum member
   and a state-driving method that logs the transition.
3. **Add a visualizer mode** if the display needs differ from existing modes.
4. **Author the per-mode logic function** following the standard shape above.
5. **Add the CLI subcommand** that builds the descriptor and calls the logic function.
6. **Export and version-bump** the new function/descriptor; pin the new shared-assets minimum.
7. **Update the per-system runtime skill** (for Mesoscope-VR, `experiment:mesoscope-vr-runtime`).

## Workflow: building a runtime for a new acquisition system

1. Compose the system statically first (see `experiment:acquisition-system-design`).
2. Define the system-state enumeration (one member per hardware mode) and the runtime-state stage codes.
3. Define the log message code enumeration (one per state axis, plus domain events).
4. Implement the orchestrator's per-cycle loop with one bounded pump per hardware lane present.
5. Implement the per-mode logic functions for the system's session types.
6. Add the CLI command group and subcommands.
7. Author the per-system runtime skill documenting the concrete states, modes, and CLI.

A new system will diverge from the current instance wherever its hardware lanes differ — fewer or more
pumps, a different state set, no VR coupling, etc. The patterns above are conventions, not a fixed
template; apply the ones that fit the system's hardware.

---

## Mesoscope-VR as a worked example

`experiment:mesoscope-vr-runtime` is the current concrete instance of every pattern here:
`MesoscopeVRStates` (the system-state enum), `_MesoscopeVRSystem` (the orchestrator with its
`runtime_cycle()` fanning out to `_data_cycle` / `_unity_cycle` / `_ui_cycle` / `_mesoscope_cycle`),
the per-mode logic functions in `data_acquisition.py`, the `BehaviorVisualizer` + `RuntimeControlUI`,
the `_MesoscopeVRLogMessageCodes`, and the `sle mesoscope run` CLI. Read that skill for the concrete
states, modes, and commands; read this skill for the pattern they instantiate.

---

## Maintenance contract

Update this skill when a **platform-general** runtime convention changes (the cycle-loop contract, the
two-state-axis model, the descriptor plan/result split, the typed-event dispatch pattern, the CLI
pattern). Do NOT update it for changes to any single system's concrete states, modes, visualizer
modes, or CLI commands — those belong in the per-system runtime skill
(`experiment:mesoscope-vr-runtime`). When a new acquisition system's runtime reveals a genuinely shared
pattern not captured here, add it.

---

## Related skills

| Skill                                  | Relationship                                                                                 |
|----------------------------------------|----------------------------------------------------------------------------------------------|
| `experiment:acquisition-system-design` | Static composition counterpart (configuration, binding classes, construction/shutdown order) |
| `experiment:mesoscope-vr-runtime`      | The current worked instance of this pattern                                                  |
| `experiment:mesoscope-vr`              | The current worked instance of the static design pattern                                     |
| `experiment:microcontroller-interface` | Per-module wrapper APIs and the SharedMemoryArray accessors the loop reads                   |
| `experiment:vr-driver-interface`       | The typed-event asset-lane source (`VRTaskEvent`) the loop dispatches                        |
| assets plugin `/session-descriptors`   | Authors the descriptors and `SessionTypes` the runtime consumes                              |
| `experiment:data-management`           | Post-acquisition session-data lifecycle                                                      |
| `experiment:pipeline`                  | Where the runtime phase sits in the end-to-end lifecycle                                     |

---

## Verification checklist

```text
When designing or auditing an acquisition-system runtime:
- [ ] Runtime is launchable only via the CLI (no MCP "start session" tool)
- [ ] Per-mode logic functions follow the standard shape (DataLogger → orchestrator → descriptor → loop → teardown)
- [ ] System state and runtime state are distinct axes, each logged with its own code
- [ ] System-state transitions are idempotent
- [ ] The per-cycle loop has one bounded pump per hardware lane; keepalive interval is never exceeded
- [ ] Asynchronous lanes surface typed events; the orchestrator dispatches on event kind
- [ ] The descriptor is read-only at runtime; results go to the DataLogger archive
- [ ] Log message codes are append-only (no recycled values)
- [ ] The per-system runtime skill documents the concrete states, modes, and CLI
```
