---
name: mesoscope-vr-fluorescence-alignment
description: >-
  Documents Mesoscope-VR's concrete fluorescence sub-assembly: the primary TTL duration-window path, the stray
  pulse-run discard that drops hand-triggered scanning, the reconciliation against cindra's frame count, and the
  ScanImage metadata fallback with its three metadata keys, anchor search, and match tolerance. Use when
  interpreting fluorescence frame alignment, debugging dropped or excess TTL pulses, or changing the alignment
  tolerances.
user-invocable: false
---

# Mesoscope-VR fluorescence alignment

Documents how the Mesoscope-VR dataset-forging pipeline aligns two-photon fluorescence frames to the
microcontroller-logged frame-acquisition TTL pulses, building the frame-indexed reference clock onto which the
experiment session assembly interpolates every other stream.

This is the concrete Mesoscope-VR instance of the agnostic processing-stage seam owned by the forging plugin. The
alignment consumes upstream cindra single-recording and multi-recording fluorescence outputs and the microcontroller
mesoscope-frame TTL feather, and produces the fluorescence sub-dataset that becomes the spine of the assembled
session `data.feather`.

---

## Scope

**Covers:**
- The primary TTL alignment path: pairing rising and falling TTL edges and keeping pulses whose duration falls inside
  the expected scan-pulse window (`expected_duration_ms` plus or minus 20 ms)
- The stray pulse-run discard that drops runs of TTL pulses which imaged no frame, before any count comparison
- The reconciliation of the surviving pulse count against cindra's frame count, and the front-clip rule with the
  single-plane assumption it rests on
- The ScanImage metadata fallback path invoked when the surviving pulses fall short of the frame count
- The three ScanImage metadata keys, the lexicographic chronological ordering, the anchor search, and the match
  tolerance
- The output index columns (`frame`, `time_us`, `elapsed_minutes`) and cell masking via the classification first
  column
- The eight single-day and multi-day fluorescence sources, their `RecordingArrays` members, and their emitted
  `DatasetColumn` names
- The `time_us` reference clock produced here and consumed as `reference_time` by the sibling sub-assemblies

**Does not cover:**
- cindra single-recording and multi-recording fluorescence production. Owned by `forging:processing-input-format`.
- Stacking the sub-datasets into the session `data.feather`, and the training-session reference clock. Owned by
  `/mesoscope-vr-dataset-assembly`.
- The forged column roster and the per-column descriptions. Owned by `/mesoscope-vr-processing-schema`.
- The agnostic forged-output layout on disk. Owned by `forging:processing-results`.
- Batch orchestration of the forging pipeline. Owned by `forging:dataset-forging` and `forging:data-processing-design`.
- Mesoscope-frame TTL feather production. Owned by `/mesoscope-vr-module-parsing`.

---

## Fluorescence stage contract

The forging plugin owns the agnostic processing-stage doctrine: feather-as-interchange between stages, the
prepare-then-execute batch model, and the worker-budget concurrency contract (see `forging:data-processing-design`).
This skill documents only the Mesoscope-VR-specific content of the fluorescence sub-assembly, the alignment algorithm
and its constants, not the orchestration that schedules it.

`assemble_cindra_dataset` in `mesoscope_vr/two_photon_dataset.py` is a sub-assembler rather than the stage entry
point. The stage entry point is `assemble_mesoscope_session` in `mesoscope_vr/forging.py`, which dispatches an
experiment session to `assemble_experiment_dataset` in `mesoscope_vr/experiment_dataset.py`. That entry point reaches
the agnostic pipeline through the `_FORGING_ASSEMBLY_REGISTRY` seam in `sollertia_forgery.registries`, read by
`resolve_forging_assembly_worker`, and the donation filling that seam is owned by
`/mesoscope-vr-dataset-assembly`. `assemble_experiment_dataset` runs this sub-assembly first and takes
`fluorescence_data["time_us"].to_numpy()` as the `reference_time` handed to the behavior, runtime, and video
sub-assemblies.

