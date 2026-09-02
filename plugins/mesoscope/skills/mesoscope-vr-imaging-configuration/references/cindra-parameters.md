# Mesoscope-VR cindra parameter values

Catalogues every cindra parameter the two Mesoscope-VR configuration builders in `mesoscope_vr/two_photon.py` write
explicitly. Both builders name every field of every section they construct, so the resulting configuration is
decoupled from cindra's evolving defaults. A value marked indicator-dependent comes from the `_IndicatorParameters`
bundle the animal's calcium indicator earns, and every other value is a literal identical for both indicators.

The parameter semantics belong to cindra. `cindra:single-recording-configuration` and
`cindra:multi-recording-configuration` (cindra marketplace) document what each field does. This file records only the
values Mesoscope-VR chooses.

---

## Indicator-dependent values

| Indicator  | `tau` | `neuropil_coefficient` | `probability_threshold` |
|------------|-------|------------------------|-------------------------|
| `GCAMP6F`  | `0.4` | `0.7`                  | `0.85`                  |
| `JGCAMP8S` | `0.7` | `0.8`                  | `0.80`                  |

---

## Single-recording configuration

`_build_single_recording_configuration(genotype)` returns a `SingleRecordingConfiguration` with these eight sections.

| Section                 | Field                                      | Value                    |
|-------------------------|--------------------------------------------|--------------------------|
| `Main`                  | `two_channels`                             | `False`                  |
| `Main`                  | `first_channel_functional`                 | `True`                   |
| `Main`                  | `second_channel_functional`                | `False`                  |
| `Main`                  | `tau`                                      | indicator-dependent      |
| `Main`                  | `ignored_flyback_planes`                   | `()`                     |
| `Main`                  | `custom_classifier_path`                   | `None`                   |
| `FileIO`                | `ignored_file_names`                       | `("zstack",)`            |
| `FileIO`                | `repeat_binarization`                      | `False`                  |
| `Registration`          | `repeat_registration`                      | `False`                  |
| `Registration`          | `align_by_first_channel`                   | `True`                   |
| `Registration`          | `reference_frame_count`                    | `500`                    |
| `Registration`          | `batch_size`                               | `100`                    |
| `Registration`          | `maximum_offset_fraction`                  | `0.1`                    |
| `Registration`          | `spatial_smoothing_sigma`                  | `1.15`                   |
| `Registration`          | `temporal_smoothing_sigma`                 | `0.0`                    |
| `Registration`          | `two_step_registration`                    | `False`                  |
| `Registration`          | `gpu_batch_size`                           | `0`                      |
| `Registration`          | `bad_frame_threshold`                      | `1.0`                    |
| `Registration`          | `normalize_frames`                         | `True`                   |
| `Registration`          | `registration_metric_principal_components` | `10`                     |
| `Registration`          | `compute_bidirectional_phase_offset`       | `False`                  |
| `Registration`          | `bidirectional_phase_offset_override`      | `0`                      |
| `OnePhotonRegistration` | `enabled`                                  | `False`                  |
| `OnePhotonRegistration` | `spatial_highpass_window`                  | `42`                     |
| `OnePhotonRegistration` | `pre_smoothing_sigma`                      | `0.0`                    |
| `OnePhotonRegistration` | `edge_taper_pixels`                        | `40.0`                   |
| `NonrigidRegistration`  | `enabled`                                  | `True`                   |
| `NonrigidRegistration`  | `block_size`                               | `(128, 128)`             |
| `NonrigidRegistration`  | `signal_to_noise_threshold`                | `1.2`                    |
| `NonrigidRegistration`  | `maximum_block_offset`                     | `5.0`                    |
| `ROIDetection`          | `enabled`                                  | `True`                   |
| `ROIDetection`          | `preclassification_threshold`              | `0.0`                    |
| `ROIDetection`          | `threshold_scaling`                        | `1.0`                    |
| `ROIDetection`          | `spatial_highpass_window`                  | `25`                     |
| `ROIDetection`          | `maximum_overlap`                          | `0.75`                   |
| `ROIDetection`          | `temporal_highpass_window`                 | `100`                    |
| `ROIDetection`          | `maximum_iterations`                       | `50`                     |
| `ROIDetection`          | `maximum_binned_frames`                    | `5000`                   |
| `ROIDetection`          | `denoise`                                  | `False`                  |
| `ROIDetection`          | `crop_to_soma`                             | `True`                   |
| `SignalExtraction`      | `extract_neuropil`                         | `True`                   |
| `SignalExtraction`      | `allow_overlap`                            | `False`                  |
| `SignalExtraction`      | `minimum_neuropil_pixels`                  | `350`                    |
| `SignalExtraction`      | `inner_neuropil_border_radius`             | `2`                      |
| `SignalExtraction`      | `cell_probability_percentile`              | `50`                     |
| `SignalExtraction`      | `classification_threshold`                 | `0.5`                    |
| `SignalExtraction`      | `batch_size`                               | `500`                    |
| `SignalExtraction`      | `colocalization_threshold`                 | `0.65`                   |
| `SpikeDeconvolution`    | `extract_spikes`                           | `True`                   |
| `SpikeDeconvolution`    | `neuropil_coefficient`                     | indicator-dependent      |
| `SpikeDeconvolution`    | `baseline_method`                          | `BaselineMethod.MAXIMIN` |
| `SpikeDeconvolution`    | `baseline_window`                          | `60.0`                   |
| `SpikeDeconvolution`    | `baseline_sigma`                           | `10.0`                   |
| `SpikeDeconvolution`    | `baseline_percentile`                      | `8.0`                    |

