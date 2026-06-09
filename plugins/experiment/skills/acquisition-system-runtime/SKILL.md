---
name: acquisition-system-runtime
description: >-
  Documents the platform-general runtime pattern for a Sollertia data acquisition system: the
  per-mode logic functions, the orchestrator's state machine and per-cycle loop, typed-event
  dispatch, and the visualizer/control-UI surface. Use when designing the runtime layer of a
  new acquisition system, adding a runtime mode/state, or auditing an existing runtime for
  pattern compliance.
user-invocable: false
---

# Acquisition system runtime

Documents the platform-general **runtime behavior** pattern for a Sollertia data acquisition system —
how the host-PC stack drives the binding classes through a session once they are composed. It is the
dynamic-behavior counterpart to `experiment:acquisition-system-design`, which covers the static
composition (configuration YAML → configuration dataclasses → binding classes → orchestrator construction).

This is a **pattern skill** — it documents the conventions and contracts every Sollertia acquisition
runtime shares, not any single system's specific states or modes. For the concrete worked instance,
see `experiment:mesoscope-vr-runtime`.

---

## Scope

**Covers:**
- The configuration-time / runtime split (AI assists configuration; runtime is deterministic and AI-free)
- Per-mode runtime logic functions — the public entry points that run a session
- The orchestrator's dynamic responsibilities: the system-state machine, the per-cycle runtime loop,
  and lifecycle (start/stop/pause/resume)
- The two state axes: system state (hardware configuration) vs runtime state (within-session stage)
- Typed-event dispatch from the hardware subsystems into the runtime loop
- Session-descriptor consumption (plan parameters plus runtime summary) and the DataLogger archive
- The real-time visualizer and interactive control-UI pattern
- Log message codes and their downstream consumption
- The CLI surface pattern for launching sessions
- Workflows for adding a runtime mode and for building a runtime for a new system

**Does not cover** (delegated):
- Static composition (configuration YAML, configuration dataclasses, binding-class construction/shutdown
  order) — see `experiment:acquisition-system-design`
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

- **Configuration time (AI-assisted):** the AI writes the validated system- and experiment-level
  configuration files through the MCP tool surface during the earlier setup phases.
- **Runtime (deterministic):** the CLI forwards its flags to the per-mode logic function, which reads
  those configuration files, builds the session's `SessionData` and descriptor, and drives the
  hardware with no further AI involvement.

---

## Runtime architecture

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│  CLI command group (one subcommand per runtime mode)                         │
│  collects shared session args + per-mode flag overrides and forwards         │
│  them to the per-mode runtime logic function                                 │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  Per-mode runtime logic functions                                            │
│  build SessionData + descriptor (flag overrides) → construct orchestrator    │
│  → drive state transitions and the per-cycle runtime loop → tear down        │
└────────────────────────────────────────────────────┬─────────────────────────┘
                                                     │ composes (Layer 3 of acquisition-system-design)
