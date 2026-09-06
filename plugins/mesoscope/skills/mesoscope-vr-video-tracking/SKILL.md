---
name: mesoscope-vr-video-tracking
description: >-
  Documents the Mesoscope-VR pupil-tracking pass and video sub-dataset assembler donated to the sollertia-forgery video
  and forging pipelines. Covers the pose-prediction locator, the thirteen canonical bodyparts and the ring-ellipse fit,
  the nineteen pupil columns with the blink and dilation partition, and the per-camera video sub-dataset with its
  interpolation split and slowest-camera clock. Use when interpreting the face-camera pupil feather, tracing a NaN or a
  blink flag back to its rule, or debugging the video columns of a forged dataset.
user-invocable: false
---

# Mesoscope-VR video tracking

Documents the two Mesoscope-VR modules that turn face-camera and body-camera recordings into per-frame metrics, and then
into the video columns of a forged dataset. `sollertia_forgery.mesoscope_vr.video_tracking` post-processes the
externally-produced DeepLabCut predictions into `face_camera_pupil.feather`, and
`sollertia_forgery.mesoscope_vr.video_dataset` interpolates every per-camera feather onto the assembly reference clock.

The token `video_tracking` names two different things on the two sides of the platform. On the acquisition side it is
the `MesoscopeVideoTracking` configuration section that runs `slvt infer` during preprocessing and writes the DeepLabCut
`.h5` beside the face-camera video, documented by `/mesoscope-vr`. On the forgery side, covered here, it is the module
that reads that `.h5` back and never runs inference. The acquisition section is the producer, this module is the
consumer, and the two live in different libraries.

---

## Scope

**Covers:**
- The two registry seams this pair of modules fills, `_POSE_PREDICTION_REGISTRY` and `_VIDEO_TRACKING_REGISTRY`
- `locate_mesoscope_pose_predictions`, its glob pattern, its natural-sort tie-break, and its two consumers
- The fourteen module-level constants of `video_tracking.py`, including the thirteen canonical bodyparts and the
  load-bearing pupil and eye ring order
- `_read_points_from_h5`, its MultiIndex column keying, and its two `ValueError` messages
- `_fit_ring_ellipse`, its least-squares model, its confidence-pattern grouping, its rejection rule, and the
  condition under which it records a residual
- The three derived geometry formulas, the openness baseline, the blink rule, and the measured, dilated, or blinked
  partition of every frame
- `PupilColumn`, the nineteen columns of `face_camera_pupil.feather`, their dtypes, and their per-column NaN masks
- `assemble_video_dataset`, the two camera sources, the output column naming, the interpolation split, the row-count
  mismatch error, and the empty-frame return
- `resolve_slowest_camera_clock`, its minimum frame count, its rate comparison, its echo, and its failure

**Does not cover:**
- The acquisition-side `video_tracking` configuration section, `MesoscopeVideoTracking`, and the `slvt infer` call
  that produces the DeepLabCut `.h5`. Owned by `/mesoscope-vr`.
- The agnostic video pipeline, its four job names, and the prepare-then-execute batch mechanics that run the
  tracking job. Owned by `forging:batch-processing`.
- The processed-data directory layout and the per-stage output locations on disk. Owned by
  `forging:processing-results`.
- The raw camera log archives and the upstream frame-timestamp extraction that writes the timestamp feathers.
  Owned by `video:log-processing` and `forging:processing-input-format`.
- The `VideoDataFiles` filename roster, the `DatasetColumn` roster, and `MESOSCOPE_COLUMN_DESCRIPTIONS`. Owned by
  `/mesoscope-vr-processing-schema`.
- The per-session-type assembly dispatch that chooses the reference clock and concatenates the sub-datasets. Owned
  by `/mesoscope-vr-dataset-assembly`.
- The mesoscope fluorescence clock that serves as the experiment-session reference time. Owned by
  `/mesoscope-vr-fluorescence-alignment`.
