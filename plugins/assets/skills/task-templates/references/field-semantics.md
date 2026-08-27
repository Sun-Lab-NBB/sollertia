# Task template field semantics

Per-field consumer roles for every `TaskTemplate` key, the firing rule of each of the five `TriggerType` modes, and the
design rationale behind the schema's shape. See [`../SKILL.md`](../SKILL.md) for the template model, the MCP tool
surface, the authoring workflow, and the verification checklist.

---

## How template fields are used

| Field                                 | Consumer-side role                                                                                                                                                                                                                                                                                                       |
|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **`cues`**                            | Unity bakes wall textures from each cue's `texture` asset. The uint8 `code` is the on-the-wire identifier the runtime uses for analysis.                                                                                                                                                                                 |
| **`vr_environment`**                  | Parameterizes corridor geometry: how many segments are visible at once, how parallel corridor instances are spaced, the centimeter to Unity-unit conversion, the padding prefab, and the cue offset that shifts the cue sequence origin relative to each corridor's spawn point. See `unity:task-prefabs` for specifics. |
| **`trial_structures`**                | Spatial config per trial type, covering the cue sequence, stimulus trigger zone bounds, stimulus location, visible-boundary flag, trigger type, and optional transitions. The trigger type tells Unity which zone prefab to bake.                                                                                        |
| **`trial_structures[].cue_sequence`** | Drives Unity's segment-prefab geometry: each trial generates a single segment prefab whose cue ordering matches this sequence. Cue prefab lengths sum to the segment length used by zone validation.                                                                                                                     |
| **`trial_structures[].transitions`**  | Drives Unity's segment-sequence resolver at session init. Sampled to materialize the deterministic trial chain. A null or empty map falls back to uniform-random successor selection.                                                                                                                                    |

After Unity emits the materialized cue sequence at session start, the acquisition runtime **decomposes it back into a
trial timeline** by motif-matching each `TrialStructure`'s cue sequence against the materialized sequence. The trial
timeline is what downstream analysis joins against licks, rewards, and gas puffs. Adding a new trial type to a paradigm
therefore means (a) adding a `TrialStructure` to the template here, and (b) binding it to a concrete trial subclass in
the per-project experiment configuration via `/experiment-configuration`.

---

## The five trigger modes

`TriggerType` carries five modes, each with its own firing rule. All five share one wire contract, owned by
`unity:mqtt-contract`, and the zone prefab each mode bakes is owned by `unity:zone-prefabs`.

| Mode                | Firing rule                                                                                                                                       |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| `interaction`       | The animal must engage an interaction sensor inside the stimulus trigger zone to fire the stimulus.                                               |
| `collision`         | Crossing an invisible boundary wall (a thin collider at `stimulus_location`) fires the stimulus unconditionally, with no sensor and no occupancy. |
| `occupancy_disarm`  | Occupying the zone disarms the boundary. Colliding with the still-armed boundary (occupancy **not** met) fires.                                   |
| `occupancy_arm`     | Occupying the zone arms the boundary. Colliding with the now-armed boundary (occupancy **met**) fires, inverting `occupancy_disarm`.              |
| `occupancy_trigger` | Occupying the zone for the required duration fires the stimulus immediately, with no boundary collision.                                          |

All three occupancy modes read the dwell time from `TrialStructure.occupancy_duration_ms`, and the template is its
single source of truth because no experiment configuration carries a copy.

**System support is a per-system subset.** The platform `TriggerType` enum carries all five, but each acquisition system
maps only the subset its `from_task_template` can resolve to its own runtime trial classes. A configuration that uses an
unmapped mode raises a clear "not mapped to a runtime trial class" error. Adding a new `TriggerType` member therefore
does **not** require a `from_task_template` branch in every system, because a system may leave a mode unsupported. See
`/library-extension` for the cross-cutting recipe. For the current Mesoscope-VR reference system's concrete mapping, see
`mesoscope:mesoscope-vr-experiment-schema`.

---

## Why the template is shaped this way

- **Cues are a flat catalog with unique uint8 codes** because the analysis pipeline indexes wall encounters by code and
  needs deterministic, low-cost packing. The 0 to 255 range caps the vocabulary at 256 cues per template, which has been
  more than sufficient in practice.
- **Cue identity `(name, length_cm)` must resolve to one texture across every template** in the configured templates
  directory. Unity stores cue prefabs and materials filesystem-keyed as `Cue_<name>_<length>cm.prefab` and `.mat`, and
  the generator reuses an existing cue prefab whenever it finds one on disk. Two templates that declare the same cue
  identity with different textures would therefore silently corrupt each other depending on generation order. The
  Unity-side `CreateFromTemplate` preflight scans every template under `Assets/InfiniteCorridorTask/Configurations/`
  before any mutation and aborts the generation request with the offending template pair(s) listed in the error. When
  authoring a new cue or modifying an existing one, either reuse the same texture for the same `(name, length_cm)`
  everywhere or rename or re-length the cue so it occupies a distinct filesystem slot.
- **Each trial structure embeds its own cue sequence** because the Unity task generator derives segment prefab geometry
  directly from the trial's cue sequence, and no separate segment catalog exists. The segment prefab name is
  `<TemplateName>-<TrialName>`, so every trial structure yields a distinct prefab keyed by the trial key, and no
  geometric coincidence between two trials can collapse them into a single prefab.
- **Transitions are a named dict (`{trial_name: probability}`)** because the trial-to-trial topology is the source of
  truth for corridor sequencing. Names make the topology order-independent and self-documenting. Omitted keys carry
  implicit zero probability, so the dict need only enumerate reachable trials.
- **`TrialStructure` is spatial-only** (no rewards, no puff durations), except for `occupancy_duration_ms`, the single
  source of truth for the dwell time used by every occupancy mode (`occupancy_disarm`, `occupancy_arm`,
  `occupancy_trigger`), which Unity reads at generation time. Rewards and puff durations are project-level behavioral
  parameters that vary between teams using the same paradigm. They therefore live on the system's runtime trial classes
  (for Mesoscope-VR, `MesoscopeWaterRewardTrial` and `MesoscopeGasPuffTrial`, see
  `mesoscope:mesoscope-vr-experiment-schema`) authored by `/experiment-configuration`, and are joined to the spatial
  structure by trial name.
- **`trigger_type` is on `TrialStructure`** because Unity must pick the zone prefab during template generation, long
  before any experiment-config trial is instantiated. The trigger type is therefore the template's contract with Unity,
  and the matching experiment-config trial class is the experiment config's contract with the runtime's stimulus
  delivery code.
- **`cue_offset_cm` lives on `vr_environment`** because the cue-origin shift is an attribute of the corridor geometry
  itself, and every Unity-spawned corridor instance sees the same value. On the Unity side, in
  `sollertia-virtual-reality`, it also drives the per-segment ResetZone placement, so the animal's spawn point falls
  inside the reset zone on every lap restart. See the unity plugin's skills for the prefab-generation specifics.