Training sessions carry no fluorescence sub-dataset at all. `assemble_training_dataset` resolves its reference clock
from `resolve_slowest_camera_clock` in `mesoscope_vr/video_dataset.py` instead.

`assemble_cindra_dataset` takes four directory paths and returns a single Polars DataFrame:

| Parameter                   | Supplies                                                                                                                       |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| `cindra_data_path`          | The single-recording cindra output directory holding the combined metadata, the cell classification, and the four trace arrays |
| `microcontroller_data_path` | The processed microcontroller-data directory holding the mesoscope-frame TTL feather                                           |
| `multiday_data_path`        | The multi-recording cindra output directory holding the same four trace arrays                                                 |
| `raw_data_path`             | The session `raw_data` directory. Holds the ScanImage per-frame metadata archive both later stages read                        |

`microcontroller_data_path` is the session's `processed_data/microcontroller_data` directory, the tree into which the
module parsers write their feathers. The filename roster that directory follows is owned by
`/mesoscope-vr-processing-schema`.

`multiday_data_path` is resolved by the caller rather than here. `assemble_experiment_dataset` calls cindra's
`resolve_dataset_path` with `output_root=session.processed_data_path` and the name that
`multi_recording_dataset_name(animal_id=..., dataset_name=...)` builds, which is `{animal_id}_{dataset_name}`. The
animal identifier qualifies the name so an animal's multi-recording output stays separate when a dataset spans
several animals, and cindra's own resolver locates the directory so this module never respells cindra's layout.

The authoritative target frame count comes from the single-recording `RecordingArrays.CELL_FLUORESCENCE` array, located
by `resolve_array_path`. The function reads its shape through a memory-mapped header read
(`np.load(..., mmap_mode="r").shape`) and unpacks the second axis as the frame count, because the cindra array is
`(rois, frames)`. Every alignment path must produce exactly that many rows.

---

## Primary TTL duration-window alignment

The mesoscope frame TTL feather (`BehaviorDataFiles.MESOSCOPE_FRAME` = `mesoscope_frame_data.feather`, produced by
the TTL module parser, see `/mesoscope-vr-module-parsing`) holds `time_us` and `ttl_state` columns, one row
per logged TTL transition. The stage reads it memory-mapped, sorts by `time_us`, and reconstructs whole pulses:

1. Compute `ttl_diff` as the first difference of `ttl_state` (null-filled with 0). A value of `1` marks a rising
   edge, `-1` a falling edge.
2. Assign a `pulse_id` as the cumulative sum of rising edges.
3. Filter rising edges into `(pulse_id, pulse_start)` and falling edges into `(pulse_id, pulse_end)`, then inner-
   join on `pulse_id`. The inner join drops any pulse missing either edge.
4. Compute `duration_ms` as `(pulse_end - pulse_start)` divided by 1000, and sort by `pulse_start`. This is the
   `paired_pulses` table, retained unfiltered because the ScanImage fallback consumes it.

The expected pulse duration derives from cindra's per-plane sampling rate. cindra's own reader supplies the rate,
loading the combined metadata and no array, so the archive's layout is stated once by the library that writes it.
The rate converts to a millisecond interval through `ataraxis_time`:

```python
scanning_frequency = CombinedData.load(root_path=cindra_data_path).sampling_rate
expected_duration_ms = rate_to_interval(rate=scanning_frequency, to_units=TimeUnits.MILLISECOND, as_float=True)
minimum_duration = expected_duration_ms - _SCAN_PULSE_TOLERANCE_MS
maximum_duration = expected_duration_ms + _SCAN_PULSE_TOLERANCE_MS
```

The primary path keeps only pulses whose `duration_ms` is between `minimum_duration` and `maximum_duration`
inclusive, renames `pulse_id` to `frame` and `pulse_start` to `time_us`, and sorts by `frame`. The duration window
is a fixed `_SCAN_PULSE_TOLERANCE_MS = 20` ms on either side of the expected duration.

---

## Stray pulse-run discard

