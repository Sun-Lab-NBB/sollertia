---
name: mesoscope-vr-fluorescence-alignment
description: >-
  Documents Mesoscope-VR's concrete fluorescence-alignment stage: the primary TTL duration-window path
  (expected duration plus/minus 20 ms), the ScanImage metadata fallback with its match tolerance and anchor
  search limit, clock-offset estimation against frame_variant_metadata.npz, cell masking by classification,
  and the single-day versus multi-day fluorescence sources. Use when interpreting fluorescence frame
  alignment, debugging dropped or excess TTL pulses, or modifying ScanImage fallback tolerances.
user-invocable: false
---

# Mesoscope-VR fluorescence alignment

Documents how the Mesoscope-VR dataset-forging pipeline aligns two-photon fluorescence frames to the
microcontroller-logged frame-acquisition TTL pulses, building the frame-indexed reference time vector that the
session assembly stage interpolates every other stream onto.

This is the concrete Mesoscope-VR instance of the agnostic processing-stage seam owned by the forging plugin.
The alignment consumes upstream cindra single-recording and multi-recording fluorescence outputs and the
microcontroller mesoscope-frame TTL feather, and produces the frame-aligned table that becomes the spine of the
assembled session `data.feather`.

---

## Scope

**Covers:**
- The primary TTL alignment path: pairing rising and falling TTL edges and keeping pulses whose duration falls
  inside the expected scan-pulse window (`expected_duration_ms` plus/minus 20 ms)
- The clip-front rule when the log yields more in-window pulses than cindra's frame count
- The ScanImage metadata fallback path invoked when the duration filter yields fewer pulses than the frame count
- The match-tolerance and anchor-search-limit constants and their production rationale
- The ScanImage metadata keys (`frameNumberAcquisition`, `frameTimestamps_sec`) and the clock-offset estimation
  against `frame_variant_metadata.npz`
- The output columns (`frame`, `time_us`, `elapsed_minutes`) and cell masking via the classification first column
- The single-day versus multi-day fluorescence sources (cell, neuropil, neuropil-subtracted, spikes)
- The frame-aligned reference time vector (`time_us`) produced here and consumed as `reference_time` by
  `mesoscope:mesoscope-vr-dataset-assembly`

**Does not cover:**
- cindra single-recording and multi-day fluorescence production (upstream; see
  `forging:dataset-forging-input-format`)
- Assembly of the fluorescence frames into the session `data.feather` and the `Array(Float32)` column shapes (see
  `mesoscope:mesoscope-vr-dataset-assembly` and `forging:dataset-forging-results`)
- The dataset-forging batch orchestration (see `forging:dataset-forging` and `forging:data-processing-design`)
- Microcontroller mesoscope-frame TTL feather production (see `mesoscope:mesoscope-vr-module-parsing`)

---

## Fluorescence stage contract

The forging plugin owns the agnostic processing-stage doctrine: feather-as-interchange between stages, the
prepare-then-execute batch model, and the worker-budget concurrency contract (see
`forging:data-processing-design`). This skill documents only the Mesoscope-VR-specific content of the
fluorescence-alignment stage — the alignment algorithm and its constants — not the orchestration that schedules
it.

`assemble_cindra_dataset` in `mesoscope_vr/fluorescence.py` is the stage entry point. It takes four directory
paths and returns a single Polars DataFrame:

| Parameter             | Supplies                                                                            |
|-----------------------|-------------------------------------------------------------------------------------|
| `cindra_data_path`    | Single-recording cindra outputs: fluorescence traces, `cell_classification.npy`, metadata |
| `behavior_data_path`  | The microcontroller mesoscope-frame TTL feather (`mesoscope_frame_data.feather`)    |
| `multiday_data_path`  | The session's multi-recording cindra outputs (`cell_fluorescence.npy` and companions) |
| `raw_data_path`       | The session `raw_data` directory; resolves the ScanImage metadata archive for the fallback path |

The authoritative target frame count comes from the cindra `cell_fluorescence.npy` array. The function reads its
shape through a memory-mapped header read (`np.load(..., mmap_mode="r").shape`) and unpacks the second axis as
`frames`. Every alignment path must produce exactly that many rows.

---

## Primary TTL duration-window alignment

The mesoscope frame TTL feather (`BehaviorDataFiles.MESOSCOPE_FRAME` = `mesoscope_frame_data.feather`, produced
by the TTL module parser; see `mesoscope:mesoscope-vr-module-parsing`) holds `time_us` and `ttl_state` columns,
one row per logged TTL transition. The stage reads it memory-mapped, sorts by `time_us`, and reconstructs whole
pulses:

1. Compute `ttl_diff` as the first difference of `ttl_state` (null-filled with 0); a value of `1` marks a rising
   edge, `-1` a falling edge.
2. Assign a `pulse_id` as the cumulative sum of rising edges.
3. Filter rising edges into `(pulse_id, pulse_start)` and falling edges into `(pulse_id, pulse_end)`, then inner-
   join on `pulse_id`. The inner join drops any pulse missing either edge.