- The `_ASSEMBLY_GEOMETRY_REGISTRY` and `_ASSEMBLY_SOURCE_REGISTRY` sizing donations that call
  `resolve_reference_clock_samples` and `count_camera_source_samples`. Owned by `forging:data-processing-design`.
- The donation protocols and the import-time coverage check that bind these functions to their registries. Owned by
  `forging:data-processing-design`.

**Handoff rules:** a question about how the `.h5` is produced, about the DeepLabCut environment, or about the project
path goes to `/mesoscope-vr`. A question about running, re-running, or sizing the tracking job goes to
`forging:batch-processing`. A question about which columns reach `data.feather` for a given session type goes to
`/mesoscope-vr-dataset-assembly`.

---

## The two registry seams

Mesoscope-VR fills two of the thirteen registries in `sollertia_forgery.registries` from `video_tracking.py`. The
locator and the tracking pass are separate donations because the pipeline needs the locator before it is willing to
schedule the pass.

| Registry                    | Donated asset                       | Public resolver                   | Consumer                                    |
|-----------------------------|-------------------------------------|-----------------------------------|---------------------------------------------|
| `_POSE_PREDICTION_REGISTRY` | `locate_mesoscope_pose_predictions` | `resolve_pose_prediction_locator` | Video job discovery and the job sizing pass |
| `_VIDEO_TRACKING_REGISTRY`  | `process_mesoscope_video_tracking`  | `resolve_video_tracking`          | The `pose_tracking` job runner              |

Job discovery calls the locator to decide whether a session supports a tracking job at all.
`orchestration/footprints.py` calls it again to charge the job the width of the table that file holds, read from the
table's own row and column counts rather than from the file's size on disk. A single donation covering both would force
discovery to load and parse the predictions merely to decide whether to schedule work.

`video_dataset.py` fills no registry of its own. `assemble_video_dataset` and `resolve_slowest_camera_clock` are
reached through the Mesoscope-VR assembly worker registered in `_FORGING_ASSEMBLY_REGISTRY`, which
`/mesoscope-vr-dataset-assembly` owns. Its two other public functions, `resolve_reference_clock_samples` and
`count_camera_source_samples`, are reached through the sizing donations instead. `resolve_mesoscope_assembly_geometry`
in `training_dataset.py` calls the first and `resolve_mesoscope_assembly_sources` in `assembly_sources.py` calls the
second, registered in `_ASSEMBLY_GEOMETRY_REGISTRY` and `_ASSEMBLY_SOURCE_REGISTRY`.

---

## Constants and the canonical bodyparts

Every name, threshold, and filename fragment the tracking pass uses is a module-level constant in
`mesoscope_vr/video_tracking.py`.

| Constant                     | Value                                                | Role                                                                             |
|------------------------------|------------------------------------------------------|----------------------------------------------------------------------------------|
| `_EYE_TRACKING_PROJECT_NAME` | `"eye_tracking"`                                     | The DeepLabCut project name baked into the prediction filename                   |
| `PUPIL_CAMERA_NAME`          | `"face_camera"`                                      | Public. Names the output feather and prefixes the face camera's dataset columns  |
| `_PUPIL_TARGET`              | `"pupil"`                                            | The tracking-target label in `{camera}_{target}.feather`                         |
| `_REFLECTION_POINT`          | `"reflection"`                                       | The corneal infrared reflection, a motion-robust positional reference            |
| `_PUPIL_POINTS`              | eight pupil-perimeter bodyparts, in ring order       | The pupil ellipse's ring                                                         |
| `_EYE_POINTS`                | `("eye_right", "eye_bottom", "eye_left", "eye_top")` | The eye ellipse's ring, in the same order convention                             |
| `_CANONICAL_POINTS`          | `(_REFLECTION_POINT, *_PUPIL_POINTS, *_EYE_POINTS)`  | The thirteen bodyparts the `.h5` must supply                                     |
| `_LIKELIHOOD_THRESHOLD`      | `0.8`                                                | The minimum DeepLabCut likelihood for a point to be trusted                      |
| `_MINIMUM_PERIMETER_POINTS`  | `3`                                                  | The ring points a feature needs for its ellipse to be determined                 |
| `_MAXIMUM_FIT_CONDITION`     | `12.0`                                               | The largest admissible design-matrix condition number                            |
| `_BLINK_FRACTION`            | `0.5`                                                | The fraction of median openness below which a frame is a blink                   |
| `_MINIMUM_FIT_EXTENT_PX`     | `1e-6`                                               | The smallest fitted ellipse extent a division may treat as a measurement         |
| `_COORDINATES`               | `("x", "y", "likelihood")`                           | The three per-bodypart channels, in read order                                   |
| `_MINIMUM_COLUMN_LEVELS`     | `2`                                                  | The column-index levels a frame must carry to be read as a DeepLabCut prediction |