`_discard_unacquired_pulse_runs` runs on the duration-filtered pulses, after the primary filter and before the count
reconciliation. Triggering the mesoscope by hand outside the acquisition emits a run of scan pulses that ScanImage
never saves, so the log holds more pulses than there are frames. Splitting the log at its idle gaps and keeping the
run grouping whose pulse counts best account for the known acquisition sizes separates the stray runs from the
acquisitions. The pulse counts carry that distinction even when a stray run and an acquisition are of a similar
length.

| Constant                            | Value                | Meaning                                                                                                                                                                                            |
|-------------------------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `_PULSE_RUN_GAP_FACTOR`             | `3.0`                | The multiple of the median pulse period above which an interval starts a new run. Splitting one acquisition is harmless, since the matching rejoins the pieces, so the split is deliberately eager |
| `_MINIMUM_SPLITTABLE_PULSE_COUNT`   | `3`                  | The fewest pulses the run splitting accepts. A shorter log carries too few inter-pulse periods for a median period to separate an idle gap from the scan cadence                                   |
| `_UNMATCHED_COST`                   | `sys.maxsize`        | The sentinel cost standing for a run-span assignment that leaves at least one acquisition unplaced                                                                                                 |
| `_SCANIMAGE_ACQUISITION_NUMBER_KEY` | `acquisitionNumbers` | The per-frame ScanImage acquisition index inside the metadata archive                                                                                                                              |

The pass returns its input unchanged at every step that cannot decide, so an ordinary session pays only the first
two checks:

1. Return unchanged when fewer than `_MINIMUM_SPLITTABLE_PULSE_COUNT` pulses survived the duration filter.
2. Take the first differences of `time_us` as the pulse periods and split at
   `np.flatnonzero(periods > np.median(periods) * _PULSE_RUN_GAP_FACTOR)`. Return unchanged when there are no
   breaks, which is the overwhelming majority single-run case.
3. Resolve the per-acquisition frame counts through `_resolve_acquisition_sizes`, in descending order. Return
   unchanged when the list is empty.
4. Assign each acquisition a consecutive run span through `_match_runs_to_acquisitions`. Return unchanged when it
   returns `None`.
5. Build a boolean keep mask from the claimed spans. Return unchanged when it keeps every pulse. Otherwise echo the
   warning below and return the filtered pulses.

`_resolve_acquisition_sizes` reads `raw_data/mesoscope_data/frame_variant_metadata.npz` and returns an empty list
when the archive is absent. It has two paths. Sessions preprocessed with acquisition-aware frame numbering carry an
explicit `acquisitionNumbers` array, read directly and used only when `np.unique(acquisitions).size > 1`. Older
sessions number every acquisition from one, so a frame number appearing in the archive N times was produced by N
separate acquisitions. Counting how many `frameNumberAcquisition` values survive each successive peel of that
multiset recovers the same sizes, for any row order the archive happens to carry.

`_match_runs_to_acquisitions` returns `None` when there are no acquisitions, or when there are more acquisitions than
runs. Otherwise it brute-forces every `permutations(range(acquisition_count))`, delegating each ordering to the
`_search_run_spans` branch-and-bound recursion, and minimizes the summed `abs(total - target)` count difference. An
acquisition claims a consecutive span of runs rather than one run, because a brief hiccup can split one acquisition
across several runs. The match is approximate rather than exact because an acquisition emits a small surplus from
scanner arming minus the pulses the duration filter rejected. `_search_run_spans` abandons any branch whose accumulated
cost already matches the cheapest complete assignment, and stops extending a span once its total passes the target. The
search returns `None` when the cheapest assignment still carries `_UNMATCHED_COST`.

The discard is never silent. It echoes at `LogLevel.WARNING`, filling the discarded count, the total pulse count,
and `raw_data_path.parent.name` into this message:

```text
Discarded {discarded} of {total} mesoscope TTL pulses that imaged no frame while assembling the cindra dataset for
'{session}'. The mesoscope was likely triggered by hand outside the acquisition.
```

---

## Frame count reconciliation

With the stray runs gone, the surviving row count is reconciled against the cindra frame count:

