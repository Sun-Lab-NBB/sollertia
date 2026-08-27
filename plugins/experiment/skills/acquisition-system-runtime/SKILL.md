---
name: acquisition-system-runtime
description: >-
  Documents the platform-general runtime pattern for a Sollertia data acquisition system: the per-mode logic functions,
  the orchestrator's state machine and per-cycle loop, typed-event dispatch, and the visualizer/control-UI surface. Use
  when designing the runtime layer of a new acquisition system, adding a runtime mode/state, or auditing an existing
  runtime for pattern compliance.
user-invocable: false
---

# Acquisition system runtime

Documents the platform-general **runtime behavior** pattern for a Sollertia data acquisition system, meaning how the
host-PC stack drives the binding classes through a session once they are composed. It is the dynamic-behavior
counterpart to `/acquisition-system-design`, which covers the static composition, from configuration YAML through
configuration dataclasses and binding classes to orchestrator construction.

This is a **pattern skill**. It documents the conventions and contracts every Sollertia acquisition runtime shares,
not any single system's specific states or modes. For the concrete worked instance, see
`mesoscope:mesoscope-vr-runtime`.

---

## Scope

**Covers:**
- The configuration-time / runtime split, where AI assists configuration and the runtime stays deterministic
- Per-mode runtime logic functions, the public entry points that run a session
- The orchestrator's dynamic responsibilities: the system-state machine, the per-cycle runtime loop, lifecycle
  (start/stop/pause/resume), and teardown isolation
- The two state axes: system state (hardware configuration) against runtime state (within-session stage)
- Typed-event dispatch from the hardware subsystems into the runtime loop
- The uninitialized-session marker and the point at which a runtime clears it
- The pre-start checkpoint, the pause clock, and the terminal-prompt operator surface
- Session-descriptor consumption (plan parameters plus runtime summary) and the DataLogger archive
- The real-time visualizer and interactive control-UI pattern, and the separate maintenance UI
- Log message codes and their downstream consumption
- The CLI surface pattern for launching sessions
- Workflows for adding a runtime mode and for building a runtime for a new system

**Does not cover:**
- Static composition, meaning configuration YAML, configuration dataclasses, and binding-class construction and
  shutdown order. See `/acquisition-system-design`
- Concrete Mesoscope-VR runtime behavior, meaning its states, modes, CLI, and visualizer. See
  `mesoscope:mesoscope-vr-runtime`
- The seam catalog a new system's runtime composes. See `/library-extension`
- Per-firmware-module wrapper APIs the orchestrator consumes. See `/microcontroller-interface`
- The Unity VR task driver event source. See `/vr-driver-interface`
- Authoring a new `SessionTypes` member and its descriptor dataclass. Owned by `assets:library-extension`
- Reading, amending, and validating an existing session descriptor. Owned by `assets:session-descriptors`
- Reading and amending the session hierarchy into which the runtime writes. Owned by `assets:session-data`
- Session-data lifecycle after acquisition, meaning preprocess, transfer, and delete. See `/data-management`

---

## The configuration-time / runtime split

The Sollertia platform inherits the ataraxis principle: **AI assistance operates at configuration time, and runtime
acquisition is fully deterministic and AI-independent.** The MCP tool surface intentionally has no "start a session"
tool, and there will not be one. A runtime is launched only through the system's CLI, which reads validated
configuration and descriptor files written during the AI-assisted phases.

A runtime layer therefore has two consumers:

- **Configuration time (AI-assisted):** the AI writes the validated system-level and experiment-level configuration
  files through the MCP tool surface during the earlier setup phases.
- **Runtime (deterministic):** the CLI forwards its flags to the per-mode logic function, which reads those
  configuration files, builds the session's `SessionData` and descriptor, and drives the hardware with no further AI
  involvement.

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

The orchestrator's *construction* and *shutdown ordering* are governed by `/acquisition-system-design` (Layer 3).
This skill governs what the orchestrator does between start and stop.

---

## Per-mode runtime logic functions

Each runtime mode a system supports has one top-level function that is the public entry point for a session of that
mode. Every such function follows the same shape:

1. Validate inputs, meaning read the AI-written configuration files and verify project and animal membership, then
   build the session's `SessionData` data hierarchy.
2. Build the session descriptor: instantiate it with defaults, inherit parameters from the animal's previous session
   of this mode when one exists on disk, and apply the per-flag overrides forwarded by the CLI.
