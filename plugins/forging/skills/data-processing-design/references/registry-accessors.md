# Registry accessors and donation Protocols

Holds the `resolve_*` accessors a pipeline calls to reach a per-system donation, and the donation Protocols those
donations satisfy.

---

## The accessors a pipeline calls

| Accessor                                          | Returns                                                             |
|---------------------------------------------------|---------------------------------------------------------------------|
| `resolve_microcontroller_parsers`                 | `dict[tuple[int, int], MicrocontrollerParser]`, re-keyed per system |
| `resolve_microcontroller_event_codes`             | `dict[tuple[int, int], tuple[int, ...]]`, by calling the donation   |
| `resolve_eligible_microcontroller_modules`        | `set[tuple[int, int]]`, taking the loaded session as a second input |
| `resolve_forging_assembly_worker`                 | `ForgingAssembler`                                                  |
| `resolve_forging_column_descriptions`             | `dict[str, str]`                                                    |
| `resolve_assembly_geometry_resolver`              | `_AssemblyGeometryResolver`                                         |
| `resolve_assembly_source_resolver`                | `_AssemblySourceResolver`                                           |
| `resolve_forging_admission_pipelines`             | `dict[SessionTypes, frozenset[ProcessingPipelines]]`                |
| `resolve_single_recording_configuration_resolver` | `Callable[[SessionData], SingleRecordingConfiguration]`             |
| `resolve_multi_recording_configuration_resolver`  | `Callable[[SessionData], MultiRecordingConfiguration \| None]`      |
| `resolve_multi_recording_session_types`           | `frozenset[SessionTypes]`                                           |
| `resolve_runtime_binding`                         | `tuple[str, _RuntimeParser]`, a source identifier and its parser    |
| `resolve_two_photon_data_locator`                 | `_TwoPhotonDataLocator`                                             |
| `resolve_pose_prediction_locator`                 | `_PosePredictionLocator`                                            |
| `resolve_video_tracking`                          | `_VideoTracker`                                                     |

Every accessor takes `system: str | AcquisitionSystems` as its first parameter and normalizes it through one gate, so a
caller holding the enum member and a caller holding its string value resolve the identical asset. An unknown system
raises a `ValueError` naming the supported members. The accessor is the API and the registry is an implementation
detail. That split buys three things a raw lookup does not, namely the string-or-member normalization, a named
`ValueError` in place of a bare `KeyError`, and the freedom to change a registry's internal shape without touching a
pipeline. A locator or the video-tracking function is reached in two steps, as in
`resolve_pose_prediction_locator(system=session.acquisition_system)(session=session)`.

---

## The donation Protocols

Every donated callable is a module-level, picklable function, because the parallel stages dispatch several of them into
spawned worker processes. `registries.py` exports two of the eight Protocols, and the remaining six appear only as the
return annotations of their accessors.

```python
class ForgingAssembler(Protocol):
    def __call__(self, source_session_path: Path, output_path: Path, dataset_name: str) -> None: ...


class MicrocontrollerParser(Protocol):
    def __call__(
        self, event_partition: dict[int, pl.DataFrame], output_directory: Path, session: SessionData
    ) -> None: ...
```

Coverage is a check on wiring rather than on capability. A system that produces none of a data class still donates an
entry. The entry takes the form of a no-op tracking function, a pose-prediction locator returning `None`, an empty
`frozenset[SessionTypes]`, and a two-photon locator returning the path the system would use.