| Condition                      | Action                                                                   |
|--------------------------------|--------------------------------------------------------------------------|
| Surviving pulses `>` `frames`  | Clip the front: keep the last `frames` rows (`.tail(frames)`)            |
| Surviving pulses `==` `frames` | Use the surviving pulses directly                                        |
| Surviving pulses `<` `frames`  | Fall back to ScanImage-metadata alignment (`_align_pulses_to_scanimage`) |

Clipping the front assumes the surplus sits before the acquisition, which holds while every plane contributes the
same sample count. The reference Mesoscope-VR recording guarantees that by acquiring one physical plane on one
channel. cindra's plane combination trims each plane to the shortest one it holds, so a recording that interleaves
several planes or two channels would carry part of its surplus at the tail instead.

---

## ScanImage metadata fallback path

The surviving count falls below cindra's frame count when the duration filter has rejected real frames whose TTL
signal briefly fell outside the tolerance window. The stage then delegates to `_align_pulses_to_scanimage`, which
consumes the unfiltered `paired_pulses` table rather than the filtered one. ScanImage writes one entry per acquired
TIFF to `frame_variant_metadata.npz`, making its per-frame timestamps the authoritative record of which TTL rising
edges correspond to real frames. Pulses matching no ScanImage frame within tolerance are dropped as noise, typically
the electrical glitches captured while the mesoscope arms.

The fallback resolves the metadata archive at `raw_data_path / mesoscope_data / frame_variant_metadata.npz`
(`MesoscopeDirectories.MESOSCOPE_DATA` is `mesoscope_data`, and the filename constant is
`_FRAME_VARIANT_METADATA_FILENAME`). It raises `ValueError` through `console.error` when the archive is missing,
when the archive entry count does not equal the cindra frame count, or when the final matching does not produce
exactly `expected_frame_count` rows.

Three ScanImage metadata keys are read from the archive:

| Constant                            | Key value                | Meaning                                                                                                                                         |
|-------------------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| `_SCANIMAGE_FRAME_NUMBER_KEY`       | `frameNumberAcquisition` | Per-frame counter. It increases with acquisition time within one acquisition and restarts at one for each further acquisition a session records |
| `_SCANIMAGE_FRAME_TIMESTAMP_KEY`    | `frameTimestamps_sec`    | Per-frame ScanImage clock timestamps in seconds                                                                                                 |
| `_SCANIMAGE_ACQUISITION_NUMBER_KEY` | `acquisitionNumbers`     | Per-frame acquisition index. Substituted with `np.zeros_like(frame_numbers)` when the archive omits it                                          |

The archive is stored in TIFF-page-concatenation order, which interleaves frames across stack files, so the fallback
restores chronological order with `np.lexsort((frame_numbers, acquisitions))`. The acquisition number is the primary
key and the frame counter the secondary key, precisely because the counter restarts at one per acquisition. The
sorted `frameTimestamps_sec` values then convert to microseconds as `int64`. NPZ archives do not support memory
mapping, so a context manager keeps the archive open only long enough to copy the arrays out.

The entry-count equality is a one-to-one check between ScanImage entries and cindra frames. It holds while the
recording delivers one cindra sample per ScanImage frame, which the reference Mesoscope-VR configuration guarantees
by acquiring one physical plane on one channel. cindra sizes each plane from its own interleave position, so a
recording carrying several planes or two channels reports a per-position count this comparison would read as an
inconsistency.

---

## Clock-offset estimation and anchor search

ScanImage and the microcontroller run on separate clocks. The fallback estimates a single constant offset by
anchoring a candidate TTL pulse to the first ScanImage frame, then choosing the offset that yields the most matches:

1. For each candidate anchor (the first `_SCANIMAGE_ANCHOR_SEARCH_LIMIT = 10` pulses, or fewer if the log is
   shorter), compute `candidate_offset = pulse_microseconds[anchor] - scanimage_microseconds[0]`.
2. Shift the ScanImage timestamps by the candidate offset and count how many TTL rising edges have a ScanImage frame
   strictly less than `_SCANIMAGE_MATCH_TOLERANCE_US` away.
3. Keep the offset with the highest match count.

