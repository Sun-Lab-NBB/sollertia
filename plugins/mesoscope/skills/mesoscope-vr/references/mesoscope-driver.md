# MesoscopeDriver MQTT contract

Wire-level reference for the MQTT contract between `MesoscopeDriver` and the `runAcquisition` MATLAB function on the
ScanImagePC, and for the pre-flight check that confirms the ScanImagePC side is running. See
[`../SKILL.md`](../SKILL.md) for the driver's place in the system composition.

---

## Construction and composition

`MesoscopeDriver` (in `mesoscope_vr/mesoscope_driver.py`) encapsulates all MQTT communication with the
`runAcquisition` MATLAB function on the ScanImagePC.

Construction signature: `MesoscopeDriver(configuration: VRTaskConfiguration, acquisition: MesoscopeAcquisition)`.

Mesoscope control shares the broker with the Virtual Reality task, so the driver reuses the `ip` and `port` fields of
`assets.vr_task` as its own broker discovery rather than defining a second pair (`MesoscopeDriver.__init__` in
`mesoscope_vr/mesoscope_driver.py`). It builds its own `MQTTCommunication` client monitoring the three reply topics
and uses no part of `VRTaskDriver`. The orchestrator constructs the driver from `assets.vr_task` and `acquisition` for
every session type it runs, and `connect()`s it only for mesoscope experiment sessions (`MesoscopeVRSystem.__init__`
in `mesoscope_vr/system_controller.py`). Window checking runs outside the orchestrator and builds its own driver,
connecting it just before the mesoscope preparation sequence (`window_checking_logic` in
`mesoscope_vr/data_acquisition.py`).

---

## Timeouts

| Constant                           | Value     | Governs                                                      |
|------------------------------------|-----------|--------------------------------------------------------------|
| `_BROKER_POLL_DELAY_MS`            | `10`      | Delay between consecutive reply-buffer polls                 |
| `_ACK_TIMEOUT_MS`                  | `5000`    | Reception acknowledgement, the liveness probe, a state query |
| `_PRELOAD_TIMEOUT_MS`              | `120000`  | Terminal wait for `preload()`                                |
| `_REFERENCE_GENERATION_TIMEOUT_MS` | `1800000` | Terminal wait for `generate_reference()`, 30 minutes         |
| `_RECOVERY_TIMEOUT_MS`             | `300000`  | Terminal wait for `recover()`, 5 minutes                     |

They are module-level constants of `mesoscope_vr/mesoscope_driver.py`.

---

## Method surface

| Method                       | Terminal status state | Role                                                                       |
|------------------------------|-----------------------|----------------------------------------------------------------------------|
| `connect()` / `disconnect()` | none                  | Open / close the MQTT connection to the ScanImagePC                        |
| `await_alive()`              | `received`            | Blocking, re-prompting liveness handshake on the Status topic              |
| `is_alive(timeout_ms=5000)`  | `received`            | One bounded, non-interactive liveness probe for the pre-flight check       |
| `preload(project, animal)`   | `preload_complete`    | Preload the persisted per-animal reference estimator as an alignment aid   |
| `generate_reference()`       | `armed`               | Generate the fresh session estimator and high-definition z-stack, then arm |
| `begin_acquisition()`        | acknowledgement only  | Begin acquiring session frames                                             |
| `abort()`                    | acknowledgement only  | Abort or end the ongoing frame acquisition                                 |
| `recover()`                  | `armed`               | Reload the session estimator and re-arm after a transient interruption     |
| `query_state()`              | State-topic reply     | Return a `MesoscopePositions` snapshot of the stage, fast-Z, and laser     |

Each command is published, resent on each acknowledgement timeout, and confirmed by a reception acknowledgement on the
Status topic. The setup commands additionally block for their terminal status state, echoing the intermediate states
to the operator. A `MesoscopeError` reply becomes a `RuntimeError`. The absence of a liveness reply within the
acknowledgement timeout means the `runAcquisition` command loop is not running. Frame acquisition start and stop are
confirmed by the caller through the hardware TTL frame stream rather than over MQTT.

`query_state()` maps the reply keys `x`, `y`, `r`, `z`, `fast_z`, `tip`, `tilt`, and `power_mW` onto
`MesoscopePositions`. It leaves `red_dot_alignment_z` at its default, because that value cannot be queried and the
caller collects it from the operator (`mesoscope_vr/mesoscope_driver.py`).

---

## MQTT topic namespace