3. Construct the runtime orchestrator (Layer 3 of `/acquisition-system-design`), passing the in-memory descriptor,
   when the mode drives the full hardware stack. Such modes delegate `DataLogger` ownership to the orchestrator. A
   mode that drives a reduced hardware subset skips the orchestrator and creates and starts a standalone
   `DataLogger` itself. That logger writes into the session's raw-data directory when the mode records a session,
   and into a discarded temporary directory when it performs maintenance.
4. Drive the state transitions and the per-cycle runtime loop for the session's lifetime.
5. Tear down in the reverse order on completion or interrupt.

These functions are the only supported surface for starting a session, and direct construction of the orchestrator
from notebooks or scripts is not supported. Modes that perform maintenance rather than recording, such as
calibration and positioning, follow the same shape without a session descriptor.

---

## The orchestrator's dynamic responsibilities

### Two state axes

A runtime distinguishes two orthogonal state axes:

- **System state**: the hardware configuration the system is currently in, meaning which actuators are engaged and
  which sensors are active. It is a system-specific enumeration with one member per hardware mode. A dedicated
  method per state drives the binding classes into that configuration and logs the transition. These state methods
  are **convergent and unconditional**. They never short-circuit on the current system state, so re-entering a state
  re-runs its full configuration sequence and logs a fresh transition. The hardware itself is not necessarily
  re-commanded, though, because each per-actuator driver wrapper short-circuits on its own cached state, so
  re-entry only drives the actuators not already in the target setting.
- **Runtime state (stage)**: an integer "stage" code that partitions a session into an ordered sequence of phases,
  such as trial blocks. Unlike system state, it touches **no hardware**. A single method updates the cached stage
  and logs it, with no binding-class calls. The stage vocabulary and the points at which it advances are
  **session-plan data defined by the experimenter**, not hardware policy. In an experiment mode the ordered stage
  sequence comes from the experiment configuration and is stepped through by the per-mode logic function, while
  simpler modes may hold a single fixed stage code for the whole session. Its only role is to stamp within-session
  boundaries onto the logged timeline so the downstream behavior pipeline can segment the recording. System state
  is what changes the rig, and runtime state merely marks the transitions for analysis. The pause and resume
  machinery preserves the pre-pause stage across an idle interval and re-logs it on resume, keeping the timeline
  correct.

Both axes are logged to the DataLogger as distinct event codes so downstream processing can reconstruct the full
state timeline.

### The per-cycle runtime loop

The session's heartbeat is a single cycle method the logic function calls repeatedly. Each cycle fans out to one
bounded step per concern, so no single concern can starve the others:

- **Data sync**: read each microcontroller subsystem's current state once per cycle through its
  shared-memory-backed accessors, such as position, lick count, and dispensed volume. Update the orchestrator's
  trackers from the change since the previous cycle, and forward derived quantities to downstream subsystems and
  the visualizer. Microcontroller messages are received and parsed **asynchronously** off the main loop and
  published to shared memory, so this step samples the latest published value rather than a per-cycle message
  backlog.
- **Event dispatch**: consume **at most one** typed event from each asynchronous asset-subsystem driver and
  dispatch it, as described below.
- **UI service**: service the interactive control UI, meaning pause and resume, parameter modifiers, and manual
  commands.
- **Auxiliary tasks**: any system-specific per-cycle bookkeeping, such as external-acquisition sync.

Keeping each step bounded per cycle is the core latency contract. The loop must return promptly so the keepalive to
every microcontroller subsystem stays within its interval, as covered by `/acquisition-system-design`.

### Typed-event dispatch from hardware subsystems

Asynchronous hardware subsystems, such as the VR task driver, surface their per-cycle messages as **typed events**
rather than raw payloads. The subsystem's driver parses the transport message and returns a small typed value, an
event kind plus any payload fields. The orchestrator then switches on the kind and acts on its own hardware, for
example by delivering a reward, pulsing a brake, or entering an emergency pause. This keeps transport parsing
inside the subsystem and hardware policy inside the orchestrator.

The Unity VR task driver is the platform's shared asynchronous subsystem, and its `VRTaskEvent` is the event value
a runtime dispatches. A runtime builds the driver for the session types that `SESSION_TYPES_USING_VR_TASK` lists
(`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`) and holds `None` for every other session
type, so every consumer site is guarded by a `None` check. See `/vr-driver-interface`.

### Lifecycle

`start()` brings the session up through semi-interactive hardware preparation, the cycle loop runs the session, and
`stop()` tears it down. Pause and resume implement an idle state that produces no valid data, and terminate handles
end-of-session shutdown. Construction and teardown ordering is owned by `/acquisition-system-design`.