`_PUPIL_POINTS` is `("pupil_right", "pupil_bottom_right", "pupil_bottom", "pupil_bottom_left", "pupil_left",
"pupil_top_left", "pupil_top", "pupil_top_right")`. The order is load-bearing. It starts at the right and advances
toward the bottom in image coordinates, where y increases downward, and it assigns each point its evenly spaced
parametric angle on the ellipse. Reordering the tuple silently re-labels the ring and corrupts every fit, so a new
bodypart is appended to the ring at its geometric position or not at all.

`_LIKELIHOOD_THRESHOLD` sits inside the field's usual 0.6 to 0.9 `pcutoff` band. It stays high because the
labeled-angle fit trusts each surviving point's ring identity, so a mislabeled low-confidence point biases the
ellipse directly rather than averaging out. It stays off the top of the band because the eye ring carries only four
points against the three-point minimum, and rejecting one borderline point costs the whole eye fit.

---

## Locating the prediction file

`locate_mesoscope_pose_predictions(session)` returns `None` when `session.raw_data.camera_data_path` is not a
directory. Otherwise it globs that directory for `*eye_tracking*.h5`, sorts the matches with `natsort.natsorted`,
and returns the first, or `None` when nothing matches. A re-run of inference leaves several files matching, and the
natural-sort-first one is the file the tracking stage opens.

`process_mesoscope_video_tracking(session, output_directory)` calls the locator first. When it returns `None`, the
pass echoes at `LogLevel.INFO` and returns without writing anything, because the stage is optional and gated
entirely on the prediction file being present:

```text
No DeepLabCut 'eye_tracking' '.h5' prediction file was found beside the face-camera video in the raw camera_data
directory of session '{session.session_name}'. Skipping pupil tracking.
```

A session without predictions therefore completes the tracking job successfully with no output. Treat a missing
`face_camera_pupil.feather` as the expected result of that path, not as a failure to investigate.

---

## Reading the predictions

The predictions are produced upstream rather than in-process because DeepLabCut pins `numpy<2` while
sollertia-forgery runs `numpy>=2` on Python 3.14, so DeepLabCut cannot be imported here. This worker only reads
DeepLabCut's `.h5` output, which keeps the `deeplabcut` library out of the forgery environment entirely.

`_read_points_from_h5` reads the file with `pandas.read_hdf`, since DeepLabCut serializes with `pandas.DataFrame.to_hdf`
in the PyTables `table` format, then sorts by index so rows are in ascending frame order. Columns carry a
`(scorer, bodypart, coordinate)` MultiIndex, and each column is keyed by the trailing `(bodypart, coordinate)` pair
alone, so the single scorer level never has to be named. Each bodypart becomes one `(frame_count, 3)` array of
`(x, y, likelihood)`.

Two `ValueError` messages come out of this function, both raised through `console.error`:

```text
Unable to read pupil tracking from '{h5_path.name}'. The file does not contain a DeepLabCut prediction frame.

Unable to read pupil tracking from '{h5_path.name}'. The DeepLabCut prediction file does not contain the required
'{bodypart}' '{coordinate}' column.
```

