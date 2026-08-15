---
name: microcontroller-primitives
description: >-
  Documents the system-agnostic microcontroller module-feather parsing primitives in the cross_system layer that every
  per-system parser builds on: non-recursive discovery, filename parsing, single-pass event-code partitioning of the
  axci five-column feather schema, typed timestamp/value extraction, and merging sorted event streams. Use when
  auditing or writing a per-system module parser, explaining the axci feather schema, or adding an agnostic helper.
user-invocable: false
---

# Microcontroller primitives

Documents the system-agnostic microcontroller module-feather parsing primitives in the cross_system layer that every
per-acquisition-system module parser builds on.

These primitives are a pure, stateless helper API. They carry no orchestration, no processing tracker, and no
acquisition-system selector. They live in `sollertia_forgery.cross_system.microcontroller` and operate on the
`.feather` files produced by the `ataraxis-communication-interface` (axci) log-processing pipeline. A per-system
parser registry (for example the Mesoscope-VR registry) composes these helpers; the registry, not these primitives,
owns the mapping from a hardware module to its calibrated output, its event-code meanings, and its unit conversions.

---

## Scope

**Covers:**
- `find_module_feathers` non-recursive discovery of `controller_*_module_*.feather` files under a data directory
- `parse_module_feather_name` extraction of the `(controller_id, module_type, module_id)` integer tuple, including
  its `ValueError`-on-malformed-name contract
- The axci five-column feather schema (`timestamp_us`, `command`, `event`, `dtype`, `data`) as the shared input
  contract these primitives consume
- `partition_events` single-pass partition of a module DataFrame into a dict keyed by event code
- `get_event_timestamps` and `get_event_data` typed extraction (uint64 timestamps plus vectorized binary-payload
  decode via `np.frombuffer`)
- `merge_event_streams` stable-mergesort merge of two chronologically sorted timestamp/value streams
- How a per-system parser registry keys output filenames and conversions off `(module_type, module_id)` without any
  acquisition-system selector

**Does not cover:**
- Concrete Mesoscope-VR module parsers, event-code meanings, output filenames, and unit conversions (see
  `mesoscope:mesoscope-vr-module-parsing`)
- Upstream axci production of the module feather files (see `ataraxis@communication:log-processing` and
  `ataraxis@communication:log-input-format`)
- Hardware-state gating of module eligibility (see `assets:session-hardware-state` and `forging:behavior-input-format`)
- The behavior-processing batch orchestration that calls these primitives (see `forging:behavior-processing`)
- Camera timestamp feathers and their renaming stage (see `forging:camera-timestamp-extraction`)
- The platform-general processing doctrine (prepare-then-execute, worker budgets) (see `forging:data-processing-design`)

---

## The axci five-column feather schema

The shared input contract for every primitive here is the uncompressed Arrow IPC (`.feather`) file that axci writes
once per `(controller_id, module_type, module_id)` tuple. Each file carries a fixed five-column schema:

| Column         | Polars dtype | Description                                                     |
|----------------|--------------|-----------------------------------------------------------------|
| `timestamp_us` | `UInt64`     | Message timestamp in microseconds since the UTC epoch           |
| `command`      | `UInt8`      | Command code the module was executing when the message was sent |
| `event`        | `UInt8`      | Event code identifying the message type                         |
| `dtype`        | `String`     | NumPy dtype string for the data payload (null if no payload)    |
| `data`         | `Binary`     | Serialized binary payload (null if no payload)                  |

The primitives in this skill never read the file themselves — they accept an already-loaded `pl.DataFrame`. A
per-system parser is responsible for the read; it uses memory-mapped Polars IPC (`pl.read_ipc(..., memory_map=True)`),
which is safe precisely because axci writes the feather uncompressed. Defer the production of these files and the
upstream message protocol to `ataraxis@communication:log-processing` and `ataraxis@communication:log-input-format`.

---

## Discovering and naming module feathers

### Non-recursive discovery

`find_module_feathers(data_directory)` searches `data_directory` **non-recursively** with
`data_directory.glob("controller_*_module_*.feather")` — it does NOT walk subdirectories. The directory is expected
to be the session's canonical `processed_data/microcontroller_data` location exposed by
`SessionData.microcontroller_data_path`. The function returns a sorted list of matching paths, and returns an **empty
list** if the directory does not exist (the body short-circuits when `data_directory.is_dir()` is false) or if no
files match. Callers therefore never need to guard for a missing directory before calling it.

### The filename naming convention and ValueError contract

`parse_module_feather_name(feather_path)` extracts the `(controller_id, module_type, module_id)` integer tuple from a
filename following the convention:

```text
controller_{controller_id}_module_{module_type}_{module_id}.feather
```

It splits the file stem on `_` and validates three conditions: the split must yield **exactly five parts**, part 0
must be the literal `controller`, and part 2 must be the literal `module`. If any check fails, it raises `ValueError`
(via `console.error(message=..., error=ValueError)`) with a message naming the offending file. On success it returns
`int(parts[1]), int(parts[3]), int(parts[4])` — that is, `(controller_id, module_type, module_id)`.

```python
controller_id, module_type, module_id = parse_module_feather_name(feather_path)
```

---

## Partitioning events by code