4. Compute `duration_ms` as `(pulse_end - pulse_start)` divided by 1000, and sort by `pulse_start`. This is the
   `paired_pulses` table.

The expected pulse duration is derived from cindra's per-plane sampling rate. The combined cindra metadata
archive (`combined_metadata.npz`) supplies `sampling_rate`; the first element becomes `scanning_frequency`, and:

```python
expected_duration_ms = 1000 / scanning_frequency        # _MILLISECONDS_PER_SECOND / scanning_frequency
min_duration = expected_duration_ms - 20                 # _SCAN_PULSE_TOLERANCE_MS
max_duration = expected_duration_ms + 20
```

The primary path keeps only pulses whose `duration_ms` is between `min_duration` and `max_duration` inclusive,
renames `pulse_id` to `frame` and `pulse_start` to `time_us`, and sorts by `frame`. The duration window is a
fixed `_SCAN_PULSE_TOLERANCE_MS = 20` ms on either side of the expected duration.

After the primary filter, the row count is reconciled against the cindra frame count:

| Condition                       | Action                                                                            |
|---------------------------------|-----------------------------------------------------------------------------------|
| In-window pulses `>` `frames`   | Clip the front: keep the last `frames` rows (`.tail(frames)`)                      |
| In-window pulses `==` `frames`  | Use the in-window pulses directly                                                  |
| In-window pulses `<` `frames`   | Fall back to ScanImage-metadata alignment (`_align_pulses_to_scanimage`)           |

Front-clipping is correct because excess in-window pulses arise when the operator manually triggers mesoscope
scanning before the main experiment runtime, so any aberrant frames precede the real session.

---

## ScanImage metadata fallback path

When the duration filter rejects real frames (the mesoscope can briefly hold a TTL outside the tolerance window
even though a scan was acquired), the duration-filtered count falls below cindra's frame count and the stage
delegates to `_align_pulses_to_scanimage`. ScanImage writes one entry per acquired TIFF to
`frame_variant_metadata.npz`, making its per-frame timestamps the authoritative record of which TTL rising edges
correspond to real frames. Pulses that match no ScanImage frame within tolerance are dropped as electrical noise
(typically clustered at session start during mesoscope arming and at session end after ScanImage stopped).

The fallback resolves the metadata archive at `raw_data_path / mesoscope_data / frame_variant_metadata.npz`
(`MesoscopeDirectories.MESOSCOPE_DATA` is `mesoscope_data`; the filename constant is
`_FRAME_VARIANT_METADATA_FILENAME`). It raises `ValueError` (via `console.error`) when the archive is missing,
when the archive entry count does not equal the cindra frame count, or when the final matching does not produce
exactly `expected_frame_count` rows.

Two ScanImage metadata keys are read from the archive:

| Constant                  | Key value               | Meaning                                                                  |
|---------------------------|-------------------------|--------------------------------------------------------------------------|
| `_SI_FRAME_NUMBER_KEY`    | `frameNumberAcquisition`| Strictly monotonic per-frame counter used to recover chronological order |
| `_SI_FRAME_TIMESTAMP_KEY` | `frameTimestamps_sec`   | Per-frame ScanImage clock timestamps in seconds                          |

The archive is stored in TIFF-page-concatenation order, which can interleave frames across stack files, so the
fallback sorts by `frameNumberAcquisition` (stable `argsort`) before converting `frameTimestamps_sec` to
microseconds (`int64`). NPZ archives do not support memory mapping, so a context manager keeps the archive open
only long enough to copy the two arrays out.

---

## Clock-offset estimation and anchor search

ScanImage and the microcontroller run on separate clocks. The fallback estimates a single constant offset by
anchoring a candidate TTL pulse to the first ScanImage frame, then choosing the offset that yields the most
matches:

1. For each candidate anchor (the first `_SI_ANCHOR_SEARCH_LIMIT = 10` pulses, or fewer if the log is shorter),
   compute `candidate_offset = pulse_microseconds[anchor] - si_microseconds[0]`.
2. Shift the ScanImage timestamps by the candidate offset and count how many TTL rising edges have a ScanImage
   frame strictly less than `_SI_MATCH_TOLERANCE_US` away.
3. Keep the offset with the highest match count.

The anchor search starts from the first few pulses because front-of-session noise is typically two short pulses;
ten anchors leave headroom for unusual setups. With the best offset applied, each pulse is matched to its nearest
aligned ScanImage frame via `searchsorted`-based nearest-target lookup. Conflicts where multiple pulses claim the
same frame are resolved by keeping the pulse closer to that frame's expected time; pulses not strictly closer than
`_SI_MATCH_TOLERANCE_US` to any frame are dropped.

| Constant                  | Value    | Rationale                                                                            |
|---------------------------|----------|-------------------------------------------------------------------------------------|
| `_SI_MATCH_TOLERANCE_US`  | `50_000` | Strict 50 ms bound absorbing pulse-edge jitter and clock drift (up to ~15 ms over an hour) |
| `_SI_ANCHOR_SEARCH_LIMIT` | `10`     | Maximum leading pulses tried as anchors; front noise is usually two short pulses     |