The second is the one a mismatched DeepLabCut project produces. It names the first missing bodypart and channel, so
read it as a roster disagreement between the trained model and `_CANONICAL_POINTS`, not as a corrupt file.

---

## The ring-ellipse fit

`_fit_ring_ellipse(points, names)` fits one centrally symmetric ellipse per frame. A point labeled `t` lies at
`P(t) = center + cos(t) * semi_a + sin(t) * semi_b`, where `semi_a` and `semi_b` are the ellipse's conjugate
semi-diameters. That model is linear in the unknowns and separates by axis, so both coordinates share one
`[1, cos(t), sin(t)]` design matrix, and three confident points already determine the fit wherever on the ring they
sit. Angles are `2 * pi * i / len(names)`, evenly spaced and starting at zero.

Frames that lost the same points share a design matrix, and therefore share a conditioning verdict. Each frame's
confidence mask is packed into one integer code, `confident.astype(np.int64) @ (1 << np.arange(len(names)))`, and
`np.unique` groups the codes with a one-dimensional pass rather than the void-row lexsort that `np.unique(..., axis=0)`
would run over the whole mask. Each distinct occlusion pattern is solved once for every frame carrying it.

Per pattern, in order:

1. Skip the pattern when fewer than `_MINIMUM_PERIMETER_POINTS` points cleared the likelihood gate. Three points
   supply six equations for the fit's six unknowns, so fewer leaves the ellipse undetermined.
2. Build the `(kept, 3)` design matrix over the surviving angles and take its condition number, which measures how
   far the surviving arc must reach to pin the rest of the ellipse.
3. Reject the pattern when `not condition_number <= _MAXIMUM_FIT_CONDITION`. The comparison is negated so that a
   non-finite condition number rejects the pattern, since every comparison against NaN is False.
4. Solve every frame in the pattern at once with `np.linalg.lstsq`, then write `center`, `semi_a`, `semi_b`, the
   shared condition number, and `valid = True` into those rows.
5. Record the RMS point-to-ellipse residual only when `kept.size > design.shape[1]`, that is only when the ring is
   overdetermined. Exactly-determined three-point fits have a structurally zero residual that says nothing about fit
   quality, so they keep NaN and are judged on the condition number alone.

Rejected frames keep NaN geometry and `valid = False`. The cap of `12.0` sits at the worst value a minimally determined
three-point fit on an evenly spaced ring produces. It therefore admits every arc the pupil and eye rings can present and
guards against a ring that samples the ellipse less evenly.

---

## Derived geometry, blinks, and the frame partition

`_compute_pupil_metrics` fits the pupil ring and the eye ring, then derives three geometric quantities:

| Quantity            | Formula                                                                         | Note                                                                        |
|---------------------|---------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| `pupil_area_px2`    | `pi * abs(semi_a_x * semi_b_y - semi_a_y * semi_b_x)`                           | Pi times the cross-product magnitude of the two conjugate semi-diameters    |
| `pupil_diameter_px` | `hypot(semi_a) + hypot(semi_b)`                                                 | The mean of the two full-axis diameters, `(2 * a + 2 * b) / 2`              |
| `eye_openness`      | `eye_height / eye_width`, where `eye_width >= _MINIMUM_FIT_EXTENT_PX`, else NaN | `eye_width` and `eye_height` are twice the eye ring's two semi-axis lengths |

`eye_openness` is an aspect ratio, so it is invariant to camera distance and comparable across animals and sessions.
A zero-width eye is a degenerate fit rather than a closed eye, so it yields NaN instead of dividing.

The blink baseline is the session median openness over the frames the eye ring actually fitted,
`np.nanmedian(eye_openness[eye_fit.valid])`, and it is NaN when no such frame is finite. With no baseline the
openness term drops out of the rule below and the eye's visibility carries the flag alone.

```text
eye_evidence_lost = ~(eye_fit.valid & reflection_valid) | ~isfinite(eye_openness)
is_blink          = (eye_evidence_lost & ~pupil_fit.valid) | (eye_openness < _BLINK_FRACTION * baseline)
pupil_measured    = pupil_fit.valid & ~is_blink
is_dilated        = ~is_blink & ~pupil_measured
```

