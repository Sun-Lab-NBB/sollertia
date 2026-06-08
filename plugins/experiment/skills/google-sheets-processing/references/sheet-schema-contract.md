# Sheet schema contract

The exact structural assumptions and required headers that `SurgeryLog` and `WaterLog` validate at
construction, plus the field-to-header mapping each read produces. Loaded on demand from
`google-sheets-processing`'s SKILL.md. Source: `cross_system/google_sheet_tools.py`.

A processor builds a `header → column-letter` map from the live sheet and asserts every required
header is present before any extract/update call. Header matching is **case-insensitive and
whitespace-stripped** (headers are lowercased on read). A column whose required header is missing
aborts construction with a `ValueError` naming the missing headers.

---

## Cross-cutting conventions

| Convention            | Rule                                                                                              |
|-----------------------|---------------------------------------------------------------------------------------------------|
| Header matching       | Lowercased and stripped; the live header text must match the required header after that normalization. |
| Animal ID format      | Zero-padded to a five-digit string for comparison (`12` → `00012`).                               |
| Empty-value placeholders | `""`, `n/a`, `--`, `---` (case-insensitive) are read as `None`.                                |
| Column-letter mapping | 0-based column index → Excel-style letter (`A`, `B`, … `Z`, `AA`, …).                             |
| Supported date formats | `%m-%d-%y`, `%m-%d-%Y`, `%m/%d/%y`, `%m/%d/%Y`. Times are `%H:%M`.                               |
| Timezone              | Human-entered dates/times are interpreted as host **local time** and stored as UTC microseconds-since-epoch. |
| Write formatting      | Written cells get CENTER horizontal / MIDDLE vertical alignment via a follow-up `batchUpdate`.   |
| Retries               | Every API call uses `num_retries=5` (`_GOOGLE_API_MAX_RETRIES`) for transient 5xx/429 errors.    |

---

## Surgery log (`SurgeryLog`)

| Structural assumption | Value                                                                                  |
|-----------------------|----------------------------------------------------------------------------------------|
| Tab identity          | One tab per **project**; tab name = `project_name`.                                     |
| Header row            | Row **1**.                                                                              |
| Data rows             | Row 2 onward.                                                                           |
| Record identity       | The **`id` column** (zero-padded five-digit animal IDs). The target animal's row index is its position in that column + 2 (header row + 0→1 indexing). |

### Required headers

The `_REQUIRED_SURGERY_HEADERS` set, grouped by the dataclass section each feeds:

- **Subject:** `id`, `ear punch`, `sex`, `genotype`, `dob`, `weight (g)`, `cage #`, `location housed`, `status`
- **Procedure:** `date`, `start`, `end`, `surgeon`, `protocol`, `surgery notes`, `post-op notes`, `surgery quality`
- **Drug:** `lrs (ml)`, `ketoprofen (ml)`, `buprenorphine (ml)`, `dexamethasone (ml)`

### Dynamic implant / injection columns

Implants and injections are discovered by header name, not fixed columns. Headers follow the
`implant<N>` / `injection<N>` convention, with companion columns sharing the same prefix:

| Header pattern                  | Maps to                                       |
|---------------------------------|-----------------------------------------------|
| `implant<N>`                    | `ImplantData.implant` (the implant name)      |
| `implant<N> location`           | `ImplantData.implant_target`                  |
| `implant<N> coordinates`        | Parsed into AP/ML/DV (optional)               |
| `implant<N> code`               | `ImplantData.implant_code` (defaults to `0`)  |
| `injection<N>`                  | `InjectionData.injection`                     |
| `injection<N> location`         | `InjectionData.injection_target`              |
| `injection<N> volume (nl)`      | `InjectionData.injection_volume_nl`           |
| `injection<N> coordinates`      | Parsed into AP/ML/DV (optional)               |
| `injection<N> code`             | `InjectionData.injection_code` (defaults to `0`) |

The "main" column (no spaces, e.g. `implant1`) drives detection. A `None` value in that cell means
the animal does not have that implant/injection even though the header exists. Drug/implant/injection
`code` columns are optional and default to `0` (interpreted as "no code") for backward compatibility
with early sheet versions.

### Coordinate string format

Stereotactic coordinates are a single string like `-1.8 AP, 2 ML, .25 DV`, parsed into an
`(AP, ML, DV)` float tuple. A surgery without coordinates (e.g., training) defaults to `(0, 0, 0)`.

### Output: `SurgeryData`

`extract_animal_data()` returns `SurgeryData(subject, procedure, drugs, implants[], injections[])`.
Notable per-field parsing: `dob` is combined with a noon time; `date`+`start`/`end` become
`surgery_start_us`/`surgery_end_us`; `weight (g)` → `float`; `cage #` → `int`. A malformed or empty
weight, cage, date, or time cell raises `ValueError`.

---

## Water restriction log (`WaterLog`)

| Structural assumption | Value                                                                                  |
|-----------------------|----------------------------------------------------------------------------------------|
| Tab identity          | One tab per **animal**; tab name = the numeric animal ID (digit-only tabs only).        |
| Header row            | Row **2** (differs from the surgery log's row 1).                                        |
| Data rows             | Row 3 onward.                                                                           |
| Record identity       | The session's date must already exist in the pre-filled **`date` column**; the matching row is the session row. |

### Required headers

The `_REQUIRED_WATER_RESTRICTION_HEADERS` set: `date`, `weight (g)`, `given by:`, `water given (ml)`,
`behavior`, `time`.

### Written cells

`update_water_log(weight, water_ml, experimenter_id, session_type)` writes the session row:

| Column            | Value                                                              |
|-------------------|-------------------------------------------------------------------|
| `weight (g)`      | Animal weight at session onset (rounded to 1 decimal).            |
| `given by:`       | `experimenter_id`.                                                 |
| `water given (ml)`| Combined runtime + post-runtime water in mL (rounded to 1 decimal). |
| `behavior`        | `session_type`.                                                    |
| `time`            | Session start time in `HH:MM` local time (cached at construction). |

The log must be **pre-filled with session dates** — `WaterLog` writes into an existing date row and
raises `ValueError` if the session's date is absent (it does not create rows).

---

## What a custom processor must redefine

When adapting to a different schema or service, the parts above are exactly the parts that change.
A custom processor declares its own:

1. **Required-header (or required-field) set** — the equivalent of `_REQUIRED_*_HEADERS`, validated
   at construction.
2. **Structural assumptions** — which row holds headers, whether record identity is a column value or
   a tab name, where data rows begin.
3. **Field mapping** — how source columns map to the typed record it emits (read) or which columns it
   writes (write).

Everything in [Cross-cutting conventions](#cross-cutting-conventions) — auth, retries, connection
teardown, write formatting — is reusable as-is and should be preserved so the processor behaves like
the rest of the platform.