### Teardown isolation

Every teardown step of a multi-asset shutdown runs through `run_shutdown_step(description, step)`, which runs the
step, catches `(Exception, KeyboardInterrupt)`, and echoes an ERROR so the later steps still run
(`cross_system/shutdown_tools.py`). Isolation is load-bearing, because a propagating failure would skip the
remaining steps and leave orphaned subprocesses for the garbage collector, which tears their shared-memory managers
down out of order. The `description` argument is a gerund phrase naming the step, such as "stopping the cameras". A
runtime whose `start()` never completed tears down through a dedicated emergency path rather than the normal
`stop()` ordering, because the ordered path assumes assets that bring-up never reached.

---

## Session initialization marker

`SessionData.create` touches `raw_data/nk.bin` before it publishes the discoverable session marker, so a freshly minted
session is flagged as uninitialized from the moment it exists
(`sollertia-shared-assets/src/sollertia_shared_assets/data_hierarchy/session_data.py`). The per-mode logic function
calls `session_data.mark_runtime_initialized()` once the hardware is up and the session is ready to acquire, and that
call unlinks the marker (`session_data.py`). An aborted initialization therefore leaves the marker in place, and the
marker gates snapshot writing, preprocessing, and purge confirmation downstream. The session hierarchy that holds the
marker is owned by `assets:session-data`.

---

## Required raw-asset snapshots

A runtime writes the session's required raw assets while it brings the session up, before acquisition begins.
`SessionData.required_raw_assets()` is the authority on the required set
(`sollertia-shared-assets/src/sollertia_shared_assets/data_hierarchy/session_data.py`). Every session requires the
session descriptor and a frozen copy of the acquisition-system configuration in effect at acquisition time. A session
that names an experiment also requires the experiment-configuration snapshot, and a session whose type sits in
`SESSION_TYPES_USING_VR_TASK` also requires the VR-configuration snapshot. `SessionData.create` copies both of those
conditional snapshots at session creation (`session_data.py`), which leaves the descriptor and the system-configuration
snapshot for the runtime itself to write.

Beyond the required set, a runtime also freezes the configuration of every active hardware module into the session as
`hardware_state.yaml`, so downstream processing can interpret the raw data without the acquisition host
(the `hardware_state_path` field of `RawData` in `session_data.py`). The parsing class is dispatched per acquisition
system through `HARDWARE_STATE_REGISTRY` in `registries.py`, and the import-time `_assert_registry_coverage` check in
that module requires every registered system to have an entry. Authoring and registering that dataclass is
`assets:library-extension` work, and `assets:session-hardware-state` reads and validates the record the runtime
writes.

Nothing in sollertia-experiment verifies that a runtime wrote these files. A missing asset surfaces only when a health
check or a downstream consumer asks for it, so treat the three writes as part of the runtime contract. See
`/system-health-check`.

---

## Pre-start checkpoint and pause clock

A runtime starts paused, so the operator controls the moment acquisition begins. Once the hardware is up, the
orchestrator services a pre-start checkpoint loop that lets the operator exercise the actuators, and it leaves that
loop when the operator marks setup complete or aborts. Consumables the operator dispenses before the session starts
are folded into a separate accumulator, which keeps the session's own consumption total clean.

The pause clock is not restarted while the runtime is already paused, so a repeated pause request cannot inflate
the accumulated idle time. The per-mode logic function reads the accumulated pause time each cycle and extends the
current stage's deadline by it, so an idle interval does not consume that stage's duration. A logic function that runs
stages back to back zeroes the accumulator at each stage boundary, so a later stage does not inherit an earlier
stage's idle time. A mode whose stages are self-timed intervals may instead subtract the accumulated pause from the
interval, letting the idle time count against it.

---

## Operator interaction

An acquisition runtime is semi-interactive and blocks on terminal prompts wherever an operator must act or decide.
The shared helpers are `wait_for_enter`, `request_confirmation`, `request_required_confirmation`, `request_text`,
and `request_selection` (`cross_system/terminal_prompts.py`). A high-stakes prompt uses
`request_required_confirmation`, which has no default and re-prompts until an explicit yes or no
(`cross_system/terminal_prompts.py`), so an accidental Enter keypress cannot decide the outcome.

Hardware calibration and hardware positioning are experimenter-operated through a maintenance GUI. You MUST NOT
write a runtime instruction that has an agent drive them.

---

## Session-descriptor consumption