A blink is read from the eye and its cornea, never from a missing pupil. Something covering the eye takes the eye
ring, the corneal reflection, and the opening down together, and a lid and a paw read the same. A pupil that
resolves is positive evidence of an open eye, since a covered eye presents no pupil ring to fit, so it overrides the
two absence terms. The measured-openness term stands on its own, because a fitted eye observed to be closing is an
observation rather than a gap.

A pupil that vanishes under an open eye stays out of the blink rule deliberately. It has outgrown the palpebral
opening rather than been hidden by a lid, and folding it in would delete the most dilated pupils from the arousal
signal exactly when arousal is highest. `is_dilated` catches those frames instead, so the two flags partition every
unmeasured frame between them. A frame is measured, dilated, or blinked, never two of the three.

---

## The pupil feather

`PupilColumn` declares the nineteen columns of `face_camera_pupil.feather`, and the assembled metric dictionary uses
the same order, so declaration order is feather column order. Every `Float64` column is cast to `Float32` because the
geometry derives from pixel coordinates far coarser than `Float32` resolves. The two flags are written as `Boolean`
and are the only non-float columns.

| Column                         | Dtype   | NaN wherever                                                                                        |
|--------------------------------|---------|-----------------------------------------------------------------------------------------------------|
| `pupil_center_x_px`            | Float32 | `pupil_measured` is False                                                                           |
| `pupil_center_y_px`            | Float32 | `pupil_measured` is False                                                                           |
| `pupil_diameter_px`            | Float32 | `pupil_measured` is False                                                                           |
| `pupil_area_px2`               | Float32 | `pupil_measured` is False                                                                           |
| `pupil_fit_condition`          | Float32 | `pupil_measured` is False                                                                           |
| `pupil_fit_residual_px`        | Float32 | `pupil_measured` is False, and on exactly-determined three-point fits                               |
| `eye_center_x_px`              | Float32 | `eye_fit.valid` is False                                                                            |
| `eye_center_y_px`              | Float32 | `eye_fit.valid` is False                                                                            |
| `eye_width_px`                 | Float32 | `eye_fit.valid` is False                                                                            |
| `eye_height_px`                | Float32 | `eye_fit.valid` is False                                                                            |
| `eye_openness`                 | Float32 | `eye_fit.valid` is False                                                                            |
| `blinking_state`               | Boolean | never, the flag is total                                                                            |
| `dilation_state`               | Boolean | never, the flag is total                                                                            |
| `reflection_x_px`              | Float32 | `reflection_valid` is False                                                                         |
| `reflection_y_px`              | Float32 | `reflection_valid` is False                                                                         |
| `pupil_reflection_offset_x_px` | Float32 | `pupil_measured & reflection_valid` is False                                                        |
| `pupil_reflection_offset_y_px` | Float32 | `pupil_measured & reflection_valid` is False                                                        |
| `pupil_in_eye_x`               | Float32 | `pupil_measured & eye_fit.valid` is False, or the eye semi-width is below `_MINIMUM_FIT_EXTENT_PX`  |
| `pupil_in_eye_y`               | Float32 | `pupil_measured & eye_fit.valid` is False, or the eye semi-height is below `_MINIMUM_FIT_EXTENT_PX` |

Positions and lengths are in the face camera's pixel coordinate frame, marked by the `_px` suffix, and are never
converted to physical units, because the camera is not calibrated against a physical scale. The two offset pairs are
the eye-position signals. Referencing the corneal reflection cancels the eye's common-mode motion relative to the
camera, and `pupil_in_eye_*` normalizes the pupil's offset from the eye center by the eye's semi-axes, which makes
it dimensionless and comparable across animals.

The eye columns answer only to their own fit, so they stay populated on a blink flagged by low openness or by a lost
reflection. The pupil columns answer to the blink as well.