The search ranges over the leading pulses because the first few TTL pulses may be electrical glitches captured while
the mesoscope arms. With the best offset applied, each pulse is matched to its nearest aligned ScanImage frame by
`_nearest_target_index`, a `searchsorted`-based nearest-target lookup whose `sorted_targets` argument must hold at
least two elements. Conflicts where several pulses claim the same frame are resolved by keeping the pulse closer to
that frame's expected time. Pulses not strictly closer than `_SCANIMAGE_MATCH_TOLERANCE_US` to any frame are dropped.

| Constant                         | Value    | Rationale                                                                             |
|----------------------------------|----------|---------------------------------------------------------------------------------------|
| `_SCANIMAGE_MATCH_TOLERANCE_US`  | `50_000` | The tolerance applied when matching TTL rising edges to ScanImage frame timestamps    |
| `_SCANIMAGE_ANCHOR_SEARCH_LIMIT` | `10`     | The maximum number of leading TTL pulses considered as candidate clock-offset anchors |

The fallback returns a DataFrame with `frame` (the matched pulse's original `pulse_id`) and `time_us` (the
rising-edge microsecond timestamp), sorted by `frame`, containing exactly `expected_frame_count` rows. `time_us` is
cast back to `np.uint64`, the width the primary path emits, so the forged feather's timestamp column carries one
dtype whichever path aligned the session.

---

## Output columns and cell masking

Regardless of which path produced the frame-aligned table, the stage finalizes a frame index and a within-session
time column:

| Column            | Type      | Derivation                                                               |
|-------------------|-----------|--------------------------------------------------------------------------|
| `frame`           | `UInt32`  | Re-assigned as a contiguous 1-based range over the surviving rows        |
| `time_us`         | `UInt64`  | The TTL rising-edge timestamp, carried from the alignment path           |
| `elapsed_minutes` | `Float32` | `(time_us - time_us.min())` divided by 60,000,000, rounded to 2 decimals |

These three match the `DatasetColumn` members `FRAME`, `TIME_US`, and `ELAPSED_MINUTES`. `time_us` is the frame-
aligned reference clock consumed by the sibling sub-assemblies as `reference_time`, and `elapsed_minutes` is a
derived within-session display column. See `/mesoscope-vr-dataset-assembly`.

Cell masking uses the single-recording cell classification, loaded through
`resolve_array_path(root_path=cindra_data_path, array=RecordingArrays.CELL_CLASSIFICATION)`. cindra stores it as a
`(num_rois, 2)` float32 array where column 0 holds the is-cell label (`1.0` or `0.0`) and column 1 the classifier
probability. The mask keeps ROIs where `classification[:, 0] == 1`, and it is applied to the single-day sources
alone. The multi-day sources are added unmasked.

---

## Single-day versus multi-day sources

Eight fluorescence Series are streamed onto the frame-aligned table one file at a time, so peak memory holds a
single fluorescence array rather than eight. Each source is memory-mapped through `resolve_array_path`, masked when
a mask applies, transposed to `(frames, rois)`, and cast to float32. The source array is named by a `RecordingArrays`
member and the emitted column by a `DatasetColumn` member, so neither the cindra filenames nor the column strings
are spelled in this module:

| Source directory     | `RecordingArrays` member  | `DatasetColumn` member               | Emitted column                       | Cell-masked |
|----------------------|---------------------------|--------------------------------------|--------------------------------------|-------------|
| `cindra_data_path`   | `CELL_FLUORESCENCE`       | `SINGLE_DAY_CELL_FLUORESCENCE`       | `single_day_cell_fluorescence`       | Yes         |
| `cindra_data_path`   | `NEUROPIL_FLUORESCENCE`   | `SINGLE_DAY_NEUROPIL_FLUORESCENCE`   | `single_day_neuropil_fluorescence`   | Yes         |
| `cindra_data_path`   | `SUBTRACTED_FLUORESCENCE` | `SINGLE_DAY_SUBTRACTED_FLUORESCENCE` | `single_day_subtracted_fluorescence` | Yes         |
| `cindra_data_path`   | `SPIKES`                  | `SINGLE_DAY_SPIKES`                  | `single_day_spikes`                  | Yes         |
| `multiday_data_path` | `CELL_FLUORESCENCE`       | `MULTI_DAY_CELL_FLUORESCENCE`        | `multi_day_cell_fluorescence`        | No          |
| `multiday_data_path` | `NEUROPIL_FLUORESCENCE`   | `MULTI_DAY_NEUROPIL_FLUORESCENCE`    | `multi_day_neuropil_fluorescence`    | No          |
| `multiday_data_path` | `SUBTRACTED_FLUORESCENCE` | `MULTI_DAY_SUBTRACTED_FLUORESCENCE`  | `multi_day_subtracted_fluorescence`  | No          |
| `multiday_data_path` | `SPIKES`                  | `MULTI_DAY_SPIKES`                   | `multi_day_spikes`                   | No          |