┌────────────────────────────────────────────────────▼─────────────────────────┐
│  Runtime orchestrator (one per acquisition system)                           │
│  - owns the binding classes, DataLogger, and any asset-subsystem drivers     │
│  - owns the system-state machine + the within-session runtime state          │
│  - per cycle: syncs hardware state, dispatches events, services UI           │
└──────────────────────────────────────────────────────────────────────────────┘
```

The orchestrator's *construction* and *shutdown ordering* are governed by
`experiment:acquisition-system-design` (Layer 3). This skill governs what it *does* between start and
stop.

---

## Per-mode runtime logic functions

Each runtime mode a system supports has one top-level function that is the public entry point for a
session of that mode. Every such function follows the same shape:

1. Validate inputs (read the AI-written configuration files, verify project/animal membership) and
   build the session's `SessionData` data hierarchy.
2. Build the session descriptor: instantiate it with defaults, inherit parameters from the animal's
   previous session of this mode (if one exists on disk), and apply the per-flag overrides forwarded
   by the CLI.
3. Construct the runtime orchestrator (Layer 3 of `experiment:acquisition-system-design`), passing the
   in-memory descriptor. For recording modes the orchestrator owns and starts the `DataLogger`;
   maintenance modes start a standalone `DataLogger` first.
4. Drive the state transitions and the per-cycle runtime loop for the session's lifetime.
5. Tear down in the reverse order on completion or interrupt.

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
  logs the transition. These state methods are **convergent and unconditional** — they never
  short-circuit on the current system state, so re-entering a state re-runs its full configuration
  sequence and logs a fresh transition. The hardware itself is not necessarily re-commanded, though:
  each per-actuator driver wrapper short-circuits on its own cached state, so re-entry only drives the
  actuators not already in the target setting.
- **Runtime state (stage)** — an integer "stage" code that partitions a session into an ordered
  sequence of phases (e.g., trial blocks). Unlike system state, it touches **no hardware**: a single
  method updates the cached stage and logs it, with no binding-class calls. The stage vocabulary and
  the points at which it advances are **session-plan data defined by the experimenter**, not hardware
  policy. In an experiment mode the ordered stage sequence comes from the experiment configuration and
  is stepped through by the per-mode logic function, while simpler modes may hold a single fixed stage
  code for the whole session. Its only role is to stamp within-session boundaries onto the logged
  timeline so the downstream behavior pipeline can segment the recording: system state is what changes
  the rig, runtime state merely marks the transitions for analysis. The pause/resume machinery
  preserves the pre-pause stage across an idle interval and re-logs it on resume, keeping the timeline
  correct.

Both axes are logged to the DataLogger as distinct event codes so downstream processing can
reconstruct the full state timeline.

### The per-cycle runtime loop

The session's heartbeat is a single cycle method the logic function calls repeatedly. Each cycle fans
out to one bounded step per concern, so no single concern can starve the others:

- **Data sync** — read each microcontroller subsystem's current state once per cycle through its
  shared-memory-backed accessors (position, lick count, dispensed volume). Update the orchestrator's
  trackers from the change since the previous cycle, and forward derived quantities to downstream
  subsystems and the visualizer. Microcontroller messages are received and parsed **asynchronously** off
  the main loop and published to shared memory — this step samples the latest published value, not a per-cycle
  message backlog.
- **Event dispatch** — consume **at most one** typed event from each asynchronous asset-subsystem driver and
  dispatch it (see below).
- **UI service** — service the interactive control UI (pause/resume, parameter modifiers, manual commands).
- **Auxiliary tasks** — any system-specific per-cycle bookkeeping (e.g., external-acquisition sync).

Keeping each step bounded per cycle is the core latency contract: the loop must return promptly so the
keepalive to every microcontroller subsystem stays within its interval (see
`experiment:acquisition-system-design`'s keepalive enforcement).

### Typed-event dispatch from hardware subsystems

Asynchronous hardware subsystems (e.g., the VR task driver) surface their per-cycle messages as **typed
events** rather than raw payloads. The subsystem's driver parses the transport message and returns a small
typed value (an event kind plus any payload fields); the orchestrator switches on the kind and acts on
its own hardware (deliver a reward, pulse a brake, enter an emergency pause). This keeps transport
parsing inside the subsystem and hardware policy inside the orchestrator. The Unity VR task driver is a standard
subsystem of every acquisition system and its `VRTaskEvent` is the async event source every runtime dispatches —
see `experiment:vr-driver-interface`.

### Lifecycle

`start()` brings the session up (semi-interactive hardware preparation), the cycle loop runs the
session, and `stop()` tears it down. Pause/resume implement an idle state that produces no valid data;
terminate handles end-of-session shutdown. Construction/teardown ordering is owned by
`experiment:acquisition-system-design`.

---

## Session-descriptor consumption

The session descriptor holds the runtime parameters for one session, authored as a dataclass owned by
`sollertia-shared-assets` (see assets plugin `/session-descriptors`):

- The per-mode logic function instantiates the descriptor with defaults, inherits parameters from the
  animal's previous session of the same mode (if one exists on disk), and applies the per-flag
  overrides forwarded by the CLI.
- The orchestrator receives the finalized descriptor in-memory, caches it to disk at construction (so
  an interrupted session can still be preprocessed), and uses it to parameterize the state machine.
- At session end the orchestrator updates the descriptor in place with runtime-discovered values (the
  dispensed water volume, the final run-speed/duration thresholds, the completion flag), prompts the
  operator for notes, and re-saves it.

The descriptor therefore carries both the session plan and a summary of its outcome, while the
DataLogger archive carries the full sample-by-sample record. The descriptor makes a session
reproducible and human-readable at a glance; the archive makes it auditable in full.

---

## Visualizer and control UI

Two operator-facing surfaces are standard:

- **A real-time visualizer** that renders behavior data, reading from the binding classes'
  `SharedMemoryArray`-backed property accessors (safe to read from the main process; see
  `experiment:microcontroller-interface`). Its display mode is selected per runtime mode.
- **An interactive control UI** for operator actions during a session (pause/resume, threshold
  modifiers, manual commands).

Both are constructed by the orchestrator. The control UI is serviced by the runtime loop's UI-service
step, while the visualizer is fed by the data-sync step and repainted once per cycle. Adding a runtime mode with
new display needs is a change to the visualizer's mode enumeration and its plotting logic.

---

## Log message codes

The orchestrator emits a system-specific `IntEnum` of log message codes to the DataLogger — at minimum
one for each state axis (system state, runtime state) plus codes for guidance/parameter changes and
event-driven snapshots. New events take the next unused code. These codes are the contract consumed by the
downstream behavior pipeline (forging plugin), so a code's meaning is durable — never recycle a freed
value.

---

## CLI surface pattern

Runtime modes are exposed as subcommands of the system's CLI group. Shared session arguments (user,
project, animal, etc.) are supplied on the parent command and inherited by each mode subcommand. Each
subcommand is a thin wrapper: it collects its per-mode flag overrides and forwards them, with the
inherited session arguments, to the per-mode logic function (which builds the `SessionData` and
descriptor). The CLI is the only public surface for starting a session.

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

---

## Workflow: building a runtime for a new acquisition system

1. Compose the system statically first (see `experiment:acquisition-system-design`).
2. Define the system-state enumeration (one member per hardware mode) and the runtime-state stage codes.
3. Define the log message code enumeration (one per state axis, plus domain events).
4. Implement the orchestrator's per-cycle loop with one bounded step per hardware subsystem present.
5. Implement the per-mode logic functions for the system's session types.
6. Add the CLI command group and subcommands.
7. Author the per-system runtime skill documenting the concrete states, modes, and CLI.

A new system will diverge from the current instance wherever its hardware subsystems differ — fewer or more
steps, a different state set, etc. The patterns above are conventions, not a fixed
template; apply the ones that fit the system's hardware.

---

## Mesoscope-VR as a worked example

`experiment:mesoscope-vr-runtime` is the current concrete instance of every pattern here:
`MesoscopeVRStates` (the system-state enum), `MesoscopeVRSystem` (the orchestrator with its
`runtime_cycle()` fanning out to `_data_cycle` / `_unity_cycle` / `_ui_cycle` / `_mesoscope_cycle`),
the per-mode logic functions in `data_acquisition.py`, the `BehaviorVisualizer` + `RuntimeControlUI`,
the `MesoscopeVRLogMessageCodes`, and the `sle mesoscope run` CLI. Read that skill for the concrete
states, modes, and commands; read this skill for the pattern they instantiate.

---

## Maintenance contract

Update this skill when a **platform-general** runtime convention changes (the cycle-loop contract, the
two-state-axis model, the descriptor parameter/summary lifecycle, the typed-event dispatch pattern, the CLI
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
| `experiment:vr-driver-interface`       | The typed-event asset-subsystem source (`VRTaskEvent`) the loop dispatches                   |
| assets plugin `/session-descriptors`   | Authors the descriptors and `SessionTypes` the runtime consumes                              |
| `experiment:data-management`           | Post-acquisition session-data lifecycle                                                      |
| `experiment:pipeline`                  | Where the runtime phase sits in the end-to-end lifecycle                                     |

---

## Verification checklist

```text
When designing or auditing an acquisition-system runtime:
- [ ] Runtime is launchable only via the CLI (no MCP "start session" tool)
- [ ] Per-mode logic functions follow the standard shape (SessionData → descriptor → orchestrator → loop → teardown)
- [ ] System state and runtime state are distinct axes, each logged with its own code
- [ ] System-state methods drive the binding classes into the target configuration and log the transition
- [ ] The per-cycle loop has one bounded step per hardware subsystem; keepalive interval is never exceeded
- [ ] Asynchronous subsystems surface typed events; the orchestrator dispatches on event kind
- [ ] Descriptor built by the logic function, updated with runtime results at session end; archive holds the full record
- [ ] Log message codes are append-only (no recycled values)
- [ ] The per-system runtime skill documents the concrete states, modes, and CLI
```