The session descriptor holds the runtime parameters for one session, authored as a dataclass owned by
`sollertia-shared-assets`. See `assets:session-descriptors`.

- The per-mode logic function instantiates the descriptor with defaults, inherits parameters from the animal's
  previous session of the same mode when one exists on disk, and applies the per-flag overrides forwarded by the
  CLI.
- The orchestrator receives the finalized descriptor in memory, caches it to disk at construction so an interrupted
  session can still be preprocessed, and uses it to parameterize the state machine.
- At session end the orchestrator updates the descriptor in place with runtime-discovered values, clears the
  `incomplete` flag to mark the session complete, prompts the operator for notes, and re-saves it. The set of
  runtime-discovered values is per-system. For the current worked example, see `mesoscope:mesoscope-vr-runtime`.

The descriptor therefore carries both the session plan and a summary of its outcome, while the DataLogger archive
carries the full sample-by-sample record. The descriptor makes a session reproducible and human-readable at a
glance, and the archive makes it auditable in full.

---

## Visualizer and control UI

Two operator-facing surfaces are standard:

- **A real-time visualizer** that renders behavior data. It owns its own display buffers rather than reading the
  binding classes' shared memory, so the runtime loop pushes each new sample and event into it through explicit
  update methods. Its display mode is selected per runtime mode.
- **An interactive control UI** for operator actions during a session, meaning pause and resume, threshold
  modifiers, and manual commands. It receives the binding classes' `SharedMemoryArray` trackers at construction and
  reads them directly, which is safe from any process that connects to the array. See `/microcontroller-interface`.

Both are constructed by the orchestrator. The control UI is serviced by the runtime loop's UI-service step. The
visualizer takes pushes from the loop's data-sync and event-dispatch steps, and from the hardware-action helpers
any step calls, and the loop repaints it once per cycle. Adding a runtime mode with new display needs is a change
to the visualizer's mode enumeration and its plotting logic.

A system may surface **two** interactive GUIs, the session control UI above and a separate maintenance UI for the
maintenance mode. Each GUI runs in its own daemon process and owns exactly one `SharedMemoryArray`, and each also
holds read-only wrapper trackers that a binding class owns. A GUI destroys only its own array at shutdown and
leaves the borrowed trackers connected. Within the owned array, a signal-style flag is cleared by the reading
property under the array lock, so one operator press is consumed exactly once, while a value-style slot is read
without clearing and holds the operator's latest setting.

---

## Log message codes

The orchestrator emits a system-specific `IntEnum` of log message codes to the DataLogger. It holds at minimum one code
for each state axis, system state and runtime state, plus codes for guidance-mode changes, for event-driven snapshots,
and for any external-acquisition state the system brackets. New events take the next unused code. These codes are the
contract the downstream behavior pipeline consumes (forging plugin), so a code's meaning is durable and a freed value
is never recycled.

---

## CLI surface pattern

Runtime modes are exposed as subcommands of a runtime subgroup inside the system's CLI group. Shared session
arguments, meaning user, project, animal, and animal weight, are declared on that subgroup, parsed before the
subcommand name, and passed to each mode subcommand through the click context. Each subcommand is a thin wrapper.
It collects its per-mode flag overrides and forwards them, with the inherited session arguments, to the per-mode
logic function that builds the `SessionData` and the descriptor. The CLI is the only public surface for starting a
session.

---

## Workflow: adding a runtime mode

1. **Author the descriptor** (`assets:library-extension`): add the `SessionTypes` member, its
   `SYSTEM_SESSION_TYPES` claim, and the descriptor dataclass, then bump `sollertia-shared-assets`. A member
   without that claim fails the import-time parity check, which raises `RuntimeError` from the module-level
   `_assert_registry_coverage()` call in `sollertia-shared-assets/src/sollertia_shared_assets/registries.py` and takes
   down every `sle` entry point.
2. **Extend the state machine** when the mode needs a new hardware configuration: add a state enum member and a
   state-driving method that logs the transition.
3. **Add a visualizer mode** when the display needs differ from existing modes.
4. **Author the per-mode logic function** following the standard shape above.
5. **Add the CLI subcommand** that collects the mode's flag overrides and forwards them, with the subgroup's shared
   session arguments, to the logic function.
6. **Export and version-bump** the new function and descriptor, then pin the new shared-assets minimum.
7. **Update the per-system runtime skill.** For the current worked example, see `mesoscope:mesoscope-vr-runtime`.

---

## Workflow: building a runtime for a new acquisition system

