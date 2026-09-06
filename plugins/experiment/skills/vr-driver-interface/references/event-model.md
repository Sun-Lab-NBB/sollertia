# VR task event and trial model

Carries the typed-event vocabulary `cycle()` surfaces and the cue-sequence decomposition that turns Unity's flat
wall-cue array into a trial sequence. Loaded on demand from `vr-driver-interface`'s SKILL.md, which owns the driver
surface these two models travel over. Paths are relative to `src/sollertia_experiment/`.

---

## Event model

`cycle()` consumes **at most one** MQTT message per call and returns a typed `_VRTaskEvent`. The asynchronous Unity
messages it surfaces are enumerated by `VRTaskEventKind` (`IntEnum`):

| Kind                      | Value | Source topic                        | Meaning / caller action                                               |
|---------------------------|-------|-------------------------------------|-----------------------------------------------------------------------|
| `NONE`                    | 0     | (buffer empty or a handshake topic) | No dispatchable event this cycle                                      |
| `STIMULUS_TRIGGERED`      | 1     | `STIMULUS`                          | Trial resolved. `delivered` drives the actuator, `cause` marks guided |
| `TRIGGER_DELAY_REQUESTED` | 2     | `DELAY`                             | Unity requests a brake pulse of `delay_ms` milliseconds               |
| `UNITY_TERMINATED`        | 3     | `SESSION_STOP`                      | Unity runtime ended. The system must enter an emergency pause         |

`SessionStop` publishes only from `MQTTClient.OnApplicationQuit`, which the Editor raises when Play Mode ends, so
`UNITY_TERMINATED` marks the whole Unity session ending. A `Task` that disables itself mid-run instead leaves the Editor
playing and the MQTT client connected, so `cycle()` keeps returning `NONE` while the corridor stops advancing.
`unity:task-generator` catalogues those bailouts, and `read_console_tool` at `level="error"` reads the logged error.

`_VRTaskEvent` (frozen slots dataclass) carries `kind: VRTaskEventKind` plus `delay_ms: int = 0`, populated only for
`TRIGGER_DELAY_REQUESTED`. Three further fields are populated only for `STIMULUS_TRIGGERED` and parsed from the
`Stimulus` payload: `trial_name: str = ""`, `delivered: bool = True`, and `cause: StimulusCause = BEHAVIOR`, where
`StimulusCause` is an exported `StrEnum` of `behavior` and `guidance`. Every no-event cycle returns the shared
frozen `_NO_EVENT` singleton (`vr_task/driver.py`), and a handshake topic consumed during `cycle()` resolves
to `NONE`, because the setup sequence handles those instead.

`_VRTaskState` (slots dataclass, the driver's `state` property) is the single source of truth shared between
the setup handshake and per-cycle events:

| Field                          | Type             | Default | Purpose                                        |
|--------------------------------|------------------|---------|------------------------------------------------|
| `position`                     | `np.float64`     | `0.0`   | Current absolute animal position (Unity units) |
| `cue_sequence`                 | `NDArray[uint8]` | empty   | The session's flattened wall-cue sequence      |
| `terminated`                   | `bool`           | `False` | Whether Unity has unexpectedly terminated      |
| `reinforcing_guidance_enabled` | `bool`           | `False` | Reinforcing-trial guidance state               |
| `aversive_guidance_enabled`    | `bool`           | `False` | Aversive-trial guidance state                  |


---

## Trial decomposition

`vr_task/trial_decomposition.py` turns Unity's flat wall-cue sequence into a trial sequence the acquisition system
can act on, using the per-trial cue motifs from the `TaskTemplate`.

- `decompose_cue_sequence(cue_sequence, task_template, motif_decomposer)` matches the flat cue array against the
  template's per-trial motifs and produces `DecomposedTrials`.
- `CachedMotifDecomposer` caches the flattened motif data between successive decomposition runs, so
  re-decomposition after a Unity restart is cheap. It returns the cache whenever the motifs are element-wise equal to
  the previous call, and otherwise sorts them longest-first before flattening.
- `DecomposedTrials` (frozen slots dataclass) holds aligned per-trial sequences, where index `i` is the i-th trial:

| Field                  | Type               | Purpose                                                                                   |
|------------------------|--------------------|-------------------------------------------------------------------------------------------|
| `cumulative_distances` | `NDArray[float64]` | Cumulative distance (cm) to reach the end of each trial                                   |
| `trial_names`          | `tuple[str, ...]`  | Join key the runtime uses to look up per-trial parameters in its experiment configuration |

`decompose_cue_sequence` raises through `console.error` in three cases (`vr_task/trial_decomposition.py`).
A template declaring no trial structures raises `ValueError`. A motif that a run of two or more other motifs
reproduces exactly raises `ValueError` naming the offending trial, which is the word-break ambiguity check in
`_find_ambiguous_motif` (`vr_task/trial_decomposition.py`). A greedy scan that matches no motif raises
`RuntimeError` reporting the failure position and the next 20 unmatched cues.

Exactly one function is jitted, `@njit(cache=True) _decompose_sequence_numba_flat` (`vr_task/trial_decomposition.py`).
It accepts only flat typed arrays and an integer, which is why `decompose_cue_sequence` pre-flattens the motifs and
sizes the output with `max_trials = len(cue_sequence) // min(motif length) + 1`.

The driver's `trial_names` are the join key a system's experiment configuration uses to look up per-trial structures.
The `TriggerType` taxonomy lives in `sollertia-shared-assets` and is owned by `assets:library-extension` and
`assets:experiment-configuration`. For the mapping from trigger types to a system's runtime trial classes and outcome
arrays, see `mesoscope:mesoscope-vr-experiment-schema`.

All trial modes share one MQTT and wire contract, because every mode publishes the same `Stimulus` event and adds no
topics (see the [MQTT topic contract](../SKILL.md#mqtt-topic-contract)). The Unity-side dispatch, prefab reuse, and
mode-aware template geometry are owned by `unity:zone-prefabs`, `unity:task-generator`, and `assets:task-templates`.