Left at cindra defaults: `file_io.data_path`, `file_io.output_path`, and the `runtime` settings.

---

## Multi-recording configuration

`_build_multi_recording_configuration(genotype)` returns a `MultiRecordingConfiguration` with these six sections.

| Section                     | Field                             | Value                              |
|-----------------------------|-----------------------------------|------------------------------------|
| `RecordingIO`               | `repeat_selection`                | `False`                            |
| `ROISelection`              | `probability_threshold`           | indicator-dependent                |
| `ROISelection`              | `maximum_size`                    | `1000`                             |
| `ROISelection`              | `mroi_region_margin`              | `30`                               |
| `ROISelection`              | `probability_threshold_channel_2` | `None`                             |
| `ROISelection`              | `maximum_size_channel_2`          | `None`                             |
| `ROISelection`              | `mroi_region_margin_channel_2`    | `None`                             |
| `DiffeomorphicRegistration` | `image_type`                      | `ReferenceImageType.ENHANCED_MEAN` |
| `DiffeomorphicRegistration` | `grid_sampling_factor`            | `1`                                |
| `DiffeomorphicRegistration` | `final_grid_sampling`             | `16.0`                             |
| `DiffeomorphicRegistration` | `scale_sampling`                  | `30`                               |
| `DiffeomorphicRegistration` | `speed_factor`                    | `3`                                |
| `DiffeomorphicRegistration` | `repeat_registration`             | `False`                            |
| `ROITracking`               | `threshold`                       | `0.75`                             |
| `ROITracking`               | `mask_prevalence`                 | `50`                               |
| `ROITracking`               | `pixel_prevalence`                | `50`                               |
| `ROITracking`               | `step_sizes`                      | `(200, 200)`                       |
| `ROITracking`               | `bin_size`                        | `50`                               |
| `ROITracking`               | `maximum_distance`                | `20`                               |
| `ROITracking`               | `minimum_size`                    | `25`                               |
| `SignalExtraction`          | `extract_neuropil`                | `True`                             |
| `SignalExtraction`          | `allow_overlap`                   | `True`                             |
| `SignalExtraction`          | `minimum_neuropil_pixels`         | `350`                              |
| `SignalExtraction`          | `inner_neuropil_border_radius`    | `2`                                |
| `SignalExtraction`          | `cell_probability_percentile`     | `50`                               |
| `SignalExtraction`          | `classification_threshold`        | `0.5`                              |
| `SignalExtraction`          | `batch_size`                      | `500`                              |
| `SignalExtraction`          | `colocalization_threshold`        | `0.65`                             |
| `SpikeDeconvolution`        | `extract_spikes`                  | `True`                             |
| `SpikeDeconvolution`        | `neuropil_coefficient`            | indicator-dependent                |
| `SpikeDeconvolution`        | `baseline_method`                 | `BaselineMethod.MAXIMIN`           |
| `SpikeDeconvolution`        | `baseline_window`                 | `60.0`                             |
| `SpikeDeconvolution`        | `baseline_sigma`                  | `10.0`                             |
| `SpikeDeconvolution`        | `baseline_percentile`             | `8.0`                              |

Left at cindra defaults: `recording_io.recording_directories`, `recording_io.dataset_name`, and the `runtime`
settings.

---

## Where the two configurations differ

`SignalExtraction` and `SpikeDeconvolution` are written by both builders. Every field of both sections carries the
same value in each, except `allow_overlap`, which is `False` for single-recording extraction and `True` for
multi-recording extraction. Overlapping ROI pixels are excluded within one recording and retained across recordings,
because a tracked cell registered against other recordings keeps the pixels it shares with a neighbor.