0. Read `/library-extension` for the seam catalog the new runtime composes and for the ordered cross-repository gating
   conditions. The platform exposes **no generic runtime base class**, so each system writes its own controller against
   the shared seams rather than subclassing a common runtime (see the "Extending the Platform" section of
   `sollertia-experiment/README.md`).
1. Compose the system statically first. See `/acquisition-system-design`.
2. Define the system-state enumeration, one member per hardware mode, and the runtime-state stage codes.
3. Define the log message code enumeration, one code per state axis plus the domain events.
4. Implement the orchestrator's per-cycle loop with one bounded step per hardware subsystem present.
5. Implement the per-mode logic functions for the system's session types.
6. Add the CLI command group and subcommands.
7. Author the per-system runtime skill documenting the concrete states, modes, and CLI.

A new system diverges from the current instance wherever its hardware subsystems differ, through fewer or more
cycle steps, a different state set, and so on. The patterns above are conventions rather than a fixed template, so
apply the ones that fit the system's hardware.

---

## Worked example

`AcquisitionSystems` currently holds one member, so Mesoscope-VR is the only registered acquisition system and the
only concrete instance of every pattern here. Read `mesoscope:mesoscope-vr-runtime` for the concrete states, modes,
GUIs, log codes, and commands, and read this skill for the pattern they instantiate.

---

## Maintenance contract

Update this skill when a **platform-general** runtime convention changes, meaning the cycle-loop contract, the
two-state-axis model, the descriptor parameter and summary lifecycle, the typed-event dispatch pattern, or the CLI
pattern. Do NOT update it for changes to any single system's concrete states, modes, visualizer modes, or CLI
commands, which belong in the per-system runtime skill (`mesoscope:mesoscope-vr-runtime`). When a new acquisition
system's runtime reveals a genuinely shared pattern not captured here, add it.

---

## Related skills

| Skill                            | Relationship                                                                                     |
|----------------------------------|--------------------------------------------------------------------------------------------------|
| `/acquisition-system-design`     | Static composition counterpart (configuration, binding classes, construction and shutdown order) |
| `/library-extension`             | Owns the seam catalog a new system's runtime composes, and the absent generic runtime base class |
| `mesoscope:mesoscope-vr-runtime` | The current worked instance of this pattern                                                      |
| `mesoscope:mesoscope-vr`         | The current worked instance of the static design pattern                                         |
| `/microcontroller-interface`     | Per-module wrapper APIs and the SharedMemoryArray accessors the loop reads                       |
| `/vr-driver-interface`           | The typed-event asset-subsystem source (`VRTaskEvent`) the loop dispatches                       |
| `assets:session-descriptors`     | Reads, amends, and validates the descriptors the runtime writes                                  |
| `assets:session-data`            | Owns the session hierarchy and the `nk.bin` uninitialized-session marker                         |
| `assets:session-hardware-state`  | Reads and validates the `hardware_state.yaml` record the runtime writes                          |
| `assets:library-extension`       | Authors new `SessionTypes` members and their descriptor dataclasses                              |
| `/data-management`               | Post-acquisition session-data lifecycle                                                          |
| `/pipeline`                      | Where the runtime phase sits in the end-to-end lifecycle                                         |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Runtime pattern compliance, reader-judged:
- [ ] Runtime is launchable only via the CLI, with no MCP "start session" tool
- [ ] Per-mode logic functions follow the standard shape (SessionData, descriptor, orchestrator, loop, teardown)
- [ ] System state and runtime state are distinct axes, each logged with its own code
- [ ] System-state methods drive the binding classes into the target configuration and log the transition
- [ ] The per-cycle loop has one bounded step per hardware subsystem, and the keepalive interval is never exceeded
- [ ] Asynchronous subsystems surface typed events, and the orchestrator dispatches on event kind
- [ ] Every teardown step of a multi-asset shutdown is wrapped in run_shutdown_step
- [ ] mark_runtime_initialized() is called once the session is ready to acquire, and never earlier
- [ ] The runtime writes the session descriptor, the system-configuration snapshot, and hardware_state.yaml
- [ ] The runtime starts paused and services a pre-start checkpoint before acquisition begins
- [ ] High-stakes operator prompts use request_required_confirmation
- [ ] The descriptor is built by the logic function and updated with runtime results at session end
- [ ] Log message codes are append-only, with no recycled values

Documentation handoffs, reader-judged:
- [ ] The per-system runtime skill documents the concrete states, modes, and CLI
- [ ] No Mesoscope-VR field name, state member, or CLI subcommand appears in this skill
```
