# Assembled column emission rules

Per-emission-site rules a new Mesoscope-VR dataset column must satisfy, one section per sub-assembler that carries a
rule beyond naming the column. See [`../SKILL.md`](../SKILL.md) for the emission-site table that names which
sub-assembler owns each `DatasetColumn` group.

---

## Behavior columns and the final_columns select

A behavior column omitted from `final_columns` is dropped by the closing `select` in `behavior_dataset.py`, which is the
one emission site where the value is computed and then silently discarded.

---

## Pupil metrics and pupil state flags

A new continuous pupil metric is declared as a `PupilColumn` member in `video_tracking.py`, which
`/mesoscope-vr-video-tracking` owns, and it reaches the assembled frame with no edit in `video_dataset.py`, because the
assembler iterates the pupil feather's own columns. A new boolean pupil state flag also joins the `_PUPIL_FLAG_COLUMNS`
frozenset in `video_dataset.py`, which holds `PupilColumn.BLINKING_STATE` and `PupilColumn.DILATION_STATE` today. The
assembler branches on membership in that set to pick nearest-prior interpolation and a `uint8` cast, so a flag left out
of it takes the linear blend and lands as an interpolated float rather than a state.

---

## String literals against DatasetColumn members

`two_photon_dataset.py` is the only sub-assembler that imports `DatasetColumn`, so the other three name their columns as
string literals and review is what holds the member and the emitted name in step.

---

## Runtime columns and run-state masking

A new runtime column carrying no meaning outside the run state also needs its own branch in
`mask_non_run_experiment_data`, with the sentinel that branch writes.