The pass writes `f"{PUPIL_CAMERA_NAME}_{_PUPIL_TARGET}.feather"`, that is `face_camera_pupil.feather`, into the
`output_directory` the job supplies, which is `session.processed_data.video_data_path`. It is written with
`write_ipc(compression="uncompressed")` and echoed at `LogLevel.SUCCESS` as
`Wrote pupil tracking for {frame_count} frame(s) to '{output_path.name}'.`

The feather is a positional table. It holds one row per frame in acquisition order and stores only the per-frame
metrics, so row position supplies the frame index and none is stored. Timestamps are left to dataset assembly, which
owns every stream's alignment to the acquisition clock. Never join this feather to anything by an index column,
because it has none.

---

## The video sub-dataset

`assemble_video_dataset(video_data_path, reference_time)` in `mesoscope_vr/video_dataset.py` reads the fixed camera
set from the processed video-data directory and interpolates every present feather onto the reference time vector.
`_CAMERA_SOURCES` is a fixed two-entry tuple of frozen `_CameraSource` records, and only the face camera carries the
eye, so only it names a pupil file.

| `name`        | `timestamps_file`                | `energy_file`                | `pupil_file`                |
|---------------|----------------------------------|------------------------------|-----------------------------|
| `face_camera` | `face_camera_timestamps.feather` | `face_camera_energy.feather` | `face_camera_pupil.feather` |
| `body_camera` | `body_camera_timestamps.feather` | `body_camera_energy.feather` | `None`                      |

Each camera's timestamp feather is its source clock, read from the `ExtractedDataColumns.FRAME_TIME` column that
`ataraxis_video_system` defines. A camera whose timestamp feather is absent is skipped entirely, and its energy and
pupil feathers are never read. Every other feather for that camera holds one row per recorded frame in the same
acquisition order, so its values align to the clock by row position.

Output column naming differs between the two feather kinds. Energy columns are emitted as
`f"{camera.name}_{source_column}"` over `_MOTION_ENERGY_COLUMN` and `_FRAME_LUMINANCE_COLUMN`, yielding
`face_camera_motion_energy`, `face_camera_frame_luminance`, `body_camera_motion_energy`, and
`body_camera_frame_luminance`. Pupil columns are iterated verbatim off `pupil_frame.columns`, so they reach the
dataset under their bare `PupilColumn` names with no camera prefix.

Interpolation splits two ways, by whether the value has a meaningful linear blend:

| Source values                                  | Mode          | Call                | Stored dtype |
|------------------------------------------------|---------------|---------------------|--------------|
| Motion energy, frame luminance, pupil geometry | Linear        | `is_discrete=False` | `float32`    |
| `blinking_state`, `dilation_state`             | Nearest prior | `is_discrete=True`  | `uint8`      |

The two flags are the members of `_PUPIL_FLAG_COLUMNS`, and they take the nearest prior frame's value because a
boolean state cannot be linearly blended. Linear interpolation propagates a not-a-number source value to the samples
that bracket it, so an unmeasured frame stays unmeasured and the NaN masks of the pupil feather survive into the
forged dataset rather than being smoothed away.

`_read_frame_aligned` checks every energy and pupil feather against its camera's timestamp row count and raises
`ValueError` on any disagreement:

```text
Unable to assemble the video dataset. The feather '{feather_path.name}' has {frame.height} rows, but the camera
timestamp feather '{timestamps_filename}' has {expected_rows}. Every per-camera video feather must hold exactly one
row per recorded frame.
```

The assembler returns an empty `pl.DataFrame()` when `video_data_path` is not a directory, and again when no camera
contributed a single column. A session processed without video therefore still forges, contributing no video columns
rather than failing assembly.

---

## The slowest-camera reference clock

`resolve_slowest_camera_clock(video_data_path)` supplies the assembly reference clock for training sessions.
Experiment sessions use the mesoscope fluorescence clock instead, so this resolver is reached only from
`assemble_training_dataset`.