The driver and the MATLAB function exchange messages on a flat `Mesoscope`-prefixed PascalCase topic namespace that
does not overlap the Unity task topics, so both surfaces share one broker (the `_MesoscopeMQTTTopics` enum in
`mesoscope_vr/mesoscope_driver.py`). Ten topics are defined. The VRPC publishes the seven command topics
`MesoscopeAlive`, `MesoscopePreload`, `MesoscopeGenerateReference`, `MesoscopeBeginAcquisition`, `MesoscopeAbort`,
`MesoscopeRecover`, and `MesoscopeQueryState`. The ScanImagePC publishes the three reply topics `MesoscopeStatus`
(reception acknowledgement and progress), `MesoscopeError` (failure detail), and `MesoscopeState` (the stage, fast-Z,
and laser snapshot answering a state query).

---

## Status states

`MesoscopeStatus` carries one of eight state values, defined by the `_MesoscopeStatusState` enum in
`mesoscope_vr/mesoscope_driver.py`:

| State                  | Meaning                                                                      |
|------------------------|------------------------------------------------------------------------------|
| `received`             | The command was received and processing started. This is the acknowledgement |
| `preloading`           | The persisted reference estimator is loading                                 |
| `preload_complete`     | The persisted estimator loaded, with automatic correction left disabled      |
| `generating_estimator` | The reference volume is being acquired and the fresh session estimator built |
| `acquiring_zstack`     | The high-definition reference z-stack is being acquired                      |
| `armed`                | The Mesoscope is armed and ready to begin frame acquisition                  |
| `grabbing`             | Session frame acquisition started                                            |
| `stopped`              | Frame acquisition stopped                                                    |

---

## Command payloads

The `acquisition` parameters are carried in the command payloads rather than configured on the ScanImagePC
(`_encode_acquisition` in `mesoscope_vr/mesoscope_driver.py`). `_encode_acquisition` always ships the four
plane-geometry parameters `z_step_um`, `z_range_um`, `z_exclusion_um`, and `acquisition_order`. The full payload used by
`generate_reference()` adds `registration_channel`, `field_curvature_correction`, `frames_per_reference_plane`, and
`zstack_scale_factor`, while `recover()` sends the geometry-only form because it re-derives the imaging planes without
regenerating the z-stack. `preload()` ships the `project` and `animal` identifiers the ScanImagePC uses to resolve the
persisted estimator path under its local data root. The remaining commands carry empty payloads. This makes the
`MesoscopeAcquisition` section the single source of truth for the acquisition geometry.

---

## The runAcquisition MATLAB counterpart

The ScanImagePC counterpart is the `runAcquisition` MATLAB function (`assets/mesoscope_vr/runAcquisition.m` in
sollertia-experiment), a top-level script deployed to the ScanImagePC. It connects to the shared broker, then enters
an MQTT command loop that acts as a state machine dispatching these commands to the ScanImage software: preload,
generate reference, begin, abort, recover, and answer the liveness probe and state queries. Only the
ScanImagePC-local output root and the broker address remain function arguments, and every acquisition parameter
arrives in the command payloads.

---

## Pre-flight bridge check

Before a Mesoscope imaging session, confirm the ScanImagePC's `runAcquisition` control loop is reachable.
`runAcquisition` is a **lock-in** command loop: the operator launches it once in the MATLAB command line on the
ScanImagePC, and it holds that command line for the whole runtime until it is interrupted with Ctrl-C or the broker
drops. An unreachable bridge means it is not running, so the runtime cannot arm or command the Mesoscope.

This skill owns the mesoscope-specific bridge check, and the platform-general `experiment:system-health-check` hands
off here for it, because the ScanImage bridge belongs to this system alone while the Unity bridge is shared. The check
is backed by `MesoscopeDriver.is_alive()`, a single bounded `MesoscopeAlive` probe that returns instead of blocking
and re-prompting the way the runtime's `await_alive()` does. It is exposed on two mesoscope-specific surfaces rather
than on the hardware-agnostic `sle get` surface:

- **MCP**: `check_mesoscope_bridge_tool` (`sle mcp`) returns `{"reachable": bool, "status": str}`, or `{"error": ...}`
  on failure.
- **CLI**: `sle mesoscope check-bridge` takes no options and prints a status line at SUCCESS when reachable and at
  WARNING when unreachable.

Both load the active configuration, resolve the shared broker (`assets.vr_task.ip` / `port`), connect, probe once, and
disconnect in a `finally` block (`check_mesoscope_bridge` in `mesoscope_vr/mesoscope_driver.py`). On an unreachable
result, the remediation is to launch `runAcquisition(hSI, hSICtl, <parameters>)` in MATLAB on the ScanImagePC, then
re-check.

You MUST NOT command the Mesoscope yourself through any of these surfaces beyond the read-only bridge probe.
Acquisition, reference generation, and recovery are operator-gated steps of the runtime documented in
`/mesoscope-vr-runtime`.