The single-day sources cover the cells detected in this session alone, which is why the session's own classification
mask applies to them. The multi-day sources cover the cells tracked across every session of this animal in the
dataset, and arrive already restricted to those cells. The subtracted pair is neuropil- and baseline-subtracted, the
spike pair is OASIS-deconvolved.

---

## Related skills

| Skill                             | Relationship                                                              |
|-----------------------------------|---------------------------------------------------------------------------|
| `forging:data-processing-design`  | Owns the agnostic processing-stage doctrine this sub-assembly concretizes |
| `forging:processing-input-format` | Documents the upstream inputs the forging pipeline consumes               |
| `forging:processing-results`      | Owns the agnostic forged-output layout on disk                            |
| `forging:dataset-forging`         | Orchestrates the forging batch that runs this sub-assembly                |
| `/mesoscope-vr-dataset-assembly`  | Calls this sub-assembly and consumes the reference clock it produces      |
| `/mesoscope-vr-module-parsing`    | Produces the upstream mesoscope-frame TTL feather                         |
| `/mesoscope-vr-processing-schema` | Owns the DatasetColumn roster and the processed-data filename roster      |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier
- [ ] rg -n 'ataraxis@|cindra@' <file> finds nothing

Alignment paths:
- [ ] Primary path keeps pulses whose duration is within expected_duration_ms plus or minus _SCAN_PULSE_TOLERANCE_MS
- [ ] expected_duration_ms comes from rate_to_interval over cindra's sampling_rate, not from a literal constant
- [ ] The stray pulse-run discard was applied before any comparison against the cindra frame count
- [ ] Surplus pulses front-clipped to the frame count, deficit routed to the ScanImage fallback
- [ ] The single-plane single-channel assumption was stated wherever the front clip or the entry-count check is cited

Stray pulse-run discard:
- [ ] _PULSE_RUN_GAP_FACTOR 3.0, _MINIMUM_SPLITTABLE_PULSE_COUNT 3, and _UNMATCHED_COST sys.maxsize quoted exactly
- [ ] Every early return that leaves the pulses unchanged was named, including the no-breaks single-run case
- [ ] Acquisition sizes resolved from frame_variant_metadata.npz by acquisitionNumbers or the frame-number multiset
- [ ] The discard was reported as a LogLevel.WARNING echo rather than a silent filter

ScanImage fallback:
- [ ] All three metadata keys named, with the zeros fallback when acquisitionNumbers is absent
- [ ] Chronological order is np.lexsort with acquisitionNumbers primary and frameNumberAcquisition secondary
- [ ] _SCANIMAGE_MATCH_TOLERANCE_US 50_000 and _SCANIMAGE_ANCHOR_SEARCH_LIMIT 10 quoted exactly
- [ ] The returned time_us was described as cast back to uint64 to match the primary path

Outputs and boundaries:
- [ ] Output index columns are frame (UInt32, 1-based), time_us (UInt64), and elapsed_minutes (Float32)
- [ ] Cell mask is classification[:, 0] == 1, applied to the single-day sources alone
- [ ] The eight sources cited by RecordingArrays and DatasetColumn member, never by cindra filename
- [ ] assemble_cindra_dataset described as a sub-assembler called by assemble_experiment_dataset
- [ ] Training sessions stated to carry no fluorescence sub-dataset
```
