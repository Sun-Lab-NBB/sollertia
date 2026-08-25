# MesoscopeDriver MQTT contract

Wire-level reference for the MQTT contract between `MesoscopeDriver` and the `runAcquisition` MATLAB function on the
ScanImagePC, and for the pre-flight check that confirms the ScanImagePC side is running. See
[`../SKILL.md`](../SKILL.md) for the driver's place in the system composition.

---

## Construction and composition

`MesoscopeDriver` (in `sollertia_experiment/mesoscope_vr/mesoscope_driver.py`) encapsulates all MQTT
communication with the `runAcquisition` MATLAB function on the ScanImagePC.

Construction signature:

```python
MesoscopeDriver(
    configuration: VRTaskConfiguration,
    acquisition: MesoscopeAcquisition,
)
```

Mesoscope control is tightly coupled to the Virtual Reality task, so the driver reuses the shared
Virtual Reality MQTT broker discovery fields (`ip` / `port`) from `assets.vr_task` rather than
defining its own. The orchestrator constructs the driver from `assets.vr_task` and
`acquisition`, and `connect()`s it only for mesoscope experiment sessions.

---

## Method surface

| Method                       | Role                                                                          |
|------------------------------|-------------------------------------------------------------------------------|
| `connect()` / `disconnect()` | Open / close the MQTT connection to the ScanImagePC                           |
| `await_alive()`              | Probe ScanImagePC liveness with a request-reply handshake on the Status topic |
| `is_alive(timeout_ms)`       | One-shot bounded liveness probe (non-blocking) for the pre-flight check       |
| `preload(project, animal)`   | Preload the persisted per-animal reference estimator as an alignment aid      |
| `generate_reference()`       | Generate the fresh session estimator + high-definition z-stack and arm        |
| `begin_acquisition()`        | Begin acquiring session frames                                                |
| `abort()`                    | Abort or end the ongoing frame acquisition                                    |
| `recover()`                  | Reload the session estimator and re-arm after a transient interruption        |
| `query_state()`              | Return a `MesoscopePositions` snapshot of the stage / fast-Z / laser state    |

Each command is published, resent on each acknowledgement timeout, and confirmed by a reception
acknowledgement on the Status topic; setup commands additionally block for a terminal status state.
The driver probes liveness by publishing an empty `MesoscopeAlive` request and waiting for the
Status-topic acknowledgement — the absence of a reply within the timeout means the `runAcquisition`
command loop is not running. Actual frame acquisition start/stop is confirmed by the caller through
the hardware TTL frame stream, not over MQTT.

---

## MQTT topic namespace

The driver and the MATLAB function exchange messages on a flat `Mesoscope`-prefixed PascalCase topic
namespace that does not overlap with the Unity task topics, so both surfaces share one broker. Ten
topics are defined: the VRPC publishes the command topics `MesoscopeAlive`, `MesoscopePreload`,
`MesoscopeGenerateReference`, `MesoscopeBeginAcquisition`, `MesoscopeAbort`, `MesoscopeRecover`, and
`MesoscopeQueryState`; the ScanImagePC publishes the reply topics `MesoscopeStatus` (reception
acknowledgement and progress), `MesoscopeError` (failure detail), and `MesoscopeState` (the
stage / fast-Z / laser snapshot answering a state query).

---

## Command payloads

The `acquisition` parameters are carried in the command payloads, not configured on the ScanImagePC:
`generate_reference()` ships the full acquisition parameter set, `recover()` ships only the
plane-geometry parameters (`z_step_um`, `z_range_um`, `z_exclusion_um`, `acquisition_order`) needed
to re-derive the imaging planes, and `preload()` ships the `project` and `animal` identifiers the
ScanImagePC uses to resolve the persisted estimator path under its local data root. The remaining
commands carry empty payloads. This makes the `MesoscopeAcquisition` section the single source of
truth for the acquisition geometry.

---

## The runAcquisition MATLAB counterpart

The ScanImagePC counterpart is the `runAcquisition` MATLAB function
(`assets/mesoscope_vr/runAcquisition.m`), a top-level script deployed to the ScanImagePC. It connects
to the shared broker, then enters an MQTT command loop that acts as a state machine dispatching these
commands to the ScanImage software (preload, generate reference, begin / abort / recover, and answer
the liveness probe and state queries). Only the ScanImagePC-local output root and the broker address
remain function arguments; every acquisition parameter arrives in the command payloads.

---

## Pre-flight bridge check

Before a Mesoscope imaging session (`window-checking` or `experiment`), confirm the ScanImagePC's
`runAcquisition` control loop is reachable. `runAcquisition` is a **lock-in** command loop: the operator launches
it once in the MATLAB command line on the ScanImagePC, and it runs continuously — holding the command line for the
whole runtime — until interrupted with Ctrl-C or the broker drops. An unreachable bridge means it is not running,
so the runtime cannot arm or command the Mesoscope.

This skill owns the mesoscope-specific bridge check; the platform-general `experiment:system-health-check` hands
off here for it (the ScanImage bridge is exclusive to Mesoscope-VR, unlike the shared Unity bridge). The check is
backed by `MesoscopeDriver.is_alive()` — a single bounded `MesoscopeAlive` probe that, unlike the runtime's
interactive `await_alive()`, does not block or re-prompt. It is exposed on two surfaces, both mesoscope-specific
(NOT on the hardware-agnostic `sle get` surface):

- **MCP**: `check_mesoscope_bridge_tool` (`sle mcp`) — returns `{"reachable": ..., "status": ...}`, or
  `{"error": ...}` on failure.
- **CLI**: `sle mesoscope check-bridge` — prints a status line at SUCCESS (reachable) or WARNING (unreachable).

Both load the active configuration, resolve the shared broker (`assets.vr_task.ip` / `port`), probe once, and
disconnect. On an unreachable result, the remediation is to launch `runAcquisition(hSI, hSICtl, ...)` in MATLAB on
the ScanImagePC, then re-check.