`partition_events(module_dataframe)` partitions the five-column DataFrame into per-event sub-DataFrames in a **single
pass** and returns a `dict[int, pl.DataFrame]` keyed by integer event code. It is the preferred replacement for
calling `module_dataframe.filter(pl.col("event") == code)` once per code, which rescans the whole DataFrame on each
call; the single partition pass makes subsequent per-code lookups O(1).

Internally it calls `module_dataframe.partition_by("event", as_dict=True)`. Polars returns single-element tuples as
the dict keys even when partitioning on one column, so the event code is unwrapped from index 0 and cast to `int`
before becoming the returned dict key. Always feed the dict this function returns into the extraction helpers below —
they expect the event-code-keyed partition, not the raw DataFrame.

---

## Extracting typed timestamp and value arrays

Two helpers read a single event code out of the partition dict. Both return empty arrays (not `None`) when the event
code is absent from the partition, so a parser can call them unconditionally for an optional event.

### State-only events

`get_event_timestamps(partition, event_code)` returns an `NDArray[np.uint64]` of timestamps for state-only events that
carry no data payload. A missing event code yields an empty `uint64` array.

### Events with payloads

`get_event_data(partition, event_code, values_dtype)` returns a tuple `(timestamps, values)` where `timestamps` is
`NDArray[np.uint64]` and `values` is cast to the caller-supplied `values_dtype`. It relies on the axci protocol
guarantee that all messages sharing an event code also share a payload dtype: the binary `data` payloads are
concatenated and decoded with a **single** `np.frombuffer(b"".join(data_list), dtype=payload_dtype)` call — taking the
dtype from the first row — rather than a per-row Python loop, then cast to `values_dtype` for uniform downstream
handling. A missing event code yields a pair of empty arrays.

```python
timestamps, values = get_event_data(partition, event_code=51, values_dtype=np.float64)
```

---

## Merging sorted event streams

`merge_event_streams(timestamps_a, values_a, timestamps_b, values_b)` merges two already-chronologically-sorted event
streams into one timestamp-sorted stream and returns `(merged_timestamps, reordered_values)`. It concatenates the two
timestamp arrays and the two value arrays, computes a `np.argsort(..., kind="stable")` ordering over the concatenated
timestamps, and applies that ordering to both arrays. NumPy's stable mergesort is near-linear on the already-sorted
runs the axci log format produces, and the stable sort keeps equal-timestamp records in their original relative order.
This consolidates the allocate-empty / fill-halves / argsort pattern that each parser would otherwise duplicate.

---

## How per-system parsers consume these primitives

A per-acquisition-system parser is a thin composition over these helpers. The canonical flow is:

1. `find_module_feathers(microcontroller_data_path)` to discover candidate files.
2. `parse_module_feather_name(path)` to recover `(controller_id, module_type, module_id)`.
3. Dispatch on `(module_type, module_id)` through the per-system registry to pick the parser for that module.
4. Inside the parser: read the feather with memory-mapped Polars IPC, call `partition_events` once, then pull each
   needed event with `get_event_timestamps` or `get_event_data`, and combine streams with `merge_event_streams`.

The registry keys off `(module_type, module_id)` only — never off the acquisition system. System specificity enters
purely as data (the registry contents and the calibration fields the parser reads), which is why these primitives stay
agnostic. The concrete Mesoscope-VR registry, its event codes, its required hardware-state calibration fields, its
output filenames, and its unit conversions are owned by `mesoscope:mesoscope-vr-module-parsing`; do not duplicate any
of that here.

---

## Related skills

| Skill                                   | Relationship                                                                              |
|-----------------------------------------|-------------------------------------------------------------------------------------------|
| `forging:data-processing-design`        | Owns the platform-general processing doctrine these primitives slot into                  |
| `forging:camera-timestamp-extraction`   | Sibling agnostic stage; handles camera feathers rather than microcontroller module feathers |
| `forging:behavior-input-format`         | Reference for session eligibility and hardware-state gating of module jobs                 |
| `forging:behavior-processing`           | Batch orchestrator that invokes the per-system parsers built on these primitives          |
| `mesoscope:mesoscope-vr-module-parsing` | Concretizes these primitives with the Mesoscope-VR module registry and conversions        |
| `ataraxis@communication:log-processing` | Upstream producer of the `controller_*_module_*.feather` files these primitives consume    |

---

## Verification checklist

You MUST verify your work against this checklist before submitting.

```text
- [ ] Function names, signatures, and return types match cross_system/microcontroller.py exactly
- [ ] Discovery is described as NON-recursive (data_directory.glob, not rglob) returning an empty list when missing
- [ ] parse_module_feather_name's ValueError contract (five parts, 'controller' / 'module' literals) is stated
- [ ] The axci five-column schema (timestamp_us, command, event, dtype, data) is presented as the input contract
- [ ] partition_events is described as a single-pass partition_by keyed by integer event code
- [ ] get_event_timestamps / get_event_data empty-array-on-missing behavior is documented, not invented
- [ ] get_event_data's single np.frombuffer decode and shared-dtype guarantee are described accurately
- [ ] merge_event_streams is described as stable mergesort over concatenated streams
- [ ] No concrete module conversions, event-code meanings, or output filenames are documented (deferred out)
- [ ] "feather" appears only as the Arrow IPC file-format term, never as a module or skill name
- [ ] Cross-references use plugin:skill syntax with the ataraxis@ prefix for ataraxis-marketplace skills
- [ ] No reStructuredText specifiers (:class:/:func:/:meth:) anywhere in the file
```