For each camera source with a present timestamp feather, it requires at least `_MINIMUM_CLOCK_FRAMES`, which is two, and
computes `duration_seconds` from the first and last timestamps. Both endpoints are pulled one at a time through a
pushed-down one-row slice and arrive as Python integers, whose difference cannot wrap the way the unsigned timestamp
column's would. An out-of-order feather therefore states a negative span and is dropped rather than read as the slowest
clock. A non-positive duration disqualifies the camera. The mean rate is the frame count divided by that duration, and
the camera with the lowest mean rate wins, its timestamps returned verbatim. The comparison is strict, so an exact tie
is settled in favour of the first camera `_CAMERA_SOURCES` names, which is the face camera.

The slowest camera is chosen because every other data source can be interpolated onto its coarser grid without inventing
samples between its frames. On success the resolver echoes
`Resolved the '{selection.camera}' clock ({selection.mean_rate:.2f} fps) as the reference clock.` When no camera
qualifies it raises `FileNotFoundError`:

```text
Unable to resolve the reference clock for the training session. No camera timestamp feather with at least two frames
spanning a positive duration was found under '{video_data_path}', so no camera clock can serve as the assembly
reference clock.
```

---

## Related skills

The `video:log-processing` entry below resolves through the ataraxis marketplace. Every other entry resolves inside
the sollertia marketplace.

| Skill                                  | Relationship                                                                                |
|----------------------------------------|---------------------------------------------------------------------------------------------|
| `experiment:external-tool-bindings`    | Owns the binding convention behind the externally produced prediction file                  |
| `/mesoscope-vr`                        | Producer: owns the acquisition-side `video_tracking` section and the `slvt infer` call      |
| `/mesoscope-vr-processing-schema`      | Owns the `VideoDataFiles` filename roster and the `DatasetColumn` rows these columns become |
| `/mesoscope-vr-dataset-assembly`       | Downstream: calls the assembly-side `video_dataset.py` pair and chooses the reference clock |
| `/mesoscope-vr-fluorescence-alignment` | Owns the mesoscope fluorescence clock used as the experiment-session reference time         |
| `forging:batch-processing`             | Owns the video pipeline jobs, including the tracking job that invokes this donation         |
| `forging:processing-results`           | Owns the processed-data layout in which these feathers are located                          |
| `forging:processing-input-format`      | Owns the raw-data prerequisites, including the camera_data directory this pass reads        |
| `forging:data-processing-design`       | Owns the registry model and the donation protocols these two seams satisfy                  |
| `forging:library-extension`            | Reference: how a new acquisition system donates its own locator and tracking pass           |
| `video:log-processing`                 | Upstream: owns the camera log archives behind the per-camera timestamp feathers             |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Naming and ownership:
- [ ] Every use of the token video_tracking states which side it means, acquisition or forgery
- [ ] The acquisition-side configuration section was routed to /mesoscope-vr rather than described here
- [ ] Both registry seams are named, _POSE_PREDICTION_REGISTRY and _VIDEO_TRACKING_REGISTRY

Tracking pass:
- [ ] Bodypart names, thresholds, and filenames were read from the video_tracking.py constants, never hardcoded
- [ ] _PUPIL_POINTS and _EYE_POINTS order was preserved, since ring order assigns each point its parametric angle
- [ ] A missing prediction file was read as the optional stage's INFO skip, not as a job failure
- [ ] A NaN geometry value was traced to its per-column mask before being called a defect
- [ ] A NaN pupil_fit_residual_px was checked against the exactly-determined three-point case first
- [ ] Any frame classification used the measured, dilated, or blinked partition rather than a two-way split
- [ ] The pupil feather was treated as positional, with row order supplying the frame index

Video sub-dataset:
- [ ] Energy columns were named camera_source and pupil columns were left unprefixed
- [ ] The two state flags were treated as nearest-prior uint8 and every other value as linear float32
- [ ] A row-count mismatch was reported against the named timestamp feather rather than silently trimmed
- [ ] An empty video dataset was read as a session processed without video, not as an assembly failure
- [ ] The slowest-camera clock was cited only for training sessions
```