The fallback returns a DataFrame with `frame` (the matched pulse's original `pulse_id`) and `time_us` (the
rising-edge microsecond timestamp), sorted by `frame`, containing exactly `expected_frame_count` rows.

---

## Output columns and cell masking

Regardless of which path produced the frame-aligned table, the stage finalizes a frame index and a within-session
time column:

| Column            | Type      | Derivation                                                                            |
|-------------------|-----------|---------------------------------------------------------------------------------------|
| `frame`           | `UInt32`  | Re-assigned as a contiguous 1-based range over the surviving rows                      |
| `time_us`         | `UInt64`  | The TTL rising-edge timestamp, carried from the alignment path                         |
| `elapsed_minutes` | `Float32` | `(time_us - time_us.min())` divided by 60,000,000, rounded to 2 decimals               |

`time_us` is the frame-aligned reference time vector consumed by the assembly stage as `reference_time` (every
behavior and runtime stream is interpolated onto it); `elapsed_minutes` is a derived within-session display
column. See `mesoscope:mesoscope-vr-dataset-assembly`.

Cell masking uses the single-recording cell classification. cindra stores `cell_classification.npy` as a
`(num_rois, 2)` float32 array where column 0 holds the is-cell label (`1.0` or `0.0`) and column 1 the classifier
probability. The mask keeps ROIs where `classification[:, 0] == 1`. The mask is applied only to the single-day
fluorescence sources; the multi-day sources are added unmasked.

---

## Single-day versus multi-day sources

Eight fluorescence Series are streamed onto the frame-aligned table one file at a time (each `.npy` is memory-
mapped, masked if applicable, transposed to `(frames, rois)`, cast to float32, and appended), keeping peak memory
to one array-worth rather than eight:

| Source directory     | File                        | Column name                            | Cell-masked |
|----------------------|-----------------------------|----------------------------------------|-------------|
| `cindra_data_path`   | `cell_fluorescence.npy`     | `single_day_cell_fluorescence`         | Yes         |
| `cindra_data_path`   | `neuropil_fluorescence.npy` | `single_day_neuropil_fluorescence`     | Yes         |
| `cindra_data_path`   | `subtracted_fluorescence.npy`| `single_day_subtracted_fluorescence`  | Yes         |
| `cindra_data_path`   | `spikes.npy`                | `single_day_spikes`                    | Yes         |
| `multiday_data_path` | `cell_fluorescence.npy`     | `multi_day_cell_fluorescence`          | No          |
| `multiday_data_path` | `neuropil_fluorescence.npy` | `multi_day_neuropil_fluorescence`      | No          |
| `multiday_data_path` | `subtracted_fluorescence.npy`| `multi_day_subtracted_fluorescence`   | No          |
| `multiday_data_path` | `spikes.npy`                | `multi_day_spikes`                     | No          |

The two neuropil-subtracted, baseline-corrected columns are enumerated as `FluorescenceColumn`
(`single_day_subtracted_fluorescence` and `multi_day_subtracted_fluorescence`); they are the columns the
downstream place-cell, reward-cell, and SCE detectors treat as valid analysis inputs.

---

## Related skills

| Skill                                    | Relationship                                                                 |
|------------------------------------------|------------------------------------------------------------------------------|
| `forging:data-processing-design`         | Owns the agnostic processing-stage doctrine this stage concretizes           |
| `forging:dataset-forging-input-format`   | Documents the upstream cindra single-recording and multi-day fluorescence inputs |
| `forging:dataset-forging-results`        | Owns the forged fluorescence array shapes and output-schema reference        |
| `mesoscope:mesoscope-vr-dataset-assembly`| Consumes the frame-aligned reference vector produced here                     |
| `mesoscope:mesoscope-vr-module-parsing`  | Produces the upstream mesoscope-frame TTL feather                            |

---

## Verification checklist

```text
- [ ] Primary path keeps pulses with duration within expected_duration_ms ± 20 ms (_SCAN_PULSE_TOLERANCE_MS)
- [ ] expected_duration_ms derived as 1000 / scanning_frequency from cindra combined_metadata.npz sampling_rate
- [ ] Excess in-window pulses front-clipped to the cindra frame count; deficit triggers the ScanImage fallback
- [ ] Fallback resolves raw_data/mesoscope_data/frame_variant_metadata.npz and raises ValueError when absent
- [ ] Fallback reads frameNumberAcquisition and frameTimestamps_sec; sorts by the frame counter before matching
- [ ] _SI_MATCH_TOLERANCE_US = 50_000 us and _SI_ANCHOR_SEARCH_LIMIT = 10 quoted exactly
- [ ] Clock offset chosen as the anchor maximizing within-tolerance matches over the first 10 pulses
- [ ] Output columns are frame (UInt32, 1-based), time_us (UInt64), elapsed_minutes (Float32)
- [ ] Cell mask is classification[:, 0] == 1, applied to single-day sources only
- [ ] Eight fluorescence sources (single/multi day cell, neuropil, subtracted, spikes) appended correctly
- [ ] No reStructuredText specifiers; cross-references use the plugin:skill syntax (ataraxis@ where applicable)
```
